# 关于绕过eBPF安全监控 · Doyensec的博客
2022 年 10 月 11 日 - 由 Lorenzo Stella 发布

当今有许多可用的安全解决方案依赖于Linux 内核的[扩展伯克利数据包过滤器 (eBPF)功能来监控内核功能。](https://ebpf.io/)多种原因推动了最新监控技术的范式转变。其中一些是出于在日益以云为主导的世界中对性能的需求，[等等](https://www.youtube.com/watch?v=44nV6Mj11uw)。Linux 内核始终具有内核跟踪功能，例如[kprobes](https://docs.kernel.org/trace/kprobes.html) (2.6.9)、[ftrace](https://www.kernel.org/doc/Documentation/trace/ftrace.txt)（2.6.27 及更高版本）、[perf](https://perf.wiki.kernel.org/index.php/Main_Page) (2.6.31) 或[uprobes](https://docs.kernel.org/trace/uprobetracer.html)(3.5)，但有了 BPF，最终可以在事件上运行内核级程序，从而修改系统状态，而无需编写内核模块。这对任何想要破坏系统并不被发现的攻击者都具有重大意义，从而开辟了新的研究和应用领域。如今，基于 eBFP 的程序用于[DDoS 缓解](https://blog.cloudflare.com/how-to-drop-10-million-packets/)、[入侵检测](https://dl.acm.org/doi/abs/10.1016/j.jnca.2021.103283)、[容器安全](https://developers.redhat.com/articles/2021/12/16/secure-your-kubernetes-deployments-ebpf)和一般可观察性。

2021 年，[Teleport引入了一项名为](https://goteleport.com/)[增强型会话记录](https://goteleport.com/blog/enhanced-session-recording/)的新功能，以弥补 Teleport 审计能力中的一些监控差距。[所有报告的问题都已按照其2021 年第四季度公开报告](https://goteleport.com/resources/audits/teleport-features-security-audit-q4-2021/)中的描述得到及时修复、缓解或记录。您可以在下面看到我们如何设法绕过基于 eBPF 的控制的图示，以及关于红队或恶意行为者如何规避这些新的入侵检测机制的一些想法。这些技术通常可以应用于其他目标，同时试图绕过任何基于 eBPF 的安全监控解决方案：

*   [关于 eBPF 工作原理的几句话](#a-few-words-on-how-ebpf-works)
*   [常见的缺点和潜在的绕过（这里是龙）](#common-shortcomings--potential-bypasses-here-be-dragons)
    *   [1.了解捕获了哪些事件](#1-understand-which-events-are-caught)
        *   [1.1 执行绕过](#11-execution-bypasses)
        *   [1.2 网络绕过](#12-network-bypasses)
    *   [2.延迟执行](#2-delayed-execution)
    *   [3\. 规避基于范围的事件监控`cgroup`](#3-evade-scoped-event-monitoring-based-on-cgroup)
    *   [4.内存限制和事件丢失](#4-memory-limits-and-loss-of-events)
    *   [5.永远不要相信用户空间](#5-never-trust-the-userspace)
    *   [6.滥用`seccomp-bpf`内核差异](#6-abuse-the-lack-of-seccomp-bpf--kernel-discrepancies)
    *   [7.干扰代理](#7-interfere-with-the-agents)

关于 eBPF 工作原理的几句话
----------------

  
![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/2065fb83-9186-4fd1-a08b-b22a3c2e5f86.png?raw=true)

扩展的 BPF 程序是用高级语言编写的，并使用工具链编译成 eBPF 字节码。用户态应用程序使用`bpf()`系统调用，其中 eBPF 验证器将执行大量检查以确保程序“安全”地在内核中运行。这个验证步骤很关键——eBPF 为非特权用户暴露了一条在 ring 0 中执行的路径。由于允许非特权用户在内核中运行代码是一个成熟的攻击面，过去的几项研究都集中在本地特权利用 (LPE) 上，我们不会在这篇博文中介绍。程序加载后，用户态应用程序将程序附加到一个挂钩点，当某个挂钩点（事件）被命中（发生）时，该挂钩点将触发执行。在某些情况下，程序也可以被 JIT 编译成本地汇编指令。用户模式应用程序可以使用 eBPF 映射和 eBPF 辅助函数与在内核中运行的 eBPF 程序交互并从中获取数据。

常见的缺点和潜在的绕过（这里是龙）
-----------------

### 1.了解捕获了哪些事件

[虽然 eBPF 速度很快（比auditd](https://linux.die.net/man/8/auditd)快得多），但由于性能原因，有很多有趣的领域无法使用 BPF 进行合理检测。根据安全监控解决方案最想保护的内容（例如，网络通信与执行与文件系统操作），可能存在过度探测可能导致性能开销的区域，从而促使开发团队忽略它们。这取决于端点代理的设计和实现方式，因此仔细审核 eBPF 程序的代码安全性至关重要。

#### 1.1 执行绕过

例如，一个简单的监控解决方案可以决定只挂钩`execve`系统调用。与流行的看法相反，多个基于 ELF 的类 Unix 内核不需要磁盘上的文件来加载和运行代码，即使它们通常需要一个文件。实现这一目标的一种方法是使用一种称为反射加载的技术。反射加载是一种重要的后开发技术，通常用于避免检测并在锁定环境中执行更复杂的工具。说明的手册页 `execve()`：“_`execve()`执行文件名指向的程序……_ ”，然后继续说“_调用进程的文本、数据、bss 和堆栈被加载的程序覆盖_”。与文件系统访问或任何其他事情不同，这种覆盖不一定构成 Linux 内核必须垄断的东西。因此，`execve()`可以在用户空间中以最小的难度模仿系统调用。因此，创建一个新的过程映像是一件简单的事情：

*   清理地址空间；
*   检查并加载动态链接器；
*   加载二进制文件；
*   初始化堆栈；
*   确定入口点和
*   转移执行控制权。

通过执行这六个步骤，可以创建并运行一个新的过程映像。由于该技术[最初于 2004 年被报道](https://grugq.github.io/docs/ul_exec.txt)，该过程如今已被 OTS 后期开发工具开创并简化。正如预期的那样，eBPF 程序挂钩`execve`将无法捕捉到这一点，因为这个自定义用户空间`exec`将有效地用新的进程映像替换当前地址空间中的现有进程映像。在此，userland exec 模仿系统调用的行为`execve()`。但是，因为它在用户空间中运行，所以描述进程映像的内核进程结构保持不变。

其他系统调用可能会不受监控并降低监控解决方案的检测能力。其中一些是`clone`, `fork`, `vfork`,`creat`或`execveat`。

如果 BPF 程序是天真的并且信任引用`execve`正在执行的文件的完整路径的系统调用参数，则可能存在另一个潜在的绕过。攻击者可以在不同位置创建 Unix 二进制文件的符号链接并执行它们——从而篡改日志。

#### 1.2 网络绕过

不挂钩所有与网络相关的系统调用可能会产生一系列问题。一些监控解决方案可能只想挂钩 EGRESS 流量，而攻击者仍然可以将数据发送到未经允许的主机，滥用与 INGRESS 流量相关的其他网络敏感操作（参见 ）[：](https://code.woboq.org/linux/linux/security/apparmor/include/audit.h.html#78) `aa_ops``linux/security/apparmor/include/audit.h:78`

*   `OP_BIND`，该`bind()`函数应将本地套接字地址分配给由描述符套接字标识的套接字，该套接字没有分配本地套接字地址。
*   `OP_LISTEN`，该`listen()`函数应将由套接字参数指定的连接模式套接字标记为接受连接。
*   `OP_ACCEPT`，该`accept()`函数应提取挂起连接队列中的第一个连接，创建一个与指定套接字具有相同套接字类型协议和地址族的新套接字，并为该套接字分配一个新的文件描述符。
*   `OP_RECVMSG`，该`recvmsg()`函数应从连接模式或无连接模式套接字接收消息。
*   `OP_SETSOCKOPT`，该`setsockopt()`函数应在 level 参数指定的协议级别将 option\_name 参数指定的选项设置为与 socket 参数指定的文件描述符关联的套接字的 option\_value 参数指向的值。攻击者感兴趣的选项是`SO_BROADCAST`, `SO_REUSEADDR`, `SO_DONTROUTE`。

_通常，网络监控应该像AppArmor_一样查看所有基于套接字的操作。

如果同一本地用户混合了受监控和未受监控的控制台会话，则受监控会话中的攻击者可能会利用打开的文件描述符和套接字将数据发送到受限主机。在 2020 年，某些版本的 Linux 内核引入了一个[新的系统调用](https://man7.org/linux/man-pages/man2/pidfd_getfd.2.html)来实现这一点`pidfd_getfd`。少数操作系统（如 Ubuntu）实现了[Yama](https://www.kernel.org/doc/Documentation/security/Yama.txt)内核模块，该模块将文件描述符访问限制为仅子父进程。[Github ( TheZ3ro/fdstealer](https://github.com/TheZ3ro/fdstealer) )上提供了使用此功能的 PoC 代码。

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/bccdc818-0a1b-420e-b708-f1e714ddf9e4.png?raw=true)

### 2.延迟执行

如果仅监视活动的控制台会话，则 eBPF 程序可能仅在会话的时间范围内存在。通过延迟命令的执行（通过`sleep`, `timeout`, `sar`, `vmstat`, `inotifywait`, `at`, `cron`…）并退出会话，可以完全避开解决方案。

### 3\. 规避基于范围的事件监控`cgroup`

安全监控解决方案可能只对审核特定用户或 cgroup 感兴趣（例如在远程控制台会话的上下文中）。以 Teleport 为例，它通过将每个事件关联到具有控制组的会话来实现这一点（`cgroupv2`尤其）。控制分组是一种 Linux 内核功能，用于将对资源的访问限制为一组进程。它用于许多容器化技术（Docker 在幕后为容器创建一组名称空间和控制组），其特点是所有子进程都将保留父进程的 ID。当 Teleport 启动 SSH 会话时，它首先会重新启动自己并将自己置于 cgroup 中。这不仅允许使用唯一 ID 跟踪该进程，还允许 Teleport 启动的所有未来进程。Teleport 运行的 BPF 程序已更新为还发出执行它们的程序的 cgroup ID。BPF 脚本检查返回的值`bpf_get_current_cgroup_id()`并且只关心重要的会话cgroup。最简单的规避此审计策略的方法是更改​​您的 cgroup ID，但攻击者需要是 root 才能实现此目的。干预 cgroupv2 伪文件系统或滥用 PAM 配置也是影响 cgroup/session 相关性的潜在机会。

另一种技术涉及由 init 重新分配。在 Teleport 的情况下，当`bash`会话产生的进程死亡时，其子进程成为孤儿进程，Teleport 进程终止其执行。当一个子进程成为孤儿时，操作系统可以在特定条件下（没有 tty、成为进程组领导、加入新进程会话）将其分配到不同的 cgroup。这允许攻击者绕过现有的限制。以下 PoC 是此设计的旁路示例：

1.  打开一个新的 eBPF 监控会话
2.  执行命令启动[tmux](https://github.com/tmux/tmux/wiki)`tmux`
3.  `tmux`按下`CTRL+B`然后分离`D`
4.  `tmux`杀死作为父进程的 bash 进程
5.  `tmux`通过执行重新附加到进程`tmux attach`。进程树现在看起来像这样：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/3/57902a0b-8cfd-4d5e-a550-d076b622169f.png?raw=true)

作为另一种攻击途径，利用不同本地用户/`cgroupv2`机器上运行的进程（滥用其他守护进程，委托 systemd）也可以帮助攻击者规避这种情况。这方面显然取决于托管监控解决方案的系统。防止这种情况很棘手，因为即使`PR_SET_CHILD_SUBREAPER`被设置为确保后代不能将自己重新作为 init 的父级，如果祖先收割者死亡或被杀死（DoS），那么该服务中的进程可以逃脱他们的 cgroup“容器” . 这个特权服务进程的任何妥协（或它的渎职行为）都会让它杀死它的层次结构管理器进程并逃脱所有控制。

### 4.内存限制和事件丢失

BPF 程序有很多限制。只为 eBPF 程序保留了 512 字节的堆栈空间。变量将在执行开始时被提升和实例化，如果脚本试图转储系统调用参数或`pt-regs`，它会很快耗尽堆栈空间。如果没有设置指令限制的解决方法，则可能会推动脚本检索太大而无法放入堆栈的内容，当执行变得复杂时很快就会失去可见性。但即使使用变通方法（例如，当使用多个探测器跟踪相同的事件但捕获不同的数据，或者将您的代码拆分为多个使用程序映射相互调用的程序时），仍有可能滥用它。BPF 程序并不意味着永远运行，但它们必须在某个时刻停止。例如，如果监控解决方案在 CentOS 7 上运行并尝试捕获进程参数及其环境变量，则发出的事件可能包含太多 argv 和太多 envp。即使在那种情况下，你可能会错过其中的一些，因为循环提前停止了。在这些情况下，事件数据将被截断。重要的是要注意，这些限制因运行 BPF 的内核以及端点代理的编写方式而异。

eBPF 的另一个特点是，如果不能足够快地使用它们，它们将丢弃事件，而不是拖累整个系统的性能。攻击者可以通过生成足够数量的事件来填充 perf ringbuffer 并在代理可以读取数据之前覆盖数据来滥用它。

### 5.永远不要相信用户空间

内核空间对 a 的理解`pid`与用户空间对 a 的理解不同`pid`。如果 eBPF 脚本试图识别文件，正确的方法是获取 inode 编号和设备编号，而文件描述符则没有那么有用。即使在那种情况下，探测器也可能会遇到 TOCTOU 问题，因为它们会将数据发送到可以轻松更改的用户模式。如果脚本改为直接跟踪系统调用（使用`tracepoint`或`kprobe`），它可能会被文件描述符卡住，并且可以通过使用当前工作目录和文件描述符来混淆执行（例如，通过组合 、`fchdir`和`openat`）`execveat`。

### 6.滥用`seccomp-bpf`内核差异

基于 eBPF 的监控解决方案应该通过使用[seccomp-BPF](https://www.kernel.org/doc/html/v4.19/userspace-api/seccomp_filter.html)`bpf()`永久放弃在生成控制台会话之前进行系统调用的能力来保护自己。否则，攻击者将能够进行 bpf() 系统调用以卸载用于跟踪执行的 eBPF 程序。Seccomp-BPF 使用 BPF 程序来过滤任意系统调用及其参数（仅常量，无指针解引用）。

使用内核时要记住的另一件事是，不能保证接口的一致性和稳定性。如果 eBPF 程序未在经过验证的内核版本上运行，则攻击者可能会滥用它们。通常，不同体系结构的条件编译对于这些程序来说非常复杂，您可能会发现您的特定内核的变体没有正确定位。[使用 seccomp-BPF 的一个常见缺陷是在不检查BPF 程序参数的情况](https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt)下过滤系统调用号 `seccomp_data->arch` [](https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt). 这是因为在任何支持多个系统调用调用约定的体系结构上，系统调用编号可能会根据特定调用而有所不同。如果不同呼叫约定中的号码重叠，则过滤器中的检查可能会被滥用。`bpf()`因此，重要的是要确保seccomp-BPF 过滤器规则考虑到每个新支持的体系结构的调用差异。

### 7.干扰代理

与 (6) 类似，可能会以不同的方式干扰 eBPF 程序加载，例如针对 eBPF 编译器库（[BCC](https://github.com/iovisor/bcc)的`libbcc.so`）或采用其他共享库预加载方法来篡改合法二进制文件的行为解决方案，最终执行有害操作。如果攻击者成功改变了解决方案的主机环境，他们可以在前面添加`LD_LIBRARY_PATH`一个目录，他们在其中保存了具有相同的恶意库`libbcc.so`命名并导出所有使用的符号（以避免运行时链接错误）。当解决方案启动时，它链接的不是合法的密件抄送库，而是恶意库。对此的防御可能包括使用静态链接程序、将库链接到完整路径或将程序运行到受控环境中。

非常感谢整个[Teleport 安全团队](https://goteleport.com/)@ [FridayOrtiz](https://tmpout.sh/2/4.html)、[@Th3Zer0](https://twitter.com/th3zer0)和[@alessandrogario](https://mobile.twitter.com/alessandrogario)在撰写这篇博文时提供的灵感和反馈。