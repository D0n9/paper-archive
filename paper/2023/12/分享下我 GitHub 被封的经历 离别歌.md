# 分享下我 GitHub 被封的经历 | 离别歌
最近好像又有人 GitHub 被封，每隔一段时间就有。分享下我自己的经历吧，好几年以前了，也许还是有点参考价值。

[账号被封，查找原因](#_1)
----------------

那是 2017 年 12 月，有天早上起来突然发现自己的号[phith0n](https://github.com/phith0n)登不上去了，具体的表现是：

*   账号登录不上，登录以后明确告诉我我好被封了
*   GitHub 个人页面访问显示 404
*   我自己名下所有项目，访问都是 404
*   但是我创建的 Group 还是好的，没有受影响

我当时也很不明所以，所以发了个[微博](https://weibo.com/1074745063/FAKtz6eVW)吐槽，后来有人在评论区告诉我他收到了 DMCA 的邮件。是因为 fork 了一个项目，这个项目是一个破解软件，安全圈的不少人都因为 fork 这个项目收到了邮件或者被封了。

我想起来我不久前也 fork 了这个项目。而且我还想起来我不是初犯了，我曾经还 fork 过另一个违反 DMCA 的项目，是某个大公司泄露的代码，当时第一时间我 fork 了，后来收到 DMCA 的邮件我没当回事：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/ce3ebd4e-24c3-4586-b254-224d2a4ce3d6.png?raw=true)
](https://www.leavesongs.com/media/attachment/2022/03/01/bf8c0589-b94a-4ff6-8f55-f13694fca2e4.png)

也就是说这次这个破解版的事件是我第二次违反 DMCA ，这确实是我的错误。我一直把 fork 项目当做是“保存快照”的步骤，所以我遇到一些我感觉可能会被删的项目我反而会把他 fork 下来保存一份。

我猜测这就是我账号被封的直接原因。

[统计损失](#_2)
-----------

当时我的号还不像现在有这么多 followers 和 stars ，这是当时我的 profile 的截图：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/e5016e9c-6f50-48ff-9b45-27e367b0f3d8.png?raw=true)
](https://www.leavesongs.com/media/attachment/2022/03/01/9663cb37-1da2-46c1-a76e-db6eed8b8aa2.png)

因此，当时账号被封对我最大的损失主要是这几个：

*   最心疼的是自己点过的一千多 star 。我是把 star 当收藏夹用的，现在等于收藏夹丢了。
*   丢了代码仓库，丢了 followers 。这其实还好，因为代码我本地都有，followers 也可以慢慢再挣。
*   有一些使用 GitHub 做第三方登录的网站登不上了

知道了事情大概的原因后，我要做的主要就是两件事，第一件事是想办法挽回上面说到的三个损失；第二件事是联系官方，看事情能不能补救。

[挽回损失](#_3)
-----------

我并没有抱着能解封的期望，所以我需要先挽回损失。我统计了自己代码没丢以后，那么主要就是找回自己点的那些 star 了。

我用谷歌搜了下自己的 GitHub ID ，的确找到了一些第三方网站的备份，但要不就是信息太旧不全，要不就没有 star 的列表，只能说挽回部分损失。

不过我很快发现，GitHub API 仍然是可以访问的。就是我们可以访问如下 API 来找到某个用户 star 过的仓库：

`https://api.github.com/users/[username]/starred` 

比如这两天被封的那位仁兄[sam01101](https://api.github.com/users/sam01101/starred)，可以找到他的 star 。

所以我很快备份了自己的star，心态迅速平复。

后来 V 站另一个仁兄[荒野无灯](https://www.v2ex.com/t/471437)也遇到了类似的问题，也是用我这个方法备份了 star 。

[邮件申述](#_4)
-----------

剩下的就是碰运气了，账号被封确实是自己的问题，但是我的问题有一个可以辩解的理由，就是我不是自己主动违反了 DMCA ，而是 fork 别人项目导致的。所以我想用这个作为一个突破口。

我发邮件过去询问我被封号的原因，被告知的确是因为多次违反 DMCA：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/59ea027e-919d-4d2a-bee6-6e9bbf159799.png?raw=true)
](https://www.leavesongs.com/media/attachment/2022/03/01/a5f8ed49-5d75-41c6-901d-9bbf1f5f3bff.png)

并且对方回复了两次，分别说了这两句话：

> Unfortunately, this means we'll have to keep your account suspended.
> 
> We're sorry for any disappointment, but we will not be restoring access to your account.

基本就宣判不能恢复了，不过我最后还是试了下，写了一大段邮件，大意是：

1.  GitHub 对我很重要，我对开源做出过很多贡献，我想继续参与开源项目
2.  我认识到了自己的错误，以后 fork 项目会非常谨慎
3.  我自己的项目没有违反 DMCA ，而且还有其他人参与了这些开源项目，直接封掉我和这些项目，对其他 contributors 不公平
4.  强硬地说虚拟资产也是资产，需要得到保护

不知道是哪一点打动了对方，这封邮件以后，Github 终于给我恢复了：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/9b01bc96-71db-433b-a1ca-7f1294fb171a.png?raw=true)
](https://www.leavesongs.com/media/attachment/2022/03/01/3ff4f82e-a999-4d19-bdd0-a0e3036b149f.png)

这整个申述的过程持续了一个多月，原因也和当时是 12 月有关，外国人都过圣诞节了，所以耽误了很长时间。

[复盘](#_5)
---------

最后对整件事进行复盘，需要吸取的教训有：

*   不要随便 fork 项目，况且是你明知他是违反 DMCA 的项目
*   及时备份自己的代码仓库、star 列表
*   各种网站登录，一定要有除第三方登录以外的另一种登录方式
*   努力沟通还是会有结果

希望对于现在其他遇到类似问题的朋友有所帮助。