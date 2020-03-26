.. _first-run-docker-label:

==============================
用Docker启动BuildBot
==============================

.. note::

    如果你没有用过Dockre，将BuildBot运行起来不是那么容易，如果你有问题，先通过下面的命令确认是BuildBot问题还是Docker的问题

    .. code-block:: bash

      docker run ubuntu:12.04 apt-get update

    如果失败了，运行 ``Docker install`` 的时候查看一下帮助。另一方面，如果成功了，是要比从Buildbot社区成员那里获得帮助可能会更好。


Docker_ 是一个能让构建与发布自定义环境变得容易的工具。


他使用一个轻量级的linux容器（LUX），能够很快执行，成为社区用来测试的一个好工具。
如果超过了三分钟才看到的成功的消息，那就尝试用pip方式启动BuildBot :ref:`首次运行 <getting-code-label>`。

.. _Docker: https://www.docker.com

当前docker依赖
---------------------------

* Linux system, with at least kernel 3.8 and AUFS support.
  For example, Standard Ubuntu, Debian and Arch systems.
* Packages: lxc, iptables, ca-certificates, and bzip2 packages.
* Local clock on time or slightly in the future for proper SSL communication.
* This tutorial uses docker-compose to run a master, a worker, and a postgresql database server

安装
------------

* 在你的系统中使用 `Docker 安装指令 <https://docs.docker.com/engine/installation/>`_ .

* 确保你已经安装 docker-compose.
  使用root权限或者是在虚拟环境中，执行:

  .. code-block:: bash

    pip install docker-compose

* 测试docker是否工作:

  .. code-block:: bash

    sudo docker run -i busybox /bin/echo Success

构建并运行BuildBot
-----------------------------

.. code-block:: bash

  # 下载样例仓库
  git clone --depth 1 https://github.com/buildbot/buildbot-docker-example-config

  # 构建BuildBot容器 (可能要画几分钟下载相关的包)
  cd buildbot-docker-example-config/simple
  docker-compose up


你应该可以进入 http://localhost:8010看到下面的web页面:

.. image:: _images/index.png
   :alt: index page

点击左侧"Builds"按钮，打开子菜单然后可以看到已经有一个worker连接到了master `Builders <http://localhost:8010/#/builders>`_

.. image:: _images/builders.png
   :alt: builder runtests is active.


docker-compose 配置概述
--------------------------------------------

This docker-compose configuration is made as a basis for what you would put in production
以下docker-compose配置是你投入使用的基本配置

- 每个组件的独立容器
- 一个独立的postgresql数据库
- 可以连接到buildbot master的docker host 配置
- 可以横向扩展的buildbot worker
- 容器链接在一起，暴露给外部的唯一端口是Web服务器
- 默认主容器基于Alpine linux，占用空间最小
- 默认的worker容器基于广为人知的Ubuntu发行版，这可能是你想用的容器。
- 通过web下载配置的压缩包

和BuidBot容器愉快的玩耍吧
-------------------------------------

如果你到了这一步，你已经又一个BuildBot环境，你可以自由发挥的实验。

如果你想更改配置，你需要fork这个仓库 https://github.com/buildbot/buildbot-docker-example-config 然后可以下载你自己的分支，然后再次运行docker-compose

通过编辑 master.cfg 文件更改你的配置，提交你的更改，然后推送到你的仓库中。
在你推送之前你可以使用命令 buildbot check-config 来确保配置的变更是有效的
你可能会更改 ``docker-compose.yml`` 变量 ``BUILDBOT_CONFIG_URL`` 来指向你的github分支

``BUILDBOT_CONFIG_URL`` 可能指向一个HTTP可以访问的 ``.tar.gz`` 文件.
几个像github这样的git服务器，可以从git仓库的master分支自动生成压缩包
如果 ``BUILDBOT_CONFIG_URL`` 不以 ``.tar.gz`` 为结尾，也可以在 ``master.cfg`` 文件中填写通过HTTP可以访问到的链接


定制你的worker容器
-------------------------------
建议跟你的工程构建需要来定制worker容器
以下是DockerFile样例，buildbot社区用于自己的CI

https://github.com/buildbot/metabbotcfg/blob/nine/docker/metaworker/Dockerfile

多个master
------------

多master环境可以通过设置  ``multimaster/docker-compose.yml`` 文件，在上面的样例中

  # 构建BuildBot容器 (花几分钟下载相关的包)
  cd buildbot-docker-example-config/simple
  docker-compose up -d
  docker-compose scale buildbot=4

冲呀
-------------

你刚才已经尝试了一下，不过你可能会对更多的东西好奇。我们在下一个教程我们会提高点难度，通过更改配置进行进行构建
继续查看 :ref:`quick-tour-label`.
