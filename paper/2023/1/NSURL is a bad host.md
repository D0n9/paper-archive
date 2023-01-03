# NSURL is a bad host
[Articles index](https://lapcatsoftware.com/articles/index.html "The Desolation of Blog")

### April 10 2021 by Jeff Johnson

If NSURL invites you to a party, you should pass. Why? Because NSURL parties are passé. To be more specific, NSURL is based on an obsolete RFC. (Note: RFC is an acronym for Read the F-ing Commandment, while NS is an acronym for No Swift.) The obsolescence of NSURL is all explained in the class documentation for NSURL.

Correction: The obsolescence of NSURL is _not_ explained in the class documentation for NSURL! Rather, it's explained in the class documentation for NSURLComponents:

> The [`NSURLComponents`](https://developer.apple.com/documentation/foundation/nsurlcomponents?language=objc) class is a class that is designed to parse URLs based on [RFC 3986](http://www.ietf.org/rfc/rfc3986.txt) and to construct URLs from their constituent parts. Its behavior differs subtly from the [`NSURL`](https://developer.apple.com/documentation/foundation/nsurl?language=objc) class, which conforms to older RFCs. However, you can easily obtain an [`NSURL`](https://developer.apple.com/documentation/foundation/nsurl?language=objc) object based on the contents of a URL components object or vice versa.

Its behavior differs subtly. How subtly? Let's look at an example:

```
#import <Foundation/Foundation.h>
int main(int argc, const char *argv[]) {
  @autoreleasepool {
    NSString *ipv6 = @"http://[FEDC:BA98:7654:3210:FEDC:BA98:7654:3210]:80/index.html";
    NSURL *url = [NSURL URLWithString:ipv6];
    NSURLComponents *components1 = [NSURLComponents componentsWithString:ipv6];
    NSURLComponents *components2 = [NSURLComponents componentsWithURL:url resolvingAgainstBaseURL:NO];
    NSLog(@"%@", [url host]);
    NSLog(@"%@", [components1 host]);
    NSLog(@"%@", [components2 host]);
  }
  return 0;
}
```

The URL `http://[FEDC:BA98:7654:3210:FEDC:BA98:7654:3210]:80/index.html` contains an IPv6 address. You all support IPv6 now amirite? According to [RFC 3986](http://www.ietf.org/rfc/rfc3986.txt), a host must enclose an IPv6 address in square brackets. \[Much like an Objective-C method call - Ed.\]

```
host          = IP-literal / IPv4address / reg-name
IP-literal    = "[" ( IPv6address / IPvFuture  ) "]"
```

Now let's see the output of previous example code:

```
FEDC:BA98:7654:3210:FEDC:BA98:7654:3210
[FEDC:BA98:7654:3210:FEDC:BA98:7654:3210]
[FEDC:BA98:7654:3210:FEDC:BA98:7654:3210]
```

Wow. So they differ subtly in that NSURLComponents is right and NSURL is wrong!

In fairness, NSURL was introduced in 1997, prior to the publication of RFC 3986 in 2005, whereas NSURLComponents was introduced in 2013. At this point, I feel that Apple ought to deprecate the obsolete/broken methods of NSURL, because there isn't a clear enough warning to unsuspecting (mis-)users. The NSURL API does mention RFC 1808, but who has RFC numbers memorized? Conforming to RFC sounds good in itself, and the reader of the docs is not going to know offhand that 1808 is not the RFC you're looking for. Anyway, given the "subtly different" behavior, you should probably treat NSURL as opaque, and if you need to parse the components of a URL, use NSURLComponents instead. The paradox of Apple's documentation is that you would only know that you need to use NSURLComponents if you already read the documentation for NSURLComponents.

By the way, the same behavior exists in Swift. (Notice all the ! characters I needed to get this to compile!)

```
import Foundation
let ipv6 = "http://[FEDC:BA98:7654:3210:FEDC:BA98:7654:3210]:80/index.html"
let url = URL(string:ipv6)
let components1 = URLComponents(string:ipv6)
let components2 = URLComponents(url:url!, resolvingAgainstBaseURL:false)
print(url!.host!)
print(components1!.host!)
print(components2!.host!)
```

[Jeff Johnson](https://lapcatsoftware.com/) ([My apps](https://underpassapp.com/), [PayPal.Me](https://www.paypal.me/JeffJohnsonWI))

[Articles index](https://lapcatsoftware.com/articles/index.html "The Desolation of Blog")

![](chrome-extension://mapjgeachilmcbbokkgcbgpbakaaeehi/assets/check.svg)
The action has been successful

![](chrome-extension://mapjgeachilmcbbokkgcbgpbakaaeehi/assets/check.svg)
The action has been successful

set 限制解除