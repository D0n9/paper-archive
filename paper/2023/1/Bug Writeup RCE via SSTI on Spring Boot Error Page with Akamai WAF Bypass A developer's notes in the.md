# Bug Writeup: RCE via SSTI on Spring Boot Error Page with Akamai WAF Bypass | A developer's notes in the world of security research and bug bounty, by pmnh
Summary[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#summary)
------------------------------------------------------------------------------

This writeup talks about a successful collab that I did with [Dark9T](https://bugcrowd.com/Dark9T) ([@UsmanMansha](https://twitter.com/UsmanMansha420)) on a private program hosted on Bugcrowd. We ended up able to bypass Akamai WAF and achieve Remote Code Execution (P1) using Spring Expression Language injection on an application running Spring Boot. This was the 2nd RCE via SSTI we found on this program, after the 1st one, the program implemented a WAF which we were able to bypass in a different part of the application. Read on to find out how we did it!

![](https://h1pmnh.github.io/images/2022-writeup-spring-el-waf/resolved_bug_screenshot.png)

Figure 1: Screenshot of Resolved Bug

Intro[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#intro)
--------------------------------------------------------------------------

Usman reached out to me on a Slack server where we are both members. They had found a potential SSTI but were not able to exploit it due to an Akamai WAF:

![](https://h1pmnh.github.io/images/2022-writeup-spring-el-waf/slack_message.png)

Figure 2: Screenshot of Slack Message

After a quick look, this seemed to be a case of the famous Spring Boot Error page issue [described on Github here](https://github.com/spring-projects/spring-boot/issues/4763) \- note that there was never a CVE issued for this as far as I am aware. This vulnerability has been covered in various forms for example by _0xdeadpoool_ on their blog [here](https://f002.backblazeb2.com/file/sec-news-backup/files/writeup/deadpool.sh/_2017_RCE_Springs_/index.html).

The basic principle of this bug is that the vulnerable version of Spring Boot will render the error message from the thrown `Exception` into the page itself using an SpEL (Spring Expression Language) expression. The vulnerable version of the Spring Boot framework will allow recursive evaluation of this expression, thus an error message which contains a valid SpEL expression (e.g. `$(7*7)`) would be evaluated at the the time the error page is rendered.

In this case we could see the `q` parameter of the vulnerable URL supported injection of the type `${x*y}` and returned a mathematical result in the error text:

![](https://h1pmnh.github.io/images/2022-writeup-spring-el-waf/spel_expression_simple.png)

Figure 3: Screenshot of Initial SpEL test

Steps with RCE via SpEL[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#steps-with-rce-via-spel)
--------------------------------------------------------------------------------------------------------------

If you haven't had experience with this type of vulnerable application before, I'd strongly suggest some practice using an application such as [https://github.com/jzheaux/spel-injection](https://github.com/jzheaux/spel-injection) where you can experiment with how SpEL is constructed and handled (and potentially secured) within Spring applications. While this application doesn't deal directly with this specific vulnerability, SpEL is used so often in the Spring Ecosystem it's worth some practice and familiarity with the code.

This blog won't introduce you to Spring Expression Language as the topic is quite complex, essentially it's a language which allows context-based navigation of Spring objects, similar to other server-side templating languages. It's used many places in various Spring Framework components and the exact extent of objects and data available depends a lot on where it's used. Typically you can execute Java methods, construct objects, etc. - not as powerfully as FreeMarker or Velocity, but similar in risk profile. You can read about SpEL and its syntax [in the Spring reference documentation](https://docs.spring.io/spring-framework/docs/3.0.x/reference/expressions.html).

Generally the goal with SpEL is to end up with an invocation of the methods `java.lang.Runtime.exec` or `java.lang.ProcessBuilder.start` which will allow execution of an OS command of the attacker's choosing, using an expression something like the following:

```
1${T(java.lang.Runtime).getRuntime().exec("<my command here>")} 
```

java

If you want the output of the command, the expression gets a bit more complex, but let's start here.

A Quick Note - Time / Effort Spent[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#a-quick-note---time--effort-spent)
-----------------------------------------------------------------------------------------------------------------------------------

Folks who know me know that I am primarily a manual tester, relying on my extensive development/architecture experience rather than brute force to find tough bugs. Although reading a blog post may make it appear that a bug was obvious or a particular path was obvious, just to give some statistics, getting from the initial Slack message from Usman to full RCE took me:

*   Approximately 500 hand-crafted attempts to bypass the WAF
*   Approximately 14 hours of _wall clock time_ from the initial attempt to the first successful RCE (execution of the `uname -a` command) - note that I took breaks to eat, take a walk, think about solutions etc. - it wasn't 14 hours straight!

I'm including these because it's often the case that blog posts make this kind of bug "seem" a lot easier than it actually is, leading readers down the dark path of impostor syndrome etc., just reinforcing that even if you know what you're doing, sometimes bugs are really tough! Don't give up! 😄

Step 1 - Try the Obvious[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#step-1---try-the-obvious)
----------------------------------------------------------------------------------------------------------------

First off we had to determine how to reach the `java.lang.Runtime` class, so that we could get an instance of it, on which to invoke the `exec` method. We tried the most obvious `${T(java.lang.Runtime)}` \- which is SpEL shorthand for referencing a Java class by name, and of course it was blocked by the Akamai WAF:

![](https://h1pmnh.github.io/images/2022-writeup-spring-el-waf/t_operator_waf_block.png)

Figure 4: First Attempt

Since Akamai WAF was in the way, I suspected this would not work, but when trying to work around a WAF it's really important to build up from small things that you know work, to larger and more complex payloads. This is true for RCE, SQLi, XSS, or any complex payload when trying to avoid WAF rules, very often WAFs are coded to recognize obvious payloads but (as we will see) can't figure out complex payloads.

Step 2 - Figure out how to get an arbitrary Class[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#step-2---figure-out-how-to-get-an-arbitrary-class)
------------------------------------------------------------------------------------------------------------------------------------------------------------------

Typically the next stage of a Java-based code injection vulnerability is to figure out how to get a reference to an arbitrary `Class`, from which we can use direct method invocation or reflection-based invocation to get at the method we want.

The easiest method is to do something like the following (which worked in this case):

Response:

This is a good sign, we know we can access the `java.lang.Integer` `Class` object (if you need a refresher \[https://stackoverflow.com/questions/1215881/the-difference-between-classes-objects-and-instances\](this SO answer is a good start)), and from here we should be able to get to the `forName` method to instantiate an arbitrary class. Let's try it!

```
1${2.class.forName("java.lang.String")} 
```

java

Response:

```
1<H1>Access Denied</H1> 2 3You don't have permission to access ... 
```

html

As expected, the obvious payload using the `forName` method with a string did not work and was easily detected by the Akamai WAF. In the next round of exploration I was able to determine that some sort of transformation was being applied to both single and double quotes that caused expressions using either of these characters to be malformed. Thus even if we could reach the `Class.forName` method, we wouldn't be able to take the straightforward route of something like `${2.class.forName("java.lang.Runtime")...}` but instead need to find some other way to construct the name of the Class to be instantiated.

Step 3 - Figure out how to get an arbitrary String[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#step-3---figure-out-how-to-get-an-arbitrary-string)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------

I knew that being able to build an arbitrary string would be required to achieve the full RCE for multiple reasons:

*   Name of class to be instantiated or referenced
*   Name of method (most likely `.exec()` is also blocked by the WAF)
*   Command to be executed

Keep in mind that I can't use quote characters of either type, so straightforward string concatenation is not possible in this circumstance. I needed to find a way to get from an integer value (ASCII or hex) to a character, and then concatenate characters to form a `String`.

> I've run into this situation a number of times, either solo or in collabs and I always refer back to the [Java API Documentation](https://docs.oracle.com/javase/8/docs/api/) which has so much useful information about available methods and classes, although I know many of the core Java classes by heart, it's often been the case that I find some hidden gem that will do exactly what I need!

A few obvious choices in the Java standard library:

*   `java.lang.String` constructor, taking a byte array (as inspired by [mykong](https://mkyong.com/java/how-do-convert-byte-array-to-string-in-java/) and [Bealdung](https://www.baeldung.com/java-string-to-byte-array))
*   `java.lang.Character.toString` method, described in [Javadoc](https://docs.oracle.com/javase/8/docs/api/java/lang/Character.html#toString-char-)

After some experimentation I determined that it was basically not possible to invoke any constructor, because both methods of invoking a constructor in SpEL, either `new`, `T()`, or through reflection and `newInstance` was also blocked by the WAF.

So it seemed like the `java.lang.Character.toString` method was the way to go, only one problem...

Step 4 - Figure out how to get a reference to a `java.lang.Character` class[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#step-4---figure-out-how-to-get-a-reference-to-a-javalangcharacter-class)
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Since `java.lang.Character.toString` is a static method on the `java.lang.Character` class, I simply needed a reference to an object of this type to be able to reach the method. Because SpEL is dynamic, I don't believe it supports casting as you could in static Java code, e.g. `(char)99` \- and unfortunately `java.lang.Class` was blocked by the WAF so I couldn't use the `java.lang.Class.cast` method.

So I ended up with the following chain:

*   Figure out how to get a reference to a `String` object
*   Call the `java.lang.String.charAt` method on that object (which returns a `java.lang.Character`)
*   Invoke the `toString` static method on this character - since it's a static method it doesn't matter what the value of the `Character` is

Thus I _finally_ had my gadget required to build an arbitrary `String`:

```
1${(2.toString()+2).charAt(0).class.toString(99)} 
```

java

Response

Note that `99` is the ASCII value for the character `c`. Success!

Since the `+` character was allowed through the WAF and in this context I was able to now build strings using this method of individual character concatenation.

Step 5 - Build attack payload[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#step-5---build-attack-payload)
--------------------------------------------------------------------------------------------------------------------------

So, now we have one ingredient we need - building arbitrary `String` \- we need one more, which is a way to invoke the `java.lang.Runtime.exec` method. I ended up using a technique similar to the one [described here](https://pulsesecurity.co.nz/articles/EL-Injection-WAF-Bypass), basically the following:

*   Use reflection to get access to the `Class.forName` method
*   Build a String with the value `java.lang.Runtime` to pass to `forName`
*   Use reflection to get access to the `java.lang.Runtime.getRuntime` method (required to get an instance of the class to invoke a method)
*   Build a String with the value `exec` and/or use reflection to find the `exec` method of the `java.lang.Runtime` class
*   Build a String with the RCE payload value to pass to the `exec` method

In this phase of the exploitation, I spent a lot of time iterating over the output from various reflection calls. This is especially important because different JVMs will return different values, particularly when you are using the `java.lang.Class.getMethods` reflection technique.

> Don't invoke reflected methods blindly! There are dangerous methods on `java.lang.Runtime` such as `shutdown` which will immediately terminate the JVM!

Step 6 - (Time Wasted) Trying to work around `GET` length restrictions[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#step-6---time-wasted-trying-to-work-around-get-length-restrictions)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

At this point I realized that for some payloads I would end up with a really long payload if I'm constructing a long RCE command e.g. an `nslookup` or similar. My payload for a single character `c` was 45 bytes long (`${(2.toString()+2).charAt(0).class.toString(99)}`)!

With a `GET` request maximum length enforced by some browsers and/or servers at approximately ~2kb this meant the longest `String` I could build might be only about 45 characters long - a big problem!

At this point I ended up going down a bit of a chase to figure out how to more efficiently create a `String` from a list of bytes. I tried a bunch of things and almost had one working using the neat [collection projection](https://docs.spring.io/spring-framework/docs/3.0.x/reference/expressions.html#d0e12113) feature of SpEL, but unfortunately I was blocked by a Spring bug in the version this target was running. Ultimately I couldn't find any more efficient method of building the `String` character by character.

In this sense I ended up getting lucky, the server accepted a `GET` request longer than 2kb (final payload was just under 3kb), and typically you are safe before 4kb on most servers.

Step 7 - Assembling the final payload[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#step-7---assembling-the-final-payload)
------------------------------------------------------------------------------------------------------------------------------------------

At this point after Step 5 I basically had all the pieces I needed to build the final payload, which was essentially a translation of the following payload:

```
1org.apache.commons.io.IOUtils.toString(java.lang.Runtime.getRuntime().exec("uname -a").getInputStream()) 
```

java

![](https://h1pmnh.github.io/images/2022-writeup-spring-el-waf/final_payload_result.png)

Figure 5: Final Payload

I'm not going to supply the payload in text form because I don't want someone blindly copy-pasting into a context where it likely won't work anyway, but hopefully this post gave you the methodology to build your own payload to bypass a WAF and server-side restrictions.

Final Thoughts[](https://h1pmnh.github.io/post/writeup_spring_el_waf_bypass/#final-thoughts)
--------------------------------------------------------------------------------------------

I find WAF bypasses on critical vulnerabilities such as RCE and SQL Injection some of the most fun bugs to work on. Of course the rewards are good, but these sort of bugs really require deep knowledge of _why_ a particular bug works, and the context in which it executes.

In this case, deep knowledge of Java and SpEL capabilities was required to construct a payload that would both bypass the Akamai WAF as well as work in the context where it was executing.

I hope you enjoyed this writeup. If you encounter this type of injection and you need help bypassing a WAF, feel free to [DM me on Twitter](https://twitter.com/pmnh_) and I'm always happy to collab if you have a confirmed injection but can't escalate it.