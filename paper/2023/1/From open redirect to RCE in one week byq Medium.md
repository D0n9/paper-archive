# From open redirect to RCE in one week | byq | Medium
In this article, I will tell you a story of how I chained multiple security issues to achieve RCE on several hosts of the _Mail.Ru Group_ (or _VK_ now). For me, as a regular reader of bug bounty write-ups, it is always interesting to learn the train of hunter’s thoughts during uncommon vulnerability exploitation, so I tried to describe my process as detailed as possible. I hope, you will enjoy it.

I am a huge fan of the [_Mail.Ru Group_ bug bounty program on HackerOne](https://hackerone.com/mailru). _VK,_ which is a new name of _Mail.Ru group,_ sometimes buys new companies, and the _VK_ BBP is replenished with new assets, which gives a nice chance for the hackers to loot low-hanging fruits.

Based on my experience I can say that it can be very profitable to access something, which no one before you tried to hack. I subscribed to _VK_ BBP to receive updates about new assets to be among the first who will get access to them.

During 2021 I was not very active at BB and didn’t closely follow updates in my favorite BBP and that is the reason why I missed the notification that [**_Seedr_**](https://seedr.ru/)_,_ which is a platform for video advertising, however now deprecated, was added to the assets.

My first meeting with _Seedr_ happened in October 2021. Almost after a few minutes, I found several simple XSS, which I decided not to report because the chance of Duplicate was too high.

Presently you can observe several disclosed reports by other hunters and check how atypical bugs for modern applications were found there:

*   [RCE в .api/nr/report/{id}/download](https://hackerone.com/reports/1348154)
*   [SSRF + RCE через fastCGI в POST /api/nr/video](https://hackerone.com/reports/1354335)
*   [XSS Stored on https://seedr.ru](https://hackerone.com/reports/1350671)
*   [OS command injection on seedr.ru](https://hackerone.com/reports/1360208)

I thought that my train was gone, so I didn’t spend a lot of time on _Seedr_ and continued my BB procrastination.

I returned to _Seedr_ during my December vacation in another country, where I was just with a backpack and laptop. After some time in such conditions BB hunger wakes up and there is a desire to find something interesting. To boost my confidence I usually take a fresh look at already familiar assets.

This time I spent more time on the recon: subdomain enumeration, port scanning, directory brute forcing, and so on. Luckily I found juicier things: GitLab, Grafana, several API hosts, cron files in the web directory, stack traces, and more. The more entry points you find — the higher chance to find something spicy. Unfortunately for me, none were worth reporting to BBP, but one functionality caught my attention.

In the HTML source code of the _https://api-stage.seedr.ru/player_ page, I noticed the following comment:

![](https://miro.medium.com/max/1400/0*FsNfTwwYa-jdhbRZ)

```
https://player.seedr.ru/video?vid=cpapXGq50UY&post_id=57b6ceef64225d5b0f8b456c&config=https%3A%2F%2Fseedr.com%2Fconfig%2F57975d1b64225d607e8b456e.json&hosting=youtube
```

I bet that you already want to modify the `**config**`  GET parameter with your host to receive incoming HTTP connection, which I did. After several attempts, I didn’t receive any connections and continued to play with other parameters.

When I opened _https://player.seedr.ru/video?vid=cpapXGq50UY&post_id=57b6ceef64225d5b0f8b456c&config=https%3A%2F%2Fseedr.com%2Fconfig%2F57975d1b64225d607e8b456e.json&hosting=youtube_ in browser and observed response I noticed that Open Graph meta tags filled with some information about video like title, description, image, etc.:

![](https://miro.medium.com/max/1400/0*MNbpKdZdssJy3CGt)

After some tests, I understood that `**post_id**`  and `**config**`  GET parameters have no significant effect on the response, so let’s simplify the URL to _https://player.seedr.ru/video?vid=cpapXGq50UY&hosting=youtube_.

I assumed that it’s unlikely that the player only supports YouTube and changed `**hosting**`  GET parameter to _coub_ and _vimeo:_

![](https://miro.medium.com/max/1400/0*vSGyBnTXzA7JuWoD)

![](https://miro.medium.com/max/1400/0*4RRsPODrFdmKdCdI)

So, it seems like depending on the value of `hosting` GET parameter server performs an HTTP request with `**file_get_contents()**` to YouTube, Vimeo, or Coub API, downloads metadata about the video (`**vid**`  GET parameter), parses it, and returns the player HTML page with this video and filled Open Graph meta tags.

`vid` GET parameter is an injection point, it’s possible to control the last part of the path in `file_get_contents()`. I could use path traversal (`/../`) and useful symbols (`?`, `#`, `@`) as well.

What is more interesting, in the case of Vimeo server makes a request to _http://vimeo.com/api/v2/video/VID.php_, you could notice it on the previous screenshot and it turned out that, when you use `**.php**` extension in the path, Vimeo returns not a JSON string, but serialized string!

![](https://miro.medium.com/max/1206/0*WFyuCQqoJQjXxzrX)

I assumed that after `file_get_contents()` the server uses `**unserialize()**` on the response from Vimeo:

![](https://miro.medium.com/max/1400/0*SDSp66MW4M4x56Do)

Wow, do I have unsafe deserialization here? It’s safe, as long as the response is under Vimeo’s control.

At that point I already saw three possible scenarios:

1.  Fuzzing of `file_get_contents()` to escape _vimeo.com_ host, achieve blind SSRF and probably unsafe deserialization;
2.  Find controlled response on _vimeo.com_ -\> probably unsafe deserialization;
3.  Find open redirect on _vimeo.com_ -\> blind SSRF -> probably unsafe deserialization.

After hours of different modifications of the `vid`  GET parameter and the fuzzing of `file_get_contents()` locally, I didn’t find anything useful and decided to share all information about this finding with several trusted mates.

Ok, the first scenario didn’t work, let’s move to the next one with a controlled response on _vimeo.com_.

Endpoint with the controlled response should meet the following requirements:

*   200 OK HTTP status code;
*   Available for an unauthenticated user;
*   Controlled string should be at the beginning of the response body (PHP will successfully parse something like this `{VALID_SER_STRING}TRASH`);
*   Controlled string should support `{ }`,  `"`  symbols required for storing serialized objects.

Here are some of my attempts to find desired behavior on _vimeo.com:_

1.  **injection** is not a valid method.  
    _Cons:_ 404 Not Found HTTP status code, doesn’t support `{}`,  `"` symbols.

![](https://miro.medium.com/max/1400/0*vZ1Nj3QcypkJN37H)

2. **injection** is not a valid format.  
_Cons:_ 404 Not Found HTTP status code, doesn’t support `{}`,  `"` symbols.

![](https://miro.medium.com/max/1400/0*BbYZmj1edac0KZW6)

3\. JS callback.  
_Cons:_ `/**/` at the beginning, doesn’t support `{}`,  `"` symbols.

![](https://miro.medium.com/max/1400/0*0VLcvvgxPQKxWBtM)

4\. Export of chat of live broadcast:  
_Cons:_ Date and name at the beginning, require authentication.

![](https://miro.medium.com/max/1400/0*vq7xF4uEPkHLj3SF)

Unfortunately, the second scenario also didn’t work, so my last hope was to find an open redirect on _vimeo.com_. Previously I already saw a disclosed report on HackerOne from 2015 with an open redirect on _vimeo.com_: [https://hackerone.com/reports/44157](https://hackerone.com/reports/44157), so I assumed that there was some chance to find one more. Actually, I already looked for an open redirect during the discovery of controlled response but didn’t find anything useful.

All that time when I tried to exploit this vulnerability I kept in mind the [article from Harsh Jaiswal](https://infosecwriteups.com/vimeo-ssrf-with-code-execution-potential-68c774ba7c1e) ([@rootxharsh](https://twitter.com/rootxharsh)) about SSRF on _vimeo.com._ I distinctly remember that he chained several open redirects on _vimeo.com_ to achieve the goal. This finding was in 2019, almost 3 years ago, so of course, at first, I thought that these open redirects were fixed. But it was probably my last chance, so I started digging in that way.

Thanks to low censoring of the screenshots it was possible to fingerprint the endpoint by GET parameters. Combining this, some googling, and reading Vimeo API docs I was able to guess what endpoint was used by Harsh in his chain. Anyway, it was still unclear what values I should provide.

![](https://miro.medium.com/max/1400/0*o9o2nLq5zstii725)

Screenshot from Harsh’s article

I very rarely ask someone for help while exploiting something, not counting several mates, but because I was at an impasse, [Harsh](https://twitter.com/rootxharsh) was my last key.

After I contacted him and provided him with information that I had on that step, he shared with me a working open redirect link which was the same as I assumed but with proper values of GET parameters. From that link, I understood that it’s not a security issue on _vimeo.com,_ but just a feature (really, it’s not a joke).

![](https://miro.medium.com/max/1400/0*AcSxUlWJYYC9y5GC)

Ok, now I have a working open redirect on _vimeo.com,_ let’s try it at work:

![](https://miro.medium.com/max/1400/0*SRMWKxA-A4XY47ba)

![](https://miro.medium.com/max/1400/1*i--oUWFn1c4xHfs2Q4oMnQ.png)

Yes, I finally got an HTTP hit on my host. Before jumping to deserialization I decided to play a little with SSRF:

*   _https://127.0.0.1_

![](https://miro.medium.com/max/1400/0*00A5V6ZE5uZ7akUH)

*   _https://127.0.0.1:22_

![](https://miro.medium.com/max/1400/0*sI9XaaY_h_y_k0CF)

*   _http://127.0.0.1:25_

![](https://miro.medium.com/max/1400/0*Qeb2LIqyKcpgDyiM)

Because the response from `file_get_contents()` goes directly to `unserialize()` I couldn’t achieve full SSRF, but at least I already had semi-blind SSRF with ability to perform the port scan:

![](https://miro.medium.com/max/1064/0*TTbbOge03Ys5ogqJ)

After understanding that I used almost the whole potential of this SSRF I switched to the exploitation of `unserialize()`.

Briefly, what is needed for successful exploitation of unsafe deserialization in PHP?

*   ̶C̶o̶n̶t̶r̶o̶l̶l̶e̶d̶ ̶i̶n̶p̶u̶t̶;̶
*   Class with magical method (`**__wakeup()**`, `**__destroy()**`, `**__toString()**`, etc.);
*   Useful for attacker functionality in magical method which can be abused for file manipulation, RCE, SQLi, etc.;
*   Class is loaded.

As you can see, at that point I had only 1 of 4 requirements. I didn’t know anything about backend code on the host, so the only way to exploit deserialization is to blindly try all known gadget chains. For that purpose, I used the awesome tool [PHPGGC](https://github.com/ambionics/phpggc) which is a library of PHP `unserialize()` payloads along with a tool to generate them. At the moment of exploitation, it has almost 90 available payloads. A big part of them is for different CMS and frameworks like WordPress, ThinkPHP, Typo3, Magento, Laravr, etc., which would be useless in my case. So I betted on commonly used libraries like Doctrine, Guzzle, Monolog and Swift Mailer.

I pre-generated all available payloads with PHPGGC, hosted them on a controlled server, and started brute-forcing. And… in all of the cases, I got the same error:

![](https://miro.medium.com/max/1400/0*qoN3AU3DWc_hnZt7)

> [The error occurs because inside serialized string there is a reference to a class that hasn’t been included yet — so the PHP autoloading mechanism is triggered to load that class, and this fails for some reason.](https://stackoverflow.com/a/19511801) © [Sven](https://stackoverflow.com/users/1627406/sven)

At that point I have come to terms with the fact that this PHP script is very primitive and doesn’t include any additional classes, which I can use. Sad, but at least I tried. It often happens when you chain cool vulnerability, but face something that completely blocks you.

After summarizing all the findings I went to HackerOne and submitted a report with the name [_\[player.seedr.ru\] Semi-blind SSRF_](https://hackerone.com/reports/1430989) and for sure invited [Harsh Jaiswal](https://hackerone.com/bugdiscloseguys) as a collaborator for his open redirect on _vimeo.com_.

Basically, this is where the story could have ended. But there was a feeling inside that kept me up at night telling me that it was not over yet and I should try something else. I guess you know how it feels.

I don’t remember where exactly but a few days later by chance, my eye caught on something about use-after-free vulnerability in PHP `unserialize()`. As it happened the version of PHP on _player.seedr.ru_ was outdated and of course, I started “researching” this area. During that “research” I got acquainted with the reports of [Taoguang Chen](https://twitter.com/chtg57) who reported to PHP may be dozens of issues with `unserialize()`. Actually, vulnerabilities related to memory are still a dark area for me, but honestly, I tried to build some payloads. After some poking on the local environment, I returned to _player.seedr.ru_, hosted the payload on a controlled server, sent a request, and …

![](https://miro.medium.com/max/1400/0*IID50Ey7OlrMV5R1)

“What? No space left on the device? Really, I just started? But, wait, this doesn’t look like a default error about space on the device.”

By the way, this error occurred probably because I sent too many requests with scanners for hidden directories and files on previous days.

```
ErrorException \[ 2 \]: file\_put\_contents(/var/www/seedr.backend.v2/application/logs/2021/12/20.php): failed to open stream: No space left on device ~ SYSPATH/classes/kohana/log/file.php \[ 81 \]
```

“Custom class for logging? Seems like this “primitive” PHP script nevertheless loads logging class, interesting. **Kohana**? I already saw this word during security testing of _Seedr_. But where?”

Thanks to Burp Suite Professional I quickly found the first mention of Kohana in Proxy history, opened that page, and observed a detailed error page.

![](https://miro.medium.com/max/1034/1*DmjHfzdQKNMVUkRFNOWGTQ.png)

_Vanilla Burp Suite Community can’t do this_

![](https://miro.medium.com/max/1400/0*Zt8n_iVkGQE354ed)

_Paradise for security tester_

Here I will make a small digression to give you some information about _Seedr_ and where _v2.nativeroll.tv_ came from. I should note that I may be wrong but here are my thoughts I had at that moment.

_Seedr_ and _Nativeroll_ are both platforms for Video advertising. _Seedr_ had an old fashioned design, so I guess that it was created long before _Nativeroll_. Both platforms were bought by _Mail.Ru Group_, probably somehow merged and listed on HackerOne at the same scope. So, _v2.nativeroll.tv/api/_, _api.seedr.ru_, _api-stage.seedr.ru_, _player.seedr.ru_ shared the same code base. Hope now it’s more clear.

Ok, let’s return to the beautiful error page. _Environment_, _Included files_, _Loaded extensions_ — looks juicy. Here is what I observed after clicked on _Included files_ link:

![](https://miro.medium.com/max/1400/0*nqz7kdF68b5jOhqC)

There were almost 90 included files, which in fact were different classes loaded with something like `autoload.php`. Is Kohana some sort of CMS or framework? Yes, it is. After some googling I found GitHub repository [https://github.com/koseven/kohana/](https://github.com/koseven/kohana/) which looks deprecated:

![](https://miro.medium.com/max/1400/0*1ANtHeWFKu9hReDb)

Because _v2.nativeroll.ru_ and _api.seed.ru_ share the same code base, I successfully triggered Error exception on _api.seedr.ru_ with the same payload (_https://api.seedr.ru/<svg>_) and got the same result.

For triggering Error exception exactly on _api.seedr.ru/video_ (endpoint which I attacked), I took the result from _http://vimeo.com/api/v2/video/123459.php_ and modified value of description attribute from `string`  to `array`.

![](https://miro.medium.com/max/1206/0*r3lPU6YDPAAcvC1S)

```
a:1:{i:0;a:23:{s:2:”id”;i:123456;s:5:”title”;s:30:”London Tornado — The aftermath”;s:11:”description”;**a:1:{i:0;i:1337;}**s:3:”url”;s:24:”https://vimeo.com/123456";s:11:"upload\_date";s:19:"2006-12-14 06:53:32";s:15:”thumbnail\_small”;s:111:”https://i.vimeocdn.com/video/46783763-254c2bbf4211bd6657c59e96a682169c8e74fc56e96ebb4e0a2882b103cab878-d\_100x75";s:16:"thumbnail\_medium";s:112:"https://i.vimeocdn.com/video/46783763-254c2bbf4211bd6657c59e96a682169c8e74fc56e96ebb4e0a2882b103cab878-d\_200x150";s:15:"thumbnail\_large";s:108:"https://i.vimeocdn.com/video/46783763-254c2bbf4211bd6657c59e96a682169c8e74fc56e96ebb4e0a2882b103cab878-d\_640";s:7:"user\_id";i:146861;s:9:"user\_name";s:11:"wordtracker";s:8:"user\_url";s:29:"https://vimeo.com/wordtracker";s:19:"user\_portrait\_small";s:51:"https://i.vimeocdn.com/portrait/defaults-blue\_30x30";s:20:"user\_portrait\_medium";s:51:"https://i.vimeocdn.com/portrait/defaults-blue\_75x75";s:19:"user\_portrait\_large";s:53:"https://i.vimeocdn.com/portrait/defaults-blue\_100x100";s:18:"user\_portrait\_huge";s:53:"https://i.vimeocdn.com/portrait/defaults-blue\_300x300";s:21:"stats\_number\_of\_likes";i:11;s:21:"stats\_number\_of\_plays";i:122560;s:24:"stats\_number\_of\_comments";i:12;s:8:"duration";i:32;s:5:"width";i:320;s:6:"height";i:240;s:4:"tags";s:0:"";s:13:"embed\_privacy";s:8:"anywhere";}}
```

During script execution the `htmlspecialchars()` function expected a `string`, but got an `array`, which caused an Error exception with partly disclosed template and stack trace:

![](https://miro.medium.com/max/1400/0*se6X4lVo3aKoLr5R)

![](https://miro.medium.com/max/1400/0*B84aJ991dPsR-GIU)

There was a composer autoload script, as I thought. Among these included files I highlighted several, which can be useful during deserialization:

*   Guzzle (_/var/www/sentry/vendor/guzzlehttp/…_)
*   Swift Mailer (_MODPATH/email/vendor/swiftmailer/…_)
*   Symfony (_/var/www/sentry/vendor/symfony/…_)
*   Mustache (_MODPATH/kostache/vendor/mustache/…_)
*   Sentry (_/var/www/sentry/vendor/sentry/…_)

I knew that PHPGGC has some gadget chains for Guzzle, Swift Mailer and Symfony. After I built and tested the payloads on _api-stage.seedr.ru_ I got new error. For example, an attempt with Guzzle payload returned the following error: `FnStream should never be unserialized`_._ Which indicated that the script used an already [patched version](https://github.com/guzzle/psr7/blob/master/src/FnStream.php#L60):

![](https://miro.medium.com/max/1400/0*4IulKCyvBryD6Bgm)

Swift Mailer and Symfony didn’t work at all and analysis of Mustache and Sentry code on Github also didn’t bear any fruit, so third-party libraries wouldn’t help me. It was time to dive into Kohana.

Search for magical methods, like `__wakeup()`_,_ `__destruct()`_,_ `__toString()`, in Kohana repository was empty:

![](https://miro.medium.com/max/1400/0*_qTSeiMOJsvaQx9O)

But this Kohana repository has a _system_ directory which in fact is a dedicated [repository Kohana Core](https://github.com/kohana/core/tree/bdbe81afb5a09cee4269d2e2210a0d293265231a):

![](https://miro.medium.com/max/1400/0*SjrcVKdSbF-p-lho)

![](https://miro.medium.com/max/1400/0*wTstXMB2xOlvMDAD)

Let’s try to search for magical methods in this repository. There were almost no results for `__destruct()`_,_ `__wakeup()`, but results for `__toString()` were reassuring:

![](https://miro.medium.com/max/1400/0*VMUltckTWkhdy4L0)

I briefly overlooked the findings, but [_classes/Kohana/View.php_](https://github.com/kohana/core/blob/bdbe81afb5a09cee4269d2e2210a0d293265231a/classes/Kohana/View.php) and its `[render()](https://github.com/kohana/core/blob/bdbe81afb5a09cee4269d2e2210a0d293265231a/classes/Kohana/View.php#L346)` function immediately caught my attention.

I should say I have some experience with backend development in the past, especially with PHP. I developed a few projects with Laravel and already knew about its MVC (Model-View-Controller) pattern. For rendering of Views Laravel uses an engine called Blade. Because such rendering engines usually load some templates (files) for rendering, I guessed that maybe somehow I can pass to function my own file or my own content.

Let’s take a look at the `**render()**` function closely. Function `**render()**` accepts 1 argument called `**$file**` and then calls function  `**capture():**`

```
public function render($file = NULL)  
{  
    if ($file !== NULL)  
    {  
        $this->set_filename($file);  
    } if (empty($this->_file))  
    {  
        throw new View_Exception('You must set the file to use within your view before rendering');  
    } // Combine local and global data and capture the output  
    return **View::capture($this->\_file, $this->\_data)**;  
}
```

In my case function `render()` calls without argument and let me bypass `set_filename()` function which also checks existence of the `$file` in the `views` directory:

```
public function set_filename($file)  
{  
    if (($path = **Kohana::find_file(‘views’, $file))** === FALSE)  
    {  
        throw new View_Exception(‘The requested view :file could not be found’, array(‘:file’ => $file,));  
 } // Store the file path locally  
    $this->_file = $path; return $this;  
}
```

Thus I call `capture()` function with `$this->_file` variable:

```
public function render($file = NULL)  
{  
    ... // Combine local and global data and capture the output  
    return **View::capture($this->\_file, $this->\_data)**;  
}
```

As it says in the comment `**capture()**` function combines local and global data and captures the output. For example, you can render a template file for email and use username as a variable there.

```
protected static function capture($kohana\_view\_filename, array $kohana\_view\_data)  
{  
    // Import the view variables to local namespace  
    extract($kohana\_view\_data, EXTR_SKIP); if (View::$\_global\_data)  
    {  
        // Import the global view variables to local namespace  
        extract(View::$\_global\_data, EXTR\_SKIP | EXTR\_REFS);  
    } // Capture the view output  
    ob_start(); try  
    {  
        // Load the view within the current scope  
        include $kohana\_view\_filename;  
    }  
    catch (Exception $e)  
    {  
        // Delete the output buffer  
        ob\_end\_clean(); // Re-throw the exception  
        throw $e;  
    } // Get the captured output and close the buffer  
    return ob\_get\_clean();  
}
```

`**capture()**` function accepts 2 arguments: `**$kohana_view_filename**` and `**$kohana_view_data**`**_._** Some of you probably already spotted the function that potentially can be abused during deserialization:

```
...  
try  
{  
    // Load the view within the current scope  
    include $kohana\_view\_filename;  
}  
...
```

`**include()**`**!**  It already smells like LFI and RCE. But do I have control over `**$kohana_view_filename**`**_?_**

Yes, I do! Because `$kohana_view_filename` is `$this->_file` in my context and `_file` is an attribute of _View_ class.

```
class Kohana_View {  
    // Array of global variables protected static   
    $\_global\_data = array();  
    ...  
    // View filename   
    protected $_file;    
    // Array of local variables   
    protected $_data = array();  
    ...  
}
```

At that moment I had all the elements for successful unsafe deserialization:

*   I controlled the input;
*   I had a magical method `**__toString()**`  of _View_ class with a useful function `**include()**`.
*   The class _View_ was loaded.

Bingo!

After some time I created gadget and chain for PHPGGC locally, which later [were committed and added](https://github.com/ambionics/phpggc/tree/master/gadgetchains/Kohana/FR/1) to main repository:

```
<?phpnamespace GadgetChain\\Kohana;class FR1 extends \\PHPGGC\\GadgetChain\\FileRead  
{  
    public static $version = ‘3.*’;  
    public static $vector = ‘__toString’;  
    public static $author = ‘byq’;  
    public static $information = ‘include()’; public function generate(array $parameters)  
    {  
        return new \\View($parameters\[‘remote_path’\]);  
    }  
}<?phpclass View   
{  
    protected $_file; public function \_\_construct($\_file) {  
        $this->\_file = $\_file;  
    }  
}
```

Then I just ran PHPGGC and got following serialized object:

![](https://miro.medium.com/max/764/0*qyNjDhRsVmK6dTt2)

Hosted payload on a controlled server, sent a request and …

![](https://miro.medium.com/max/1400/0*IxazTbthZk7j-nKb)

Oookey, at least it was something new. But what I was hoping for? I used not `__wakeup()` or `__destruct()` methods which trigger at the moment of Object creation and destruction respectively, I used `__toString()`. As per [PHP docs](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring):

```
The __toString() method allows a class to decide how it will react when it is treated like a string. For example, what echo $obj; will print.
```

So I somehow should output my _View_ object. Actually it wasn’t difficult to understand that I should provide my _View_ object as a value of title or description attribute, the trick which I made before with an `array`  to trigger an Error exception. Here is what my payload looked like:

```
a:1:{i:0;a:23:{s:2:”id”;i:123456;s:5:”title”;s:30:”London Tornado — The aftermath”;s:11:”description”;**O:4:”View”:1:{s:8:”*_file”;s:11:”/etc/passwd”;}**s:3:”url”;s:24:”https://vimeo.com/123456";s:11:"upload\_date";s:19:"2006-12-14 06:53:32";s:15:”thumbnail\_small”;s:111:”https://i.vimeocdn.com/video/46783763-254c2bbf4211bd6657c59e96a682169c8e74fc56e96ebb4e0a2882b103cab878-d\_100x75";s:16:"thumbnail\_medium";s:112:"https://i.vimeocdn.com/video/46783763-254c2bbf4211bd6657c59e96a682169c8e74fc56e96ebb4e0a2882b103cab878-d\_200x150";s:15:"thumbnail\_large";s:108:"https://i.vimeocdn.com/video/46783763-254c2bbf4211bd6657c59e96a682169c8e74fc56e96ebb4e0a2882b103cab878-d\_640";s:7:"user\_id";i:146861;s:9:"user\_name";s:11:"wordtracker";s:8:"user\_url";s:29:"https://vimeo.com/wordtracker";s:19:"user\_portrait\_small";s:51:"https://i.vimeocdn.com/portrait/defaults-blue\_30x30";s:20:"user\_portrait\_medium";s:51:"https://i.vimeocdn.com/portrait/defaults-blue\_75x75";s:19:"user\_portrait\_large";s:53:"https://i.vimeocdn.com/portrait/defaults-blue\_100x100";s:18:"user\_portrait\_huge";s:53:"https://i.vimeocdn.com/portrait/defaults-blue\_300x300";s:21:"stats\_number\_of\_likes";i:11;s:21:"stats\_number\_of\_plays";i:122560;s:24:"stats\_number\_of\_comments";i:12;s:8:"duration";i:32;s:5:"width";i:320;s:6:"height";i:240;s:4:"tags";s:0:"";s:13:"embed\_privacy";s:8:"anywhere";}}
```

Once again I updated payload on controlled server, sent request and finally:

![](https://miro.medium.com/max/1400/0*w76oIrYVSY8UwvBh)

I got the content of `/etc/passwd` inside `og:description` meta tag. Awesome, local file read is better than semi-blind SSRF, but it’s still not a RCE.

Local file inclusion is such a rare gem in modern web applications that I had to remember where I can store my RCE payload to `include()` it later. The most common techniques are:

*   file upload (in my case application didn’t have such functionality);
*   logs (apache, nginx, mail, ssh, …);

![](https://miro.medium.com/max/1400/0*3e8hDouUzohsGoS0)

![](https://miro.medium.com/max/1400/0*lm12I7PAdSDyhUkX)

*   _/proc/*/fd_, …;

![](https://miro.medium.com/max/1400/0*TN89wta0YTO43yR_)

*   session file;

![](https://miro.medium.com/max/1400/0*1-PHaH-CyIPIQpRl)

As you can understand I tried almost everything and nothing worked.

It’s time to take a few steps back, namely to the error related to “no space left on device”:

![](https://miro.medium.com/max/1400/0*nWE0f1SXbrO4qb6F)

From that error, I could extract a path to some log file:  `/application/logs/2021/12/20.php`_._ After I tried to open _https://api.seedr.ru/application/logs/2021/12/20.php_ in the browser I got the following error: _No direct script access._ Actually, almost every PHP file in Kohana framework has such a string in the beginning:

![](https://miro.medium.com/max/1400/0*SfbU3i0tBAqrpZiw)

Seems like I can’t access log files with the `.php` extension directly from the browser. I gave a try at the staging host: [_http://api-stage.seedr.ru/application/logs/2021/12/20.php_](http://api-stage.seedr.ru/application/logs/2021/12/20.php) and to my surprise, I received a 404 HTTP status code. I don’t know what pushed me to do that, but I changed the `.php` extension to `**.log**`_,_ and …

![](https://miro.medium.com/max/1400/0*ySawx41RKuf6XazV)

Yes, I got the huge log file which even froze my Burp Suite a little. I should note that such a trick didn’t work on the production host _api.seedr.ru_. I can only guess that _Seedr_ developers changed something on the staging environment to make accessing log files easier. But as usual, it led to a security issue.

One more time I opened a new door and started exploring it. Do you still remember how I triggered the Error exception the first time? Here is a record about it in the log file:

![](https://miro.medium.com/max/1400/0*mxJoD-5sCyi72kXM)

After short analysis of log file I poisoned it with such a record:

![](https://miro.medium.com/max/1400/0*_4syAwRVKHUPJ6jV)

With PHPGGC I created a new serialized _View_ object with __file_ attribute `/var/www/t1.seedr.backend/application/logs/2021/12/20.log`, hosted it on the controlled server, sent request and got following error:

![](https://miro.medium.com/max/1400/0*W2XNzgb5PD9H2j1X)

Seems like because the log file was too huge (>200000 lines), some function  failed on one of the `?` symbols, threw an exception and stopped the execution of the script. From [PHP docs](https://www.php.net/manual/en/language.oop5.magic.php#object.tostring) I learned that:

![](https://miro.medium.com/max/1400/0*btQJtEuAUtVLaCQ4)

Because the log file for December 20 was ruined with my unsuccessful payload, all other tests on that host were useless so I moved to the local environment. Hours of debugging and tests with `include()` and log file did not lead to the desired result.

During the morning shower, I remembered one more awesome article from [Charlese Fol](https://twitter.com/cfreal_) [Laravel <= v8.4.2 debug mode: Remote code execution (CVE-2021–3129)](https://www.ambionics.io/blog/laravel-debug-rce). It uses the technique with multiple base64 decoding feature which ignore not base64. Firstly, I read about it from [Orange Tsai](https://twitter.com/orange_8361) [blog](http://blog.orange.tw/2018/10/). My idea was to poison a log with multiple base64 encoded PHP payload, then decode it with multiple `convert.base64-decode` filters inside the `include()` function to bypass that exception with the `?` symbol. But because it was a sleepless night my brain didn’t work well and I completely forgot that in the Laravel case it abused `file_get_contents()` and `file_put_contents()` functions chain with the same arguments inside it which allowed Charlese to rewrite log. I also forgot about this limitation:

![](https://miro.medium.com/max/1400/0*NsaXMtVDsvScNwyi)

Screenshot from Charlese’s article

Because of the predictable log file path (`/application/logs/2021/12/20.log`) I downloaded a few log files for previous days and planned to poison the log for December 21 at the beginning of the day until it wasn’t too huge.

I posted all collected information to the H1 report and had a full day before December 21. Without wasting time I tried to exploit my finding on the production environment _api.seedr.ru_, because all my last tests were on _api-stage.seedr.ru_. One more time with the help of PHPGGC I created a _View_ object with `_file` attribute `/etc/passwd`, hosted it on the controlled server and … I didn’t observe the content of the `/etc/passwd` file in the response. I repeated the same steps on _api-stage.seedr.ru_ and all worked fine. “Oops, does it only work on a staging environment?”

Here I must confess that when I generated serialized object with PHPGGC I modified it a little:

![](https://miro.medium.com/max/764/0*H3lcfcB6bhuJUJCP)

Does the `*_file`  string has really 8 symbols? No, it has only 6. That is what I modified every time and it worked flawlessly on _api-stage.seedr.ru._ Later in the stack trace, I noticed the following:

![](https://miro.medium.com/max/924/0*NHh2p40s_YsIB1Z1)

Value of protected `_file`  attribute is NULL, but for some reason _View_ object has public `*_file` attribute with my payload. Probably PHP experts already understood the reason for such behavior, but I had to spend some time dealing with this problem.

As you could notice from screenshots for storing payloads I used [https://webhook.site/](https://webhook.site/), it’s a fast and easy solution for receiving incoming HTTP connections and hosting the payloads. Unfortunately that time it played a bad joke on me. The thing is [to store protected value in serialized string PHP use null characters (_\\0_) around “*” symbol](https://stackoverflow.com/a/37468673), that is why `*_file` has 8 symbols:

![](https://miro.medium.com/max/1092/0*nhYKCUrgwRL1cnx2)

Because I just copy-pasted payload to _webhook.site_, it didn’t store these null characters and delivered to `unserialize()` public attribute `*_file`. To resolve such a problem I just hosted a serialized string with null bytes on my server. Now _vimeo.com_ redirects requests to my server where I just `echo()` payload with null characters. After I was able to observe the content of `/etc/passwd` on _api.seedr.ru_ I one more time returned to analyze downloaded log files.

Log files had a lot of record types, but only a few could be used to store payload and most of them required authentication. Even during the first analysis of log files, I noticed the following record type:

![](https://miro.medium.com/max/1400/0*DU0xaFc7cGLRauq_)

What was good about this record type, was that it stored payload only once per record and didn’t repeat it several times like my previous attempt. I also noted a possible injection point: _user-agent_. But the problem was that I didn’t know how to generate such a record in a log and what endpoint should I access. I “grepped” logs with my IP and discovered that today’s log file already has such a record with my IP, which meant that I definitely touched the required endpoint. By that time my Burp Proxy history had already more than 40000 records, so it was kind of difficult to find a proper endpoint. Comparing the time of the record with my IP and the activity that I performed at that time I understood that the record was probably generated during my scan with _dirsearch_. I rerun it and after some time the endpoint which generated such a record was found: _api-stage.seedr.ru/inc._

On the local environment, I hid the new payload in a test log file, `include()` it, and got the output of the bash command. All that was left was to wait until December 21 and fresh log file, because on December 20 log files for _api.seedr.ru_ and _api-stage.seedr.ru_ were poisoned with my unsuccessful payloads.

The next day I poisoned the log with the following request:

![](https://miro.medium.com/max/1400/0*OQ3bz6Yhtu79lQ9Q)

Generated the payload, hosted on the server, sent request …

![](https://miro.medium.com/max/1400/1*mQiTZuixdPHSLIK0O04SxQ.png)

Yeap, I forgot to change `$argv[1]` to `$_GET[1]`  after local tests… Looking forward to waiting one more day I remembered that for today I have one more attempt at _api-stage.seedr.ru:_

![](https://miro.medium.com/max/1400/0*40yjy3DA59MnyDOX)

_This is how victory looks_

![](https://miro.medium.com/max/1400/0*5MIPw-u1XuZbqsQL)

[https://i.imgur.com/DrNEGRH.png](https://i.imgur.com/DrNEGRH.png) without compression