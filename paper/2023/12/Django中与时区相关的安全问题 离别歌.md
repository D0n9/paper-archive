# Django中与时区相关的安全问题 | 离别歌
在开发国际化网站的时候，难免会与时区打交道，通用CMS更是如此，毕竟其潜在用户可能是来自于全球各地的。Django在时区这个问题上下了不少功夫，但是很多资深的开发者都有可能尚未完全屡清楚Django中各种时间的实际意义和使用方法，导致写出错误的代码；作为安全研究人员，时区问题也可能和一些安全问题挂钩，比如优惠券的过期时间、订单的下单与取消时间等，如果没有考虑时区问题，有可能将导致一些逻辑漏洞。

本文就从多个常用模块开始，了解一下Django中的时区究竟是怎么回事，以及在时间的比较中可能出现的一些逻辑错误。

[从“两种时间”说起](#_1)
----------------

我们都知道，在Python中表示“时间”的对象是`datetime.datetime`。

其实在Python中，这个对象被分成了[两个类型](https://docs.python.org/3/library/datetime.html#aware-and-naive-objects)：

*   aware datetime
*   naive datetime

他们的区别是：如果`datetime`对象的`tzinfo`属性有设置时区值，则这个对象是一个aware datime；否则它是一个naive datetime。

举个例子，我们平时在编写Python脚本的时候，使用下面这行代码获取当前时间：

`from datetime import datetime

t = datetime.now()` 

此时，t是一个naive datetime，因为我们没有给他设置时区：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/52bd9c80-cd77-45ae-8848-d6372bff9191.png?raw=true)
](https://www.leavesongs.com/media/attachment/2020/10/11/a8ab3caf-a708-43e2-9f01-0f43ac20b005.png)

**naive**的中文意思大家应该都很熟悉，这里的大概意思就是“simple”，这是一个很简单、原始的时间对象。实际上就是指，计算机不知道这个时间，他的时区究竟是什么，它可能代表着北京时间，也可能是UTC时间，因为我们没有指定时区，我们无法“假设”其是计算机系统所在的时区，也无法“假设”其是UTC时区。也就是说，计算机拿到了一个naive datetime，是无法准确地定位到某一个时间点的，也无法直接转换成一个unix时间戳。

那么相对的，aware datetime就是计算机能准确知道其时区的时间对象，他是一个准确的时间点，就落在时间轴上的某个地方，不管从哪个时区看，这个点都是绝对固定的。所以，我们可以将一个aware datetime转换成unix时间戳。

有的同学可能比较好奇，你说naive datetime无法转换成时间戳，那么为什么这个对象有一个`timestamp()`方法呢：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/f93a980e-ce9b-4410-ae7c-00946295f801.png?raw=true)
](https://www.leavesongs.com/media/attachment/2020/10/11/5c1c8254-91bd-4e41-8fec-4d34fa7b025a.png)

原因我们[查文档](https://docs.python.org/3/library/datetime.html#datetime.datetime.timestamp)可以得出结论，如果对象是naive datetime，则会以当前系统本地时区为准。

[Django的时区配置](#django)
----------------------

回到Django。由于Django是一个国际化框架，时区相关处理自然是其必不可少的组成部分。Django的配置项中，有下面两个选项与时区相关：

*   `USE_TZ`
*   `TIME_ZONE`

`USE_TZ`用来指定整个项目是否**使用时区**，`TIME_ZONE`是默认时区的值。

如果`USE_TZ`的值设置为False，那么Django项目中所有时间都使用naive datetime（除非有明确指定时区的情况）。也就是说，网站内存储和使用的时间全部是`TIME_ZONE`的值所指定的时区。

这样做有一些弊端：

*   数据库中保存的是naive datetime，导致在跨区域迁移数据的时候，可能无法准确定位到某个时间点
*   国际化企业可能面向不同国家有不同的网站，但后台数据库相同，此时究竟使用哪个时区保存和展示时间，将引起混乱
*   即使是同一个网站的用户，他们可能来自于全球各地，查看到的时间却是统一的服务器时间，对于高交互式的应用十分不友好
*   即使网站面向的用户仅来自于某一个地区，也会涉及到“夏时令”（Daylight Saving Time）相关的问题，每年可能将会导致两次时间误差

默认情况下，用`django-admin`生成的项目，其设置中`USE_TZ`等于True，这也是Django官方建议的配置。此时，在网站内部存储与使用的是UTC时间，而与用户交互时使用`TIME_ZONE`或手工的时区。

我们后文中也以Django的默认配置`USE_TZ=True`为前提条件，否则也没有讨论的必要了。

[Django的时间函数](#django_1)
------------------------

Django的包`django.utils.timezone`中有下面几个常用的时间相关函数：

*   `now()`，返回当前的UTC时间
*   `localtime()`，返回当前的本地时间（默认是`TIME_ZONE`配置指定的时区时间）
*   `is_aware()`，传入的时间是否是aware datetime
*   `is_naive()`，传入的时间是否是naive datetime
*   `make_aware()`，将naive时间转换成aware时间
*   `make_naive()`，将aware时间转换成naive时间

因为开启了`USE_TZ`，Django内部操作时间时都应该使用aware时间，否则会出现异常。所以，我们在获取当前时间的时候，一定要使用Django自带的`now()`或`localtime()`函数，而不能使用Python的`datetime.datetime.now()`函数。

[数据库存储的时间](#_2)
---------------

我们在使用ORM的DatetimeField时，常常会有这样的疑虑：**我们究竟应该给DatetimeField传入哪个时区的时间呢？**

可以做个试验，编写下面这个model：

`class Archive(models.Model):
    title = models.CharField('title', max_length=256)

    now_time = models.DateTimeField(default=timezone.now)
    local_time = models.DateTimeField(default=timezone.localtime)` 

这个model有三个属性，title是他的名字，now\_time和local\_time是两个时间，他们的默认值分别是timezone.now和timezone.localtime。

也就是说，默认情况下，now\_time字段传入的是UTC时区的当前时间，local\_time字段传入的是本地时区的当前时间，我这里是`Asia/Shanghai`。

然后，我们创建一个Archive对象：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/aa531a82-34d6-4346-b855-9068b9ed0d6f.png?raw=true)
](https://www.leavesongs.com/media/attachment/2020/10/11/a5ce41e7-ecb9-44fb-8998-f60017bf5ec3.png)

可以发现，不管我们使用`a.now_time`还是`a.local_time`，读取到的datetime对象的tzinfo都是UTC。

这也印证了Django文档中说到的，不管传入的时间对象时区是什么，其内部存储的时间均为UTC时区。但是，值得注意的是，如果我们传入了一个不带时区的naive datetime，将会出现一个警告，并使用默认时区填充其tzinfo：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/57f72bab-c32f-4bf9-b42d-c1c3603a19ba.png?raw=true)
](https://www.leavesongs.com/media/attachment/2020/10/11/3ae8b50c-439f-452c-9fb5-755300a1e51d.png)

[模板中展示的时间](#_3)
---------------

对于网站的用户来说，他们想看到的时间显然不是UTC时间，而是某一个具体时区的时间。比如，我的网站几乎全部是中国用户，那么展示时使用的时区应该是`Asia/Shanghai`。

这一部分的转换，Django放在的模板引擎中。

Django在渲染模板变量时，将会遇到两种与时间有关的情况：

`<p>origin value: {{ object.now_time }}</p>
<p>date filter: {{ object.now_time | date:'Y-m-d H:i:s' }}</p>` 

前者是直接将时间渲染到页面中，后者是通过date这样的模板filter处理后渲染在页面中。这两种情况在内部处理方式略有不同此处不细表，总体而言，任意模板中变量的渲染，都会被转换时区。

那么，脱离模板引擎，我们会得到怎样的结果呢？

在流行的前后端分离架构中，后端服务器通常只提供JSON格式的接口给前端，那么，我们编写下面这样一个view，看看返回值是什么：

`from django.shortcuts import get_object_or_404
from django.http.response import JsonResponse
from django.utils import timezone

from . import models

def json(request):
    object = get_object_or_404(models.Archive, pk=1)
    data = dict(
        id=object.pk,
        now_time=object.now_time,
        local_time=timezone.localtime(object.local_time)
    )
    return JsonResponse(data=data)` 

返回对象的now_time，我直接将`object.now_time`返回；返回对象的local_time，我将数据库值转换成本地时间`timezone.localtime(object.local_time)`返回。

我前文说过，这两个值在数据库中的值是完全相等的，不过在json返回中，now\_time是UTC时间，而local\_time是北京时间：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/3c5b0142-2add-4460-bd36-4b8bf7b3632a.png?raw=true)
](https://www.leavesongs.com/media/attachment/2020/10/11/e3f034ba-f35a-48fc-a5c6-bc7a12f0797f.png)

也就是说，在前后端分离的网站中，如果直接使用Model的字段，那么前端需要负责进行时区的转换，否则将会出现时间的偏差。

[时间的校验和比较](#_4)
---------------

在一些业务场景下，我们可能会涉及到时间的校验和比较，如：

*   付费服务、商品、用户的有效期检查
*   活动的开始与结束时间检查
*   订单、商品的收货、取消时间检查

我们就以付费用户为例：用户购买了30天的VIP会员，我们需要给用户表中设置一个过期时间，比如下面这个model。

`from django.db import models
from django.utils import timezone

class Account(models.Model):
    username = models.CharField(max_length=256)
    password = models.CharField(max_length=64)

    created_time = models.DateTimeField(default=timezone.now)
    expired_time = models.DateTimeField()` 

如果某个用户某一个时刻对网站进行访问，我们如何判断他是否具有VIP权限呢？

通常情况下我们有两种常见的判断方法。一是，用户访问时，直接从model中取出这个对象，然后和`now()`进行比较：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/0ea6267c-e64a-48a2-af85-633dcc3c4434.png?raw=true)
](https://www.leavesongs.com/media/attachment/2020/10/11/e24594c4-e8ce-4f13-aa95-a047c67f1543.png)

这种情况下，当前时间不管是`now()`还是`localtime()`都不影响比较的结果，因为两个datetime对象在比较时会考虑时差。

另一种情况是，通过ORM的queryset进行比较，等于在数据库层面进行操作：

`if models.Account.objects.filter(expired_time__gt=timezone.now()).exists():
    # doing sth` 

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/a3e0ade8-2f49-4fb7-b207-a7b40c799fe0.png?raw=true)
](https://www.leavesongs.com/media/attachment/2020/10/11/d4581df0-274e-4d34-b80b-7734bc81cd22.png)

Django也帮我们考虑过这种情况，即使此时我们使用本地时间`timezone.localtime()`进行查询，系统也会将其转换成UTC时间传入SQL语句：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/fc9a2544-5e74-4415-a988-be583608ed77.png?raw=true)
](https://www.leavesongs.com/media/attachment/2020/10/11/4942649c-450c-4201-9de9-1554f9e72651.png)

但是，如果我们使用到了和日期、时间有关的lookups，将产生相反的结果。

怎么理解这个问题呢，我们还是来举个例子。比如，网站以用户注册当天的日子作为“会员日”（比如1月2日注册的会员，以后每月的2日都是他的会员日），会员日这一天会给这个用户赠送优惠券。

那么，发送优惠券时，我们如何筛选网站内会员日为今日的用户呢？

下面这个filter是否正确？

`models.Account.objects.filter(created_time__day=timezone.now().day).all()` 

答案是**否定的**，我们应该使用`timezone.localtime()`表示**今天**，而非`timezone.now()`：

`models.Account.objects.filter(created_time__day=timezone.localtime().day).all()` 

这是为什么呢？你不是说数据库中存储的都是UTC时间吗，为何会使用到`timezone.localtime()`？

原因是，Django在使用日期、时间有关的lookups时，会在数据库层面对时间进行时区的转换再进行比较，所以我们需要使用本地时间而不是UTC时间。

可以看看原始的SQL语句：

[![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/12/00765265-f4f5-45df-aa4e-f27703060cce.png?raw=true)
](https://www.leavesongs.com/media/attachment/2020/10/11/a4e41479-8e41-4485-b67c-6bbf85fcec21.png)

可见，SQL语句中使用了`django_datetime_extract('day', "sample_account"."created_time", 'Asia/Shanghai', 'UTC')`将UTC时间转换成了北京时间，因此后面比较的时候，也应该使用北京时间。

这一点需要格外注意。时间比较的不谨慎，说小点是一个Bug，说大点就是漏洞，毕竟很多涉及到时间比较的情景，都是非常需要严谨的。

所以，我们总结一下：

*   任何比较都使用aware时间，不能使用naive时间
*   时间属性直接比较时，使用任何aware时间均可（会被自动转换成UTC）
*   queryset查询，不涉及`__day`、`__date`、`__year`等时间lookups时，使用任何aware时间均可（会被自动转换成UTC）
*   queryset查询，涉及到时间lookups时，使用本地时间