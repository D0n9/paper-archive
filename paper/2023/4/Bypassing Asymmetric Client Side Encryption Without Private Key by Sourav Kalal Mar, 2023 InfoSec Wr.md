# Bypassing Asymmetric Client Side Encryption Without Private Key | by Sourav Kalal | Mar, 2023 | InfoSec Write-ups --- 在没有私钥的情况下绕过非对称客户端加密 |通过 Sourav Kalal | 2023 年 3 月 |信息安全报告
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/6c6e9457-7a41-4591-bcd8-165195797903.png?raw=true)

Keys

I recently wrote an article on how we can bypass client-side encryption. With the help of the PyCript burp suite extension, we can make manual and automated pentesting or bug bounty much easier on applications with client-side encryption. The use of the PyCript extension fails when the application uses asymmetric encryption.  
我最近写了一篇关于如何绕过客户端加密的文章。在 PyCript burp 套件扩展的帮助下，我们可以在具有客户端加密的应用程序上更轻松地进行手动和自动渗透测试或漏洞赏金。当应用程序使用非对称加密时，使用 PyCript 扩展失败。

Since asymmetric encryption uses private and public key mechanisms. It's not possible to decrypt the request without having both keys. The application will store the public on the client side and will use the public key to encrypt the request. To decrypt we need a private key and in most cases, we won’t have the private key since it's stored on the server side.  
由于非对称加密使用私钥和公钥机制。没有两个密钥就无法解密请求。应用程序将在客户端存储公开信息，并使用公钥对请求进行加密。要解密我们需要一个私钥，在大多数情况下，我们不会有私钥，因为它存储在服务器端。

The only options left to test the application with asymmetric encryption are using the browser console with breakpoint and in the case of mobile application, we need to use the Frida to log the plain text.  
使用非对称加密测试应用程序的唯一选项是使用带有断点的浏览器控制台，在移动应用程序的情况下，我们需要使用 Frida 来记录纯文本。

Solution
--------

The only possible solution I was able to figure out was using the Chrome override feature with PyCript configured in Burp Suite. The Chrome browser allows us to edit the JavaScript file and load the JavaScript file from the local system. We can use it to modify our application JavaScript file to send a plain text request instead of an encrypted request.  
我能想到的唯一可能的解决方案是使用 Chrome 覆盖功能和在 Burp Suite 中配置的 PyCript。 Chrome 浏览器允许我们编辑 JavaScript 文件并从本地系统加载 JavaScript 文件。我们可以使用它来修改我们的应用程序 JavaScript 文件以发送纯文本请求而不是加密请求。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/12a73732-4043-4778-8724-7ee79db300a9.jpeg?raw=true)

Browser Flow

The above flow is simple where JavaScript code will send the HTTP request for each action we perform on the browser UI. The same code will call another JavaScript code to encrypt the request and return the encrypted value. The main code will now send the encrypted request and the same will be visible in the burp suite proxy.  
上面的流程很简单，其中 JavaScript 代码将为我们在浏览器 UI 上执行的每个操作发送 HTTP 请求。相同的代码将调用另一个 JavaScript 代码来加密请求并返回加密后的值。主代码现在将发送加密的请求，并且在 burp 套件代理中也是可见的。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/e0b9ceb5-ee6b-4744-94e9-23d1b88e01cd.jpeg?raw=true)

Modified Flow

In the above, we have modified the encryption code using the chrome override feature to return the original plain text instead of the encrypted request. In this case, we can see the decrypted request in the burp suite proxy. Since the server will expect the encrypted request we will configure the PyCript to encrypt the request using the same public key and encryption logic as the application's JavaScript code.  
在上面，我们使用 chrome 覆盖功能修改了加密代码以返回原始明文而不是加密请求。在这种情况下，我们可以在 burp 套件代理中看到解密的请求。由于服务器需要加密请求，我们将配置 PyCript 以使用与应用程序的 JavaScript 代码相同的公钥和加密逻辑来加密请求。

Using the above approach we will have a plain text request in the burp suite proxy history and we can use the same plain text request everywhere like for repeater or intruder. The application on the server side will receive the encrypted request with the help of the PyCript extension.  
使用上述方法，我们将在 burp 套件代理历史记录中有一个纯文本请求，我们可以在任何地方使用相同的纯文本请求，例如转发器或入侵者。服务器端的应用程序将在 PyCript 扩展的帮助下接收加密请求。

Example
-------

I have the below application by intercepting the request in the proxy I can confirm that the application is using some kind of encryption.  
通过拦截代理中的请求，我有以下应用程序，我可以确认该应用程序正在使用某种加密。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/4950eece-6426-468b-a014-9c9eac653974.png?raw=true)

Encrypted Request

After looking into the JavaScript code and searching for keywords like `encrypt, key, encryption,decrypt` etc. I got the encryption code.  
在查看 JavaScript 代码并搜索 `encrypt, key, encryption,decrypt` 等关键字后，我得到了加密代码。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/26a332f6-3de8-4f11-a94a-e1e783a1cf7b.png?raw=true)

Encryption Code

The above code looks quite simple and easy to understand. The function takes one argument and it will be the plain text data. Next, the code converts the PEM public key and encrypts the plain text data using the key. Lastly, the code base64 encodes the encrypted data and returns it. After doing some research I found that the application is using the node-forge library for encryption.  
上面的代码看起来相当简单易懂。该函数接受一个参数，它将是纯文本数据。接下来，代码转换 PEM 公钥并使用密钥加密纯文本数据。最后，代码 base64 对加密数据进行编码并返回。在做了一些研究之后，我发现该应用程序正在使用 node-forge 库进行加密。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/d640144c-d150-4813-98a1-2e86f01669d1.png?raw=true)

Debug

Now just to confirm that the same code is responsible for encryption, I add a breakpoint and submit the request in the browser. The browser stops when the breakpoint code is triggered. I can confirm that the same code is used for encryption and the function is taking plain text value.  
现在只是为了确认相同的代码负责加密，我添加了一个断点并在浏览器中提交了请求。触发断点代码时浏览器停止。我可以确认相同的代码用于加密并且该函数采用纯文本值。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/0e3a3dab-d0af-47dd-80de-d7a03d3d9215.png?raw=true)

Values

I keep the breakpoint the same as it is and call the variables from the console and I got the public key used for encryption. Now we want that the function should return the plain text or the same value instead of the encrypted value.  
我保持断点不变，并从控制台调用变量，我得到了用于加密的公钥。现在我们希望函数应该返回纯文本或相同的值而不是加密值。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/f6740f9b-4718-4081-9009-169704cceb71.png?raw=true)

Override

Now to modify the JavaScript code we need to use the override from the chrome browser. Select the `Overrides`from the source tab. Click on the `Select folder for overrides`.  
现在要修改 JavaScript 代码，我们需要使用 chrome 浏览器中的覆盖。从源选项卡中选择 `Overrides` 。点击 `Select folder for overrides` 。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/f23643f5-6752-43f7-beb7-58cc12e0d942.png?raw=true)

Override

Once you select the folder you need to approve it. Click on the allow and it will allow you to modify the files. Now go to the file which we want to edit.  
选择文件夹后，您需要批准它。单击允许，它将允许您修改文件。现在转到我们要编辑的文件。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/367d15be-e680-4439-81a9-86e014ea3e0c.png?raw=true)

Save override

Right-click on the file and select `Save for overrides`. and now if we back to the override tab we can see the same file is added which allows us to edit the JavaScript code.  
右键单击该文件并选择 `Save for overrides` 。现在，如果我们返回覆盖选项卡，我们可以看到添加了相同的文件，这允许我们编辑 JavaScript 代码。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/8ba9019f-4054-48f1-94c8-4d390db9ad82.png?raw=true)

Edit Encryption Logic编辑加密逻辑

Now we need to edit the JavaScript code. We modify the code and remove the encryption code and return the same plain text value that was passed to the function. Once we complete the editing we can save it with `ctrl+s`.  
现在我们需要编辑 JavaScript 代码。我们修改代码并删除加密代码并返回传递给函数的相同纯文本值。完成编辑后，我们可以使用 `ctrl+s` 保存它。

Once it's completed I need to verify if it's working as expected. I go back to the application and perform any action so any request will be sent. In the burp suite, I need to verify if the request is in plain text or not.  
完成后，我需要验证它是否按预期工作。我返回应用程序并执行任何操作，以便发送任何请求。在 burp 套件中，我需要验证请求是否为纯文本。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/bb8a679b-87b3-4017-a18b-54a57c2f6b9f.png?raw=true)

Plain the request简单的请求

Now I can confirm that I was able to modify the client-side logic and can see the data in plain text format in the burp suite proxy. The application is giving an error as it was expecting the encrypted data and we have sent the plain text data. Now I need to implement the PyCript to auto-encrypt the request before sending it to the server. but before that, we need to use the same encryption logic to write the encryption code for PyCript.  
现在我可以确认我能够修改客户端逻辑并且可以在 burp 套件代理中看到纯文本格式的数据。应用程序出现错误，因为它需要加密数据，而我们已发送纯文本数据。现在我需要实现 PyCript 以在将请求发送到服务器之前自动加密请求。但在此之前，我们需要使用相同的加密逻辑为 PyCript 编写加密代码。

Encryption with PyCript使用 PyCript 加密
------------------------------------

As mentioned the application is using the node-forge library for encryption, I need to install it using the command provided in the node-forge readme file.  
如前所述，应用程序使用 node-forge 库进行加密，我需要使用 node-forge 自述文件中提供的命令安装它。

```
var forge = require('node-forge');  
const program = require("commander");  
const { Buffer } = require('buffer');  
program  
  .option("-d, --data <data>", "Data to process")  
  .parse(process.argv);

  const options = program.opts();  
const requestbody = Buffer.from(options.data, 'base64').toString('utf8');

var mypubkey = "-----BEGIN PUBLIC KEY-----MIIwYIYquwxIqzkgkI+oA9oyrbYQIDAQAB-----END PUBLIC KEY-----"  
var m = forge.pki.publicKeyFromPem(mypubkey)

var encoutput = m.encrypt(requestbody) 

 console.log(forge.util.encode64(encoutput));


```

In the above code, I am using the same format required for PyCript and with node-forge, I am encrypting the data and lastly printing the encrypted data after encoding it with base64. The encryption method and logic is the same as the application was using with some modification based on PyCript requirements.  
在上面的代码中，我使用 PyCript 和 node-forge 所需的相同格式，我正在加密数据，最后在使用 base64 编码后打印加密数据。加密方法和逻辑与应用程序使用的相同，只是根据 PyCript 要求进行了一些修改。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/34398f91-fee6-484d-baf9-92c47ed6d35b.png?raw=true)

PyCript

Since in my case the request is already in plain text format and I just need to encrypt it and doesn't require decryption. I don’t need to write the decryption code but since the extension requires both codes to work, I can provide the same code for both encryption and decryption just to ignore the restrictions. Once I load the script I can select the request type in my case only values are encrypted in JSON I will select the appropriate type.  
因为在我的例子中，请求已经是纯文本格式，我只需要对其进行加密而不需要解密。我不需要编写解密代码，但由于扩展需要两种代码才能工作，我可以为加密和解密提供相同的代码来忽略这些限制。加载脚本后，我可以选择请求类型，在我的情况下，只有值在 JSON 中加密，我将选择适当的类型。

Lastly, I need to select the tool type, since all requests will come from the browser I select the proxy and turn on the auto encrypt.  
最后，我需要选择工具类型，因为所有请求都来自浏览器，我选择代理并打开自动加密。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/f756ce7f-89eb-4c95-b2b0-ce7cc3188f82.png?raw=true)

Plain text

I go back to the browser and perform any action and verify that again the browser will send the plain text request. But now I am not getting any error as an invalid request and the application is giving valid responses even though my request is in plain text format.  
我返回浏览器并执行任何操作并再次验证浏览器是否会发送纯文本请求。但是现在我没有收到任何无效请求的错误，即使我的请求是纯文本格式，应用程序也给出了有效的响应。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/4/5f249b10-1204-4190-a70b-8ad5adea7b9a.png?raw=true)

Encrypted Request

If I go to the logger in the burp suite I can see that request is encrypted even though I can see the plain text request in the proxy. This approach will keep the plain text request in the burp proxy so I can use it later and will not require any private key to decrypt it.  
如果我转到 burp 套件中的记录器，我可以看到该请求已加密，即使我可以在代理中看到纯文本请求。这种方法会将纯文本请求保留在 burp 代理中，以便我以后可以使用它，并且不需要任何私钥来解密它。

The approach is not the best but still works and allows us to ignore symmetric encryption. A similar approach can use used in the mobile application as well. For the mobile application, we need to return the plain text value instead of the encrypted value same as we did for the web. But in the case of mobile we either need to use Frida or we can modify the mobile application code  
该方法不是最好的但仍然有效并且允许我们忽略对称加密。类似的方法也可以用在移动应用程序中。对于移动应用程序，我们需要返回纯文本值，而不是像我们为 Web 所做的那样返回加密值。但是在移动端我们要么使用Frida，要么修改移动端应用代码

All the tools can be found below.  
所有工具都可以在下面找到。

Twitter — [https://twitter.com/ano\_f\_](https://twitter.com/ano_f_)  
推特—— [https://twitter.com/ano\_f\_](https://twitter.com/ano_f_)

GitHub — [https://github.com/Anof-cyber/](https://github.com/Anof-cyber/)  
GitHub—— [https://github.com/Anof-cyber/](https://github.com/Anof-cyber/)

Linkedin — [https://www.linkedin.com/in/sourav-kalal/](https://www.linkedin.com/in/sourav-kalal/)  
领英—— [https://www.linkedin.com/in/sourav-kalal/](https://www.linkedin.com/in/sourav-kalal/)