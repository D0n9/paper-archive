# 调教某数字杀软，权限维持so easy
大家好！这里是219攻防实验室！

219攻防实验室专注于前沿攻防研究、武器平台化、红队评估、攻防演练，虽然我们是新实验室，但我们的成员有深入操作系统多年的老牌程序员、有专注攻防领域10余年的老手、亦有攻防演练中崭露头角的新锐，我们热爱技术热爱分享，就像我们的名字一样，219 “爱依旧”，对技术的爱永远依旧。

0x00 背景
-------

    权限维持（Persistence），顾名思义，在攻击者获得权限后需要长期维持，早期我们通常将其称为开机启动，如今MITRE建立的ATTACK标准已将其定义为战术TA0003，https://attack.mitre.org/tactics/TA0003/。

    虽说权限维持不是攻击路径中的必备一环，但它却是持久性攻击的必经之路，因此它必然会面临两个挑战：有效性和隐蔽性。

*   有效性是指权限维持是否长期有效、能否稳定触发，以及触发时机和触发方式都是需要根据实际攻击环境而定。
    
*   隐蔽性是指维持点和建立维持的过程是否会被各大AV/EDR检测到，而检测又分为事前检测（行为拦截）和事后检测（用户主动扫描）。
    

    TA003涵盖了所有公开的权限维持点，因为这些都来源于实战攻击活动中，从有效性这个范畴来说，这些点基本上不存在问题，因此隐蔽性便成了攻防对抗中老生常谈的话题。本文打算从具体示例出发，通过COM劫持聊一聊某杀软对权限维持的检测以及绕过。

    注：笔者分析的是最新版本，文章公布后可能会修复BUG，读者可自行测试研究。

0x01 COM劫持
----------

    COM是Component Object Model（组件对象模型）的简称，由Windows3.11引入的，目的在于代码复用、进程间通信，广泛用于ActiveX，COM+、DCOM等框架。以前很多人会搞混COM和RPC，简单的说前者是组件标准，后者是远程调用方式。COM可以基于RPC来实现远程调用，也可以通过窗口消息实现。

  COM信息存在注册表HKLM、HKCU\\Software\\Classes\\CLSID中，并且HKCU加载优先于HKLM，常见键值如下：

```


`

{CLSID}  
--InprocServer32  REG_SZ  模块路径  
----ThreadingModel  REG_SZ  线程模型  


`


```

    由于HKLM下的CLSID只能TrustInstaller权限进行读写，因此通常方法是在HKCU下新建相同的键值，以达到劫持目的。

   有人可能会说为什么不SetObjectSecurity更改注册表权限、或者DuplicateToken TrustInstaller，这一是为了简单，二是尽量不影响原系统，遵从安全最小原则，尽量少生产行为日志。更多有关COM劫持的技术，可参考文章《Persistence – COM Hijacking》。

    同样地，杀软主要也是通过注册表做检测，因此接下来聊聊杀软对COM劫持的主动扫描。

0x02 主动扫描及BUG分析
---------------

    CacheTask是Wininet缓存的计划任务，在用户登录时执行，CSLID为{0358b920-0ac7-461f-98f4-58e32cd89148}，劫持它并不会对系统造成什么不良影响，并且杀软对其有保护，因此我们选择将它作为分析对象。

    首先我们在注册表里新建如下键值：

    然后在杀软中执行扫描自启动项，提示存在风险项：

    下面调试下这个扫描过程，由于有进程保护，因此先在内核中将ProcessObject回调移除，经过分析后发现是deepscan.dll模块负责扫描，进而通过IDA定位到扫描的代码，如图所示：

    同时通过Windbg条件断点dump出所有扫描的位置：

```


`

Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders  
Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders  
.....  
SOFTWARE\Classes\exefile\shell  
SOFTWARE\Classes\batfile\shell  
.....  
System\CurrentControlSet\Services\WinSock2\Parameters\Protocol_Catalog9\Catalog_Entries\000000000001  
.....  
Software\Classes\*\ShellEx\ContextMenuHandlers\Open With  
.....  


`


```

    根据这些信息可以推断出权限维持的检测点，同时为攻击者提供大量的数据佐证。继续跟进上面的CLSID扫描函数，review代码发现其对HKCU和HKLM都有检查，没有遗漏。

    再梳理一下检测逻辑：扫描注册表 => 获取路径 =\> 文件检测。从攻击视角来看，这里可能会存在两个风险点：1、**扫描注册表** 2、**文件检测**，文件检测不属于本文所探讨的范围，因此我们将目光集中在注册表扫描。

    作为检测类软件，首先应该确保自身数据源的准确性，而注册表数据可能存在伪造（API Hook、注册表回调等），因此在这个点需要考虑。我们分析发现该杀软对注册表等操作进行了封装，实现了一套BAPI，具体操作由BAPIDRV.sys驱动实现，从底层保证了数据源的准确性。

    因此接下来考虑数据向上传递的链路是否存在问题，尤其是在错误处理上。获取数据后返回的代码如下图所示：

    向上回溯，最终来到sub_713EF函数，发现代码并没有校验函数返回值，而是直接获取结果（这样必然不严谨），再跟踪参数发现，读注册表的大小固定为0x1000，代码如图所示：

    该函数除了读取固定大小，也没有对返回值做检查，这就产生了BUG。如果构造一个场景：当注册表里的路径超过了0x1000，读取就会失败，那么路径就是一个空字符串。

    路径超过0x1000当然不太现实（\\?\\UNC长路径除外），但可以考虑填充0x00。（也可以填充0x20，COM组件加载解析时会自动trim掉前后空格）。

```


`

// 在路径后填充0x1000个0x00：  

c:\cachetask-hijack.dll + '\x00' * 0x1000

`


```

    填充后，再次扫描，注册表数据返回空，自然也不会告警，结果如图：

    最终利用这个BUG逃避了该杀软的检测，实现了权限维持的隐蔽性。

0x03 行为拦截及绕过
------------

    上一小节探讨了事后检测，即通过主动扫描去发现风险，并利用BUG绕过检测。接下来分析一下事前检测，行为拦截，也就是常说的主动防御。

    关于注册表的行为拦截，标准做法是内核中的CmCallback回调，作为一个成熟的杀软，在设计上必然会保证策略和机制分离，机制的实现出现逻辑问题的概率较低，因此策略会是对抗中更关注的点。

    当我们通过regedit.exe写入CacheTask这个COM注册表时，确实杀软告警了，并且追踪到了我们的测试程序。

   我们先别盲目测试，先形式化分析一下这个检测过程，我们将操作者定义为A（regedit.exe），将客体对象定义为B（CacheTask的COM注册表路径），将操作内容定义为C（cachetask-hijack.dll），从而就有 A (C) => B。

    接下来分析每个变量的安全性，B的安全性取决于是否全面，上节已经提到，因为COM劫持的位置固定且没有遗漏，所以不再分析。某些情况下，当A写入C时，杀软会根据路径去扫描C文件，如果C文件不受信任也会提示风险，因此可以考虑先写入C路径，后释放C文件来绕过检测。下面着重探讨一下A的安全性。

    A是行为的直接发起者，A是否可信，除了对A本身的静态特征外，还有一个很关键的点就是A的来源，从上图可以看到，风险程序就是推断出的初始发起者，这就要引出下一个话题：进程信任链。

### 1.进程信任链

    通常是否可信可通过PKI的签名校验来判定，而建立完整的进程链的却是一个复杂的过程，一般杀软会在CreateProcessNotify回调中建立父子关系，然而进程启动有很多方式，诸如RPC、模拟点击、漏洞等等，Windows并没有提供标准的接口来监控这些点，因此通常的做法是要么放弃、要么看ETW能否监控，或者在调用点Hook。

    针对攻击者而言，需要寻找一种杀软不容易监控到的方式，让其不能追溯到原始进程，将进程链断掉。RPC就是比较常见的断链方式，例如SecLogon、LSA等服务会暴露许多RPC接口，这些接口或许能执行程序，此外Explorer也是经常关注的对象，例如IWebBrowser COM接口，然而测试发现这些点都能被追溯到，看来都被照顾了。

    当然除了构造进程链，还有规则检查，经过简单测试，发现进程链的检查规则如下：

*   **告警×**：explorer => regedit.exe
    
*   **通过√**：explorer => sideload.exe(合法签名) => regedit.exe
    
*   **告警×**：explorer => evil.exe(无签名) => sideload.exe(合法签名) => regedit.exe
    

    因此要绕过杀软的进程链检查，首先需要一个白程序，这个不难找到，其次得保证整个链都是合法签名。基于此，笔者挖掘了一种新的断链启动方式。

### 2.断链挖掘

众所周知，Windows给快捷方式提供了一个快捷键，如下图所示：

    这里有个前提条件是：要实现快捷键启动，快捷方式必须位于特定目录或其子目录，比如：

*   桌面：%USERPROFILE%\\Desktop, %PUBLIC%\\Desktop
    
*   开始菜单：Microsoft\\Windows\\Start Menu
    
*   快速启动栏：Quick Launch
    

    快捷键启动是由explorer完成的，因此通过这种方式就能实现断链，让杀软无法关联到原始发起程序，只能找到explorer，这样便可绕过检查。

    关于启动方式（这里不考虑Session隔离）：虽说可以通过模拟按键触发，但不够优雅和稳定，还会对系统造成额外影响，因此我对这种启动原理做了一些研究，发现了一种更为简单高效的启动方式。

  通过对explorer的ntdll!NtCreateUserProcess下断点发现，是由HotkeySearchFailed函数调用ExecItemByPidls来执行快捷方式，栈回溯如下：

```


`

SHELL32!HDXA_LetHandlerProcessCommandEx+0x10c  
SHELL32!CDefFolderMenu::InvokeCommand+0x13d  
shlwapi!SHInvokeCommandOnContextMenu2+0x1f2  
shlwapi!SHInvokeCommandWithFlagsAndSite+0xb4  
shlwapi!SHInvokeDefaultCommand+0x21  
Explorer!_ExecItemByPidls+0x85  
Explorer!CTray::_HotkeySearchFailed+0xd5  
Explorer!CTray::v_WndProc+0xb96  
Explorer!CImpWndProc::s_WndProc+0x78  
user32!UserCallWinProcCheckWow+0x2f8  
user32!CallWindowProcW+0x8e  


`


```

   在IDA找到s_WndProc调用HotkeySearchFailed的代码，如下：

    由上可知，调用是由WM_TIMER触发，根据Event ID，不难找到通过SetTimer来设置定时器的代码，如下图所示：

   分析代码发现CTray::\_HandleHotKey函数由WndProc调用，对其下断点，得到HWND、MessageID和wparam参数，其中wparam是一个ID，每次设置新的快捷键后都会自增，DSA\_GetItemPtr函数根据ID可得到具体PIDL，关于PIDL的介绍参考MSDN《Common Explorer Concepts》，最后设置Timer，调用ExecItemByPidls完成启动。

    最终整个过程可简化成直接给Shell_TrayWnd发消息完成启动，代码如下：

```


`

HWND tray_wnd = FindWindowA("Shell_TrayWnd", "");  
UINT msgid = 0x312;  
WPARAM wparam = 0;  //默认从0自增  
LPARAM lpram = NULL;  
PostMessage(tray_wnd, msgid, wparam, lpram);  


`


```

    此外创建快捷方式、设置快捷键都可以通过ShellLink COM接口完成，代码如下：

```


`

IShellLink* psl;  
HRESULT hr = CoCreateInstance(CLSID_ShellLink, NULL, CLSCTX_INPROC_SERVER, IID_PPV_ARGS(&psl));  
if (SUCCEEDED(hr)) {  
    psl->SetPath(TARGET);   //设置成sideload.exe路径  
    psl->SetDescription(TEXT("Test"));  
    psl->SetHotkey(MAKEWORD('E', HOTKEYF_CONTROL | HOTKEYF_ALT));   //可设置成任意值，调用是根据ID来的，无需考虑快捷键  
    psl->SetShowCmd(SW_NORMAL);   //设置启动方式，美中不足的是只能设置最大化、最小化和Normal，不能设置SW_HIDE  
    IPersistFile* ppf;  
    hr = psl->QueryInterface(&ppf);  
    if (SUCCEEDED(hr)) {  
        hr = ppf->Save(LNKFILE, TRUE);  //生成的快捷方式路径  
        ppf->Release();  
    }  
    psl->Release();  
}  


`


```

0x04 完整流程
---------

最终梳理的整个权限维持流程如下图所示：

```


`

// 进程链  
evil.exe => 断链  
explorer.exe => sideload.exe => regedit.exe

`


```

    为了方便阐述进程链关系，选择regedit.exe来操作注册表，当然也可以换成自己调用API。

0x05 总结
-------

    本文讲述了一次绕过某杀软来进行权限维持的过程：从调试杀软的BUG到挖掘进程断链的方法，同时分析了杀软的事前和事后检测。权限维持也还有很多方式，选择COM劫持是为了更好体现检测过程。文章更多体现的是笔者的一种分析思路，希望能抛砖引玉给读者带来一些启发。

    最后要说明的是，在不断完善的现代安全体系中，在面对全方位的防御检测时，攻击方式也得顺势改变，不断创新。同时攻击方如果要想走得更远，也需充分学习防御方的检测思路，加强软件工程化以及武器化的建设，不是吗。