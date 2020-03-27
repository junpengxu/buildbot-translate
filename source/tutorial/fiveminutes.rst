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

So our builder is called ``simplebuild`` and can run on either of ``worker1``, ``worker2`` and ``worker3``.
If our repository has other branches besides trunk, we could create another one or more builders to build them; in the example, only the checkout step would be different, in that it would need to check out the specific branch.
Depending on how exactly those branches have to be built, the shell commands may be recycled, or new ones would have to be created if they are different in the branch.
You get the idea.
The important thing is that all the builders be named differently and all be added to the ``c['builders']`` value (as can be seen above, it is a list of ``BuilderConfig`` objects).

Of course the type and number of steps will vary depending on the goal; for example, to just check that a commit doesn't break the build, we could include just up to the ``make all`` step.
Or we could have a builder that performs a more thorough test by also doing ``make test`` or other targets.
You get the idea.
Note that at each step except the very first we use ``haltOnFailure=True`` because it would not make sense to execute a step if the previous one failed (ok, it wouldn't be needed for the last step, but it's harmless and protects us if one day we add another step after it).

Schedulers
----------

Now this is all nice and dandy, but who tells the builder (or builders) to run, and when?
This is the job of the `scheduler`, which is a fancy name for an element that waits for some event to happen, and when it does, based on that information decides whether and when to run a builder (and which one or ones).
There can be more than one scheduler.
I'm being purposely vague here because the possibilities are almost endless and highly dependent on the actual setup, build purposes, source repository layout and other elements.

So a scheduler needs to be configured with two main pieces of information: on one hand, which events to react to, and on the other hand, which builder or builders to trigger when those events are detected.
(It's more complex than that, but if you understand this, you can get the rest of the details from the docs).

A simple type of scheduler may be a periodic scheduler: when a configurable amount of time has passed, run a certain builder (or builders).
In our example, that's how we would trigger a build every hour::

    from buildbot.plugins import schedulers

    # define the periodic scheduler
    hourlyscheduler = schedulers.Periodic(name="hourly",
                                          builderNames=["simplebuild"],
                                          periodicBuildTimer=3600)

    # define the available schedulers
    c['schedulers'] = [hourlyscheduler]

That's it.
Every hour this ``hourly`` scheduler will run the ``simplebuild`` builder.
If we have more than one builder that we want to run every hour, we can just add them to the ``builderNames`` list when defining the scheduler and they will all be run.
Or since multiple scheduler are allowed, other schedulers can be defined and added to ``c['schedulers']`` in the same way.

Other types of schedulers exist; in particular, there are schedulers that can be more dynamic than the periodic one.
The typical dynamic scheduler is one that learns about changes in a source repository (generally because some developer checks in some change), and triggers one or more builders in response to those changes.
Let's assume for now that the scheduler "magically" learns about changes in the repository (more about this later); here's how we would define it::

    from buildbot.plugins import schedulers

    # define the dynamic scheduler
    trunkchanged = schedulers.SingleBranchScheduler(name="trunkchanged",
                                                    change_filter=util.ChangeFilter(branch=None),
                                                    treeStableTimer=300,
                                                    builderNames=["simplebuild"])

    # define the available schedulers
    c['schedulers'] = [trunkchanged]

This scheduler receives changes happening to the repository, and among all of them, pays attention to those happening in "trunk" (that's what ``branch=None`` means).
In other words, it filters the changes to react only to those it's interested in.
When such changes are detected, and the tree has been quiet for 5 minutes (300 seconds), it runs the ``simplebuild`` builder.
The ``treeStableTimer`` helps in those situations where commits tend to happen in bursts, which would otherwise result in multiple build requests queuing up.

What if we want to act on two branches (say, trunk and 7.2)?
First we create two builders, one for each branch (see the builders paragraph above), then we create two dynamic schedulers::

    from buildbot.plugins import schedulers

    # define the dynamic scheduler for trunk
    trunkchanged = schedulers.SingleBranchScheduler(name="trunkchanged",
                                                    change_filter=util.ChangeFilter(branch=None),
                                                    treeStableTimer=300,
                                                    builderNames=["simplebuild-trunk"])

    # define the dynamic scheduler for the 7.2 branch
    branch72changed = schedulers.SingleBranchScheduler(
        name="branch72changed",
        change_filter=util.ChangeFilter(branch='branches/7.2'),
        treeStableTimer=300,
        builderNames=["simplebuild-72"])

    # define the available schedulers
    c['schedulers'] = [trunkchanged, branch72changed]

The syntax of the change filter is VCS-dependent (above is for SVN), but again once the idea is clear, the documentation has all the details.
Another feature of the scheduler is that it can be told which changes, within those it's paying attention to, are important and which are not.
For example, there may be a documentation directory in the branch the scheduler is watching, but changes under that directory should not trigger a build of the binary.
This finer filtering is implemented by means of the ``fileIsImportant`` argument to the scheduler (full details in the docs and - alas - in the sources).

Change sources
--------------

Earlier we said that a dynamic scheduler "magically" learns about changes; the final piece of the puzzle are `change sources`, which are precisely the elements in Buildbot whose task is to detect changes in the repository and communicate them to the schedulers.
Note that periodic schedulers don't need a change source, since they only depend on elapsed time; dynamic schedulers, on the other hand, do need a change source.

A change source is generally configured with information about a source repository (which is where changes happen); a change source can watch changes at different levels in the hierarchy of the repository, so for example it is possible to watch the whole repository or a subset of it, or just a single branch.
This determines the extent of the information that is passed down to the schedulers.

There are many ways a change source can learn about changes; it can periodically poll the repository for changes, or the VCS can be configured (for example through hook scripts triggered by commits) to push changes into the change source.
While these two methods are probably the most common, they are not the only possibilities; it is possible for example to have a change source detect changes by parsing some email sent to a mailing list when a commit happens, and yet other methods exist.
The manual again has the details.

To complete our example, here's a change source that polls a SVN repository every 2 minutes::

    from buildbot.plugins import changes, util

    svnpoller = changes.SVNPoller(repourl="svn://myrepo/projects/coolproject",
                                  svnuser="foo",
                                  svnpasswd="bar",
                                  pollinterval=120,
                                  split_file=util.svn.split_file_branches)

    c['change_source'] = svnpoller

This poller watches the whole "coolproject" section of the repository, so it will detect changes in all the branches.
We could have said::

    repourl = "svn://myrepo/projects/coolproject/trunk"

or::

    repourl = "svn://myrepo/projects/coolproject/branches/7.2"

to watch only a specific branch.

To watch another project, you need to create another change source -- and you need to filter changes by project.
For instance, when you add a change source watching project 'superproject' to the above example, you need to change::

    trunkchanged = schedulers.SingleBranchScheduler(
        name="trunkchanged",
        change_filter=filter.ChangeFilter(branch=None),
        # ...
        )

to e.g.::

    trunkchanged = schedulers.SingleBranchScheduler(
        name="trunkchanged",
        change_filter=filter.ChangeFilter(project="coolproject", branch=None),
        # ...
        )

else coolproject will be built when there's a change in superproject.

Since we're watching more than one branch, we need a method to tell in which branch the change occurred when we detect one.
This is what the ``split_file`` argument does, it takes a callable that Buildbot will call to do the job.
The split_file_branches function, which comes with Buildbot, is designed for exactly this purpose so that's what the example above uses.

And of course this is all SVN-specific, but there are pollers for all the popular VCSs.

But note: if you have many projects, branches, and builders it probably pays to not hardcode all the schedulers and builders in the configuration, but generate them dynamically starting from list of all projects, branches, targets etc. and using loops to generate all possible combinations (or only the needed ones, depending on the specific setup), as explained in the documentation chapter about :doc:`../manual/customization`.

Reporters
---------

Now that the basics are in place, let's go back to the builders, which is where the real work happens.
`Reporters` are simply the means Buildbot uses to inform the world about what's happening, that is, how builders are doing.
There are many reporters: a mail notifier, an IRC notifier, and others.
They are described fairly well in the manual.

One thing I've found useful is the ability to pass a domain name as the lookup argument to a ``mailNotifier``, which allows you to take an unqualified username as it appears in the SVN change and create a valid email address by appending the given domain name to it::

    from buildbot.plugins import reporter

    # if jsmith commits a change, mail for the build is sent to jsmith@example.org
    notifier = reporter.MailNotifier(fromaddr="buildbot@example.org",
                                   sendToInterestedUsers=True,
                                   lookup="example.org")
    c['reporters'].append(notifier)

The mail notifier can be customized at will by means of the ``messageFormatter`` argument, which is a class that Buildbot calls to format the body of the email, and to which it makes available lots of information about the build.
For more details, look into the :ref:`Reporters` section of the Buildbot manual.

Conclusion
----------

Please note that this article has just scratched the surface; given the complexity of the task of build automation, the possibilities are almost endless.
So there's much, much more to say about Buildbot. However, hopefully this is a preparation step before reading the official manual. Had I found an explanation as the one above when I was approaching Buildbot, I'd have had to read the manual just once, rather than multiple times. Hope this can help someone else.

(Thanks to Davide Brini for permission to include this tutorial, derived from one he originally posted at http://backreference.org .)
