.. _fiveminutes:

===============================
一个用户贡献的五分钟BuildBot教程
===============================

（好吧，可能是十分钟）

BuildBot的确是一款出色的软件，不过对于新人来说可能有点难懂（像我刚开始看的时候一样）。通常，初次来看它看起来像一堆复杂的概念，没有任何意义，并且它们之间的关系也不清晰。

你需要多次阅读才会恍然大悟，事情才开始变得有意义。一旦你到了这种程度，你会知道文档写的是什么，你才会意识到文档很棒。
至少对我来说是这样的。在这里，我将会用一种对新人容易理解的方式去解释。

我讲述的方法可能和文档中写的或多或少有些相反，我将会从实际工作的组件开始讲起（builders）然后反着去讲述。希望那些有规则洁癖的人不要怪我
这里我只会去澄清概念，并不会详细的讲述每个对象或者属性的细节，文档里面写的很好了。

安装
------------

我就不写怎么安装的了，任何情况下，官方文档中的说明均适用与任何主要发行版的Buildbot master和worker。本文档将会引用Buildbot 0.8.5版本，幸运的是其他的版本都是差不多的。
所有的代码都是python写的，并且都包含 master.cfg 主配置文件。

我本不过多关注基础，比如怎么去定义workers和项目的名字或者是其他的配置文件中有的信息，如果想了解这些的话，去看官方文档吧。

主力：Builders
------------------------

BuildBot是一个将软件自动化构建作为目标的工具。对我而言，从告诉Buildbot如何构建软件的位置开始是有意义的：构建器（一个或多个构建器，因为可以有多个）
简而言之，构建器是负责执行某些操作或一系列操作的元素，通常与构建软件有关（例如，检查源代码或`` make all  ``），它也可以运行任意命令。

一个builder配置了多个可以执行任务的worker。
builder需要的另一条基本信息是它必须要做的事情的清单（通常将在选定的worker上运行）在BuildBot中，这个清单是一个``BuildFactory`` 对象，其本质是一系列步骤，每个步骤定义一个特定的操作或命令。

说的够多了，我们来看的样例吧。我们假设我们的项目可以通过使用简单的命令 ``make all`` 执行构建，通过``make packages`` 可以完成打包操作，打包成rpm，deb，或者是tgz的二进制包。
真实的情况可能更加复杂，（例如可能会有个 ``configure`` 配置的步骤或者多个）但是概念是相同的。这只是向builder添加更多步骤或创建多个builder的问题，尽管有时生成的builder可能非常复杂

 (assuming we are at the root of the local copy of the repository):
这里演示手动构建一个我们的工程，我们将输入这些命令行，假设我们位于项目仓库的的根目录

.. code-block:: bash

    $ make clean    # 清除先前版本的残留物
    ...
    $ svn update
    ...
    $ make all
    ...
    $ make packages
    ...
    # 这是可选的操作，将包拷贝到其他机器中
    $ scp packages/*.rpm packages/*.deb packages/*.tgz someuser@somehost:/repository
    ...

假设我们使用SVN，其实使用git，mercurian或这其他的版本控制工具也是一样的

现在，让整个过程自动化起来，我们创建一个builder，每一个步骤都是我们上面输入的命令
一个操作步骤可以是一句shell命令或者源代码中的命令。（不同的仓库用法不同，可以参考文档）或者其他的信息::

    from buildbot.plugins import steps, util

    # 首先，我们单独创建每个步骤

    # 步骤 1: make clean; 如果worker没有本地副本，可能会失败，只会在第一次运行的时候发生，不过影响很小
    makeclean = steps.ShellCommand(name="make clean",
                                   command=["make", "clean"],
                                   description="make clean")

    # step 2: svn 更新 (可以查看文档怎么更新).
    checkout = steps.SVN(baseURL='svn://myrepo/projects/coolproject/trunk',
                         mode="update",
                         username="foo",
                         password="bar",
                         haltOnFailure=True)

    # step 3: make all
    makeall = steps.ShellCommand(name="make all",
                                 command=["make", "all"],
                                 haltOnFailure=True,
                                 description="make all")

    # step 4: make packages
    makepackages = steps.ShellCommand(name="make packages",
                                      command=["make", "packages"],
                                      haltOnFailure=True,
                                      description="make packages")

    # step 5: 将包上传到中心服务器. 这一步需要通过不需要密码的ssh连接（提前设置好）
    uploadpackages = steps.ShellCommand(
        name="upload packages",
        description="upload packages",
        command="scp packages/*.rpm packages/*.deb packages/*.tgz someuser@somehost:/repository",
        haltOnFailure=True)

    # 创建一个build工厂，将上面的步骤加入
    f_simplebuild = util.BuildFactory()
    f_simplebuild.addStep(makeclean)
    f_simplebuild.addStep(checkout)
    f_simplebuild.addStep(makeall)
    f_simplebuild.addStep(makepackages)
    f_simplebuild.addStep(uploadpackages)

    # 最后，声明builder列表，在这里，我们只有一个build
    c['builders'] = [
        util.BuilderConfig(name="simplebuild", workernames=['worker1', 'worker2', 'worker3'],
                           factory=f_simplebuild)
    ]

所以我们的builder只是一个 ``simplebuild`` ， 可以被运行在 ``worker1``, ``worker2`` 和 ``worker3`` 上
如果我们的仓库有其他的分支，我们也可以创建更多的builder去构建他们，在上面的演示中，只有checkout的部分会不一样，只需要特殊指定分支，shell 命令也可以重复使用，
重要的一点是，所有的builder名称不同，并且都会被加入到 ``c['builders']`` 的值中（就像是我们在上面例子中看到的，他们是一个  ``BuilderConfig`` 对象列表

当然，步骤的类型和数量将取决于目标。例如仅检查一次提交不会破坏构建，我们可以包括直至 ``make all``步骤，或者，我们也可以让builder通过执行制造测试或其他目标来执行更全面的测试。你能了解吧，
请注意，除了第一步以外，在每个步骤中，我们都使用haltOnFailure = True，因为如果前一个步骤失败，则执行步骤就没有意义（好的，这不是最后一步所必需的，加不加都可以，如果有一天我们在其后增加另一步操作，则可以保护我们）

调度者
----------

现在，一切都好，但是谁来告诉builder什么时候去运行呢？这是调度者的工作，调度者是一个等待某个事件发生的因素的名称。并且何时发生，根据该信息决定是否以及何时运行构建器（或者运行一个或多个构建器）。
我在这里含糊其词，因为可能性几乎是无限的，并且高度取决于实际的设置，构建目的，源仓库设置和其他因素。

因此，调度程序需要配置两条主要信息：一方面，对哪些事件做出反应，另一方面，在检测到这些事件时触发哪些构建器或哪些构建器。（实际上复杂多了，如果你都看明白了，剩下的细节都可以通过文档找到）

一个简单的调度器可能是一个定期调度器，当设置的时间过去后，运行一个确定的builder或者多个builder，在我们的例子中，每小时触发一次构建。::

    from buildbot.plugins import schedulers

    # 定义定期调度程序
    hourlyscheduler = schedulers.Periodic(name="hourly",
                                          builderNames=["simplebuild"],
                                          periodicBuildTimer=3600)

    # 定义可用的调度程序
    c['schedulers'] = [hourlyscheduler]

就是这了， ``hourly`` 调度器每小时会执行一次 ``simplebuild`` 构建器，如果我们有多个构建器也想每小时运行一次，我们也可以把他们加入到  ``builderNames`` 列表中，他们最后也会被运行。

使用多个调度程序也可以，可以用同样的方式将其他调度程序添加到 ``c['schedulers']`` 中

也有其他类型的调度程序，特别是有些调度程序比周期性的调度程序更具动态性，典型的动态调度程序是一种了解代码仓库中的更改的动态调度程序（通常是因为某些开发人员加入了某些更改），并响应这些更改触发一个或多个构建器

现在，让我们假设调度程序“神奇地”了解了代码仓库中的更改（稍后会详细介绍）；这是我们的定义方式::

    from buildbot.plugins import schedulers

    # 定义一个动态调度器
    trunkchanged = schedulers.SingleBranchScheduler(name="trunkchanged",
                                                    change_filter=util.ChangeFilter(branch=None),
                                                    treeStableTimer=300,
                                                    builderNames=["simplebuild"])

    # 定义一个可用的调度器
    c['schedulers'] = [trunkchanged]

这个调度者接收仓库的改变，并且在所有这些更改中，请注意“trunk”中发生的更改（这就是branch = None的意思）
总结来说，它会过滤更改以仅对感兴趣的更改做出处理，当检测到此类更改，并且"tree"已静默5分钟（300秒）时，它将运行simplebuild构建器
使用 ``treeStableTimer`` 有助于突然提交的情况下，否则将导致多个构建请求排队。

如果我们想在两个分支（例如，trunk和7.2）上采取行动怎么办？
首先，我们创建两个 builder，每个builder对应一个分支（可以参考builder段落），然后我们创建两个调度器::

    from buildbot.plugins import schedulers

    # 为trunk定义一个调度器
    trunkchanged = schedulers.SingleBranchScheduler(name="trunkchanged",
                                                    change_filter=util.ChangeFilter(branch=None),
                                                    treeStableTimer=300,
                                                    builderNames=["simplebuild-trunk"])

    # 为7.2 分支定义一个调度器
    branch72changed = schedulers.SingleBranchScheduler(
        name="branch72changed",
        change_filter=util.ChangeFilter(branch='branches/7.2'),
        treeStableTimer=300,
        builderNames=["simplebuild-72"])

    # 将调度器可用
    c['schedulers'] = [trunkchanged, branch72changed]

变更过滤器的语法取决于VCS（以上是针对SVN的），但是一旦思路清晰，该文档就会包含所有详细信息。
调度程序的另一个功能是，它可以告诉你哪些变化是重要的，哪些不重要。
例如，调度程序正在监视的分支中可能有一个文档目录，但是该目录下的更改不应触发二进制文件的构建。
这种更精细的过滤是通过调度程序的 ``fileIsImportant`` 参数实现的（文档中的详细信息，以及源码中的全部）。

变更资源
--------------

前面我们曾说过，动态调度程序可以“神奇地”了解更改。
难题的最后一部分是变更源，这些正是Buildbot中的元素，其任务是检测存储库中的变更并将其传达给调度程序。
请注意，定期计划程序不需要更改源，因为它们仅取决于经过的时间；
另一方面，动态调度程序确实需要更改源

通常，更改源配置有有关源存储库（发生更改的位置）的信息；
变更源可以监视存储库层次结构中不同级别的更改，因此例如可以监视整个存储库或其一部分，或者仅监视单个分支。
这确定了传递给调度程序的信息范围。

变更源可以通过多种方式了解变更。
它可以定期轮询存储库以查找更改，也可以配置VCS（例如通过提交触发的挂钩脚本）将更改推送到更改源中。
虽然这两种方法可能是最常见的，但它们并不是唯一的可能性。
例如，更改源可以通过在提交发生时解析发送到邮件列表的某些电子邮件来检测更改，并且还存在其他方法。
该手册再次有详细信息。

为了完成我们的示例，这里有一个更改源，该更改源每2分钟轮询一次SVN存储库::

    from buildbot.plugins import changes, util

    svnpoller = changes.SVNPoller(repourl="svn://myrepo/projects/coolproject",
                                  svnuser="foo",
                                  svnpasswd="bar",
                                  pollinterval=120,
                                  split_file=util.svn.split_file_branches)

    c['change_source'] = svnpoller

该轮询器监视存储库的整个“ coolproject”部分，因此它将检测所有分支中的更改。
我们可以说::

    repourl = "svn://myrepo/projects/coolproject/trunk"

或者::

    repourl = "svn://myrepo/projects/coolproject/branches/7.2"

来监听特殊的分支。

要监听另一个项目，你需要创建另一个更改源–并且需要按项目过滤更改。
例如，当你在上面的示例中添加监听更改源的项目“ superproject”时，您需要进行以下更改::

    trunkchanged = schedulers.SingleBranchScheduler(
        name="trunkchanged",
        change_filter=filter.ChangeFilter(branch=None),
        # ...
        )

例如::

    trunkchanged = schedulers.SingleBranchScheduler(
        name="trunkchanged",
        change_filter=filter.ChangeFilter(project="coolproject", branch=None),
        # ...
        )

coolproject将在superproject项目变更后构建

由于我们正在观察多个分支，因此我们需要一种方法来在检测到一个分支时告诉更改发生在哪个分支。
这就是split_file参数所做的事情，Buildbot将调用该Callable来完成这项工作。
Buildbot带有的split_file_branches函数正是为此目的而设计的，因此上面的示例使用了该函数

当然，这都是特定于SVN的，但是对于所有流行的VCS都有轮询器。


但请注意：如果您有许多项目，分支和构建器，则可能不对配置中的所有调度程序和构建器进行硬编码，而是从所有项目，分支，目标等的列表开始动态生成它们，并使用循环生成所有
可能的组合（或仅需要的组合，取决于特定的设置），如有关自定义的文档章节所述。 :doc:`../manual/customization`.

报告者
---------

现在已经具备了基础知识，让我们回到构建者那里，这是实际工作的地方。
报告者只是Buildbot用来向世界通报正在发生的事情（即建造者的工作状况）的手段。
报告者有很多：邮件通知者，IRC通知者和其他。
在手册中对它们进行了很好的描述。

我发现有用的一件事是能够将域名作为查找参数传递给mailNotifier的功能，该功能允许您采用SVN更改中出现的不合格用户名，并通过附加给定域名来创建有效的电子邮件地址::

    from buildbot.plugins import reporter

    # if jsmith commits a change, mail for the build is sent to jsmith@example.org
    notifier = reporter.MailNotifier(fromaddr="buildbot@example.org",
                                   sendToInterestedUsers=True,
                                   lookup="example.org")
    c['reporters'].append(notifier)

可以通过messageFormatter参数随意自定义邮件通知程序，该参数是Buildbot调用以格式化电子邮件正文的类，并且该类可提供有关构建的大量信息
了解更多细节， 参考 :ref:`Reporters` Buildbot 手册中相关章节.

总结
----------

请注意，本文只是从头开始。
考虑到构建自动化任务的复杂性，可能性几乎是无限的。
因此，关于Buildbot还有很多要说的。
但是，希望这是阅读正式手册之前的准备步骤。
在我接触Buildbot时，如果能找到上述解释，那么我只需要阅读一次而不是多次阅读该手册。
希望这可以帮助其他人

（感谢Davide Brini允许发布此教程，该教程源于他最初在http://backreference.org上发布的教程。）