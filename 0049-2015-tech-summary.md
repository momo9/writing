# 2015 年度总结（技术篇）

本文的作者是**深深爱着人类**的「我」。

2016 要来了，先上一篇技术总结。

我建立了个人网站「momo9.me」，大家多去我那看看吧。点击「阅读原文」可以访问哈~

---

![](0049-title.JPG)

敬这一年来我挖的那些坑。

## 如何抛异常

之前一直是 C/C++ 为主，处理异常流就靠返回码，C++ 的异常完全没用过。学会了通过抛异常与 try...catch 来处理异常流之后，对比之下能够发现，与返回码比起来，抛异常的代码写起来方便很多。

一开始的时候，通常是一遇到异常流，就抛出一个异常，其它什么也不管。慢慢才发现，异常也不是乱抛的：

* 一抛异常，整个流程就中断了；可很多时候，流程中断并不是我们想要的，这时候就要想办法不抛异常或者 try...catch 住
* 有些异常一抛，可能就抛到用户那里了，让用户觉得这个操作是有问题的（或者程序有 bug）；可有些时候，比如删除一个不存在的实体，虽然也是异常流，但并不说明确实有问题，不一定要通过抛异常来表达
* 抛异常的时候，可能流程已经做了一半，那么之前做过的事情，需不需要回滚？获取的资源，需不需要释放？这也是抛异常的时候需要考虑的问题

## 墨菲定律

异常的情况总是发生，没有什么是不可能的。

编码上典型的就是空引用的问题，稍不注意就是 NullPointerException。

另一个很常见的就是用户的各种奇怪输入，偷懒不注意校验的话会导致各种异常。

业务上让我印象深刻的：

* 机器多了，就必然会有几台宕机：我们一开始没有考虑这个问题，流程要所有机器成功才能继续往下跑；宕机的机器当然是没法成功的，遇到这种情况流程会跑不下去，结果引入了很多东西来填这个坑
* 任何系统都会有不稳定的时候，要考虑调用系统无法访问的时候会发生什么：比如一段代码，先做一些事情，再调用一个外部接口，调用的时候恰好碰到对方重启，调用失败；如果没有考虑好这种情况，开始做的那些事情会引入脏数据，就可能导致程序不能重试，陷入一种卡住的状态

## 不要 block

这一点其实和上一点有很大的关联。正常流一般不 block，可怕的就是有些没考虑到的异常流出现的时候，陷入一种 block 的状态，既没法重试也没法绕过。

比如我们有段校验服务器的代码，最初的想法是校验没通过就不让用户过去，以免过去了以后机器不靠谱，要花很多时间来支持。这样就没有考虑到机器会有宕机的情况，也没考虑到校验的 API 会有不靠谱的时候，一旦出现这两种情况，用户就会卡在校验的地方。这种情况，是要避免出现的。

## 数据量的影响

在设计，实现，甚至直到测试的时候，我们可能都只考虑了典型但数据量很小的场景。从逻辑上来说，数据量的大小可能不会带来什么差别，但却会带来一些其它的问题，我遇到的比较典型的有：

* 对外提供一个同步的 API，当数据很大的时候，无法在 Nginx 的默认超时时间（一分钟）内完成，结果要想办法做成异步的
* 调用外部的 API，所有数据都走 HTTP，当数据量很大的时候，会超出 Nginx 对 HTTP 请求大小的默认限制
* 还有一些体验上的
    * 一些需要用户选择的地方，既无搜索，也不排序，测试的时候选项少没觉得有什么问题，生产环境中选项特别多，基本没法用
    * 一些展示信息的地方，列表页几乎没有任何信息，就一个名字，细节需要用户点到详情页面里才能看到，也是到生产环境中数据很多的时候才发现，一个一个点简直要累死，最好是能在一个页面里就看到重要信息的摘要
    * 一些需要操作的地方也是一样，如果没有批量操作的话，数据量大的时候是会累死人的
    
## 兼容性

加新的功能有时候也会影响到老的功能，这个时候就要尽量考虑兼容性。虽然说即使新的代码不能兼容老的数据，也可以通过洗数据的方法来解决，但这样做还是有很大风险的：

* 洗数据和代码上线不可能是同步的，这个时间差内不知道会出现什么问题
* 洗数据不一定能洗的干净
* 最重要的，万一新上去的东西有问题呢？回滚代码是比较容易，至于回滚数据嘛……

另外就是做新功能的时候，原来运行正常的代码最好不要去动（我只是说做新功能的时候，不是说代码质量差不要去重构），代码考虑在外面包一层，数据库也可以考虑加表而不是加字段：

* 第一个原因很直观，原来正常的东西，改一下可能改出 bug 来了
* 另一个原因就是一段代码不止一个人在用的，随便改会影响到别人；虽然可以用各种工具追踪到当前的引用情况，可以后的事情谁能知道呢？

## 程序的结构

说来惭愧，我自己写代码的时候，多次被 if...else... 给绕晕了，编出了低级的 bug。因此我比较喜欢在一个方法开始的时候就做异常校验，抛异常或者直接返回，尽量少用 else，这样就不太容易出逻辑问题。不过一直听说有些代码风格要求必须在方法结尾才能返回的，因此也不知道自己的这种写法到底好不好。

然后就是一条大家都懂，但是确实很有用的道理了：每个方法都短一些。这个在写代码的时候是感知不到的，很长一个方法写下来，好像也没什么问题，调试也都顺利。到一段时间之后再来读代码就麻烦了，长长的一串很难读懂，每段内容之间的界限很难区分，不好维护。

## 异步

如果说程序都是同步的话，那当然是很好啦，要简单多了。

但是，这是不可能的，我们总有会要用到多线程、定时任务、消息的时候。异步的东西要比同步的复杂不少，调试起来挺费劲的。对于这个，我有几点体会：

* 要能够控制异步的代码
    * 比如我们有多台机器，有个定时任务靠分布式锁控制它只能在一台机器上执行，如果没法控制有的机器不能抢占这个锁的话，那么在调试的时候，我们是没法保证调试用的机器一定能抢占到这个锁的，调试也就没法完成
    * 同样的道理，如果是消息的话，如果没法控制有的机器不能作为消费者，我们一样没法保证调试的机器一定能够消费掉消息，结果也是没法调试
* 监控异步代码的状态：和同步代码不同，异步的代码调用了以后，在主流程中往往不知道运行到哪里，也不知道结果什么时候完成，这个时候，最起码要把日志给打清楚，关键的地方留下痕迹；如果有必要的话，还可以给异步的任务做个控制台

## 技术态度

技术债总是要还的，这句话很有道理。遇到棘手的技术问题时，时间紧急的话可以想办法绕过，但之后要想办法正面攻克它，深入地了解原理，把问题的来龙去脉搞清楚，这样自身才能得到成长。如果仅仅是绕过的话，下次这个问题要是再出现（这种事情常常发生），搞不好就没法跳过了，那时候真是后悔莫及。

正面攻克问题，会让人有畏难心理。不过，只要不怕困难，耐心分析背后的原理，对问题的成因做出假设并一一验证，解决问题也并没有那么困难。有时候解决了问题再回头看，其实问题本身并没有想象的那么困难，最困难的反而是克服自己畏惧困难的心障。

不过，也许你也有过这样的体验：有时候面对一问题，其实也没有头绪，但是有一种信心，觉得自己就能解决它，开始碰了几次壁也没有气馁，再换个思路就解决了问题；有时候一看到问题就烦到不行，根本不想思考，脑子乱成一团，稍有碰壁就灰心丧气。

我觉得这个是由当时的状态决定的，起起伏伏很正常：

* 我认为状态其实是一种消耗品，解问题本身就会消耗状态，解得累了就烦了是很正常的事情；这时候，休息休息恢复状态是不错的选择，死撑着没有什么好处
* 一些微妙的心态变化其实会对状态有影响
    * 如果把解问题作为一个任务来做，那真的会挺烦的：问题经常是解了这里又冒出来那里，做任务如果越做越多，还发现后面还有任务，那绝对不是什么好体验
    * 如果把解问题看做一场自我提升的修行，一个有趣的游戏，效果就好一点：这种情况下会沉浸到问题的原理里面，那些奇怪的问题, 好像也没有那么可恶了
    
总之，能做得好一些就做得好一些，能了解得深入一些就了解得深入一些。如果交付的东西连自己都觉得很对付的话，丧失了的荣耀感是很难找回来的。
    
## 拆解任务

有的人能应付复杂一些的问题，而有些人只能应付简单一些的问题。但我觉得，这之间的差别，其实并不是很大。因为实际的问题往往都是很复杂的，如果把问题作为一个整体来看，问题的难度要超过人的能力范围。

![](0049-difficulty.png)

所以我觉得重点不是要去解一个很复杂很复杂的问题，那不仅很难搞定，做的过程中也非常容易出错。重点是要把复杂的事情拆解开，拆分为一个个很小很简单的事情，这才是解决问题的办法。

## 解 bug、日志与 debug 工具

出 bug 了，我觉得首先是要通过现象来分析问题，并根据现象对问题的原因做出一些假设。有了这些假设，再进入到代码的细节中，验证这些假设，最终解决问题。

如果一开始就进入到代码的细节中，是不利于解决问题的：

* 看代码，首先就是要当人肉编译器/解释器，这就消耗了很多精力，分散了解决问题的精力
* 代码中是有很多细节的，直接看代码的时候，我们很难对问题有整体的认识，结果就变成在代码的细节中穷举问题，而不是分析和解决问题

因此我认为，打好日志是解决问题的关键，而不是靠 debug 工具来解决问题：从日志中，我们看到的是能够帮助我们分析问题的现象；而 debug 工具能够帮我们获取的是代码运行的状态，是代码细节上的东西。

当然使用日志还有一些其他的原因，比如线上环境、异步代码不方便使用 debug 工具，比如有些代码不能停下来看效果。而 debug 工具也有它的优点，比如在看一段代码的时候，它让我们很容易地就了解到整个调用的链路，也还是挺有用的。

## 黑屏与白屏

Terminal 可以说是程序员的标志。很多命令行的工具，比如 sed、awk 之类的也确实好用，用起来也非常酷。一些个性化的事情，写个脚本临时解决一下，也挺灵活的。

可是要说命令行工具（黑屏）能够完胜具有用户界面的工具（白屏），就有点自欺欺人。就像被吹得很高的两大上古神器 VIM 和 Emacs，我觉得和现代的 IDE 相比还是有差距的（至少在 Java 开发中是如此）。对于黑屏还是不要盲目的迷信，工具这东西，找到适合自己的才是真。

## 严肃地看待前端

也许因为团队以后端为主，大家对待 Javascript 的态度稍微有点草率。在后端中大家比较注意的规则，在前端代码中都不太注意：重复代码、全局变量、面条代码随处可见，甚至缩进都不好好缩了……

Javascript 现在已经是个严肃的语言了，虽然原生 Javascript 本身不是那么严肃，但是这也能通过引入一些库来解决。最主要的是，还是要严肃地看待前端代码，像后端开发时一样写出容易维护的代码，别再让前端代码的维护如此之痛了。

## 冗余

冗余是能提高稳定性的。大自然可能说不出大道理，但他通过优胜劣汰的进化，造出的世间万物，却都有一些冗余：比如人有两个肾，坏了一个还能活……而我们写的代码呢，没有冗余，可能看起来很干净，结果出了一点小状况，就不能正常工作了。

其实更想说的是代码之外的冗余。我发现大家估算工作量通常都是偏乐观的，估算的都是「正常」情况下的工作时间。可是人生哪有正常的时候呢？状态会不好，会被别人打断，会有插入的事情……不留下冗余，不仅工作在边缘的自己是疲惫不堪的，而且总是延期对自己的信心也是一个打击。