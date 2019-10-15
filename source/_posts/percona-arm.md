---
title: Percona-server ARM编译总结 
date: 2019-07-01 02:12:56
tags:
  - OpenLab
  - ARM
categories: OpenLab
---
# 背景

和Linux类似，Mysql也有众多发行版，Percona-server是其中一个。Percona-server由Percona公司发起，在Mysql的基础上对性能、功能进行了优化和增强。是一款使用免费、支持商业付费咨询的开源数据库软件。目前官方只有X86平台的release安装包。我们的目标是在ARM平台上对它进行测试并验证，推动Percona社区支持官方的ARM平台release。
<!-- more -->
# 挑战

1. Percona-server由C/C++编写而成，在ARM上对源码编译和打包有一定挑战。
2. Percona-server社区由商业公司把控，商业公司往往会有自己的利益和人力考虑，推动社区支持ARM有一定难度。

## 编译

本地测试使用的操作系统是CentOS 7.5。其Yum源上的编译器gcc和glibc库的版本无法满足percona-server的要求，因此需要先手动在本地通过源码编译gcc、glibc以及它们需要的依赖库。而Openlab CI环境使用的是Ubuntu16.04，其APT源的gcc等工具版本合适，能直接使用。命令如下：

```
# apt-get install -y python python-pip libmysqlclient-dev cmake gcc bison ncurses-dev libedit-dev libaio-dev openssl libssl-dev libreadline6 libreadline6-dev libcurl4-openssl-dev
# pip install mysql-python
```

下载percona-server代码，然后进入代码目录，创建build目录并进入：

```
# mkdir build
# cd build
```

执行cmake命令，生成MakeFile，再执行编译命令make：

```
# cmake -H.. -B. -DDOWNLOAD_BOOST=1 -DWITH_BOOST=/opt/boost/ -DCMAKE_BUILD_TYPE=RelWithDebInfo -DBUILD_CONFIG=mysql_release -DFEATURE_SET=community
# make -j8
```

耗时1个多小时，编译成功。然后执行test：

```
# ./mysql-test/mysql-test-run.pl --force --max-test-fail=0 --parallel=auto --suite-timeout=1440
```

Test非常大，在8U8G的VM中，大概耗时13+小时。最后成功9437个，失败92个，未执行2个，其中有一些Error的Test是Percona-server的已知问题。由于Mysql 8.0本身就支持ARM，而Percona-server的最新版就是基于Mysql 8.0的，因此ARM化的难度不高。

## 社区推动

在社区的JIRA上提了[issue](https://jira.percona.com/browse/PS-5637)。从社区的回复来看，人力不足是当前最大的阻力。由于Percona-server是一个商业公司主导的小型开源软件，参与人数很少，主要由Percona公司的人员维护。而他们目前没有精力支持ARM release，OpenLab也只是提供ARM CI平台而已，后续的ARM功能看护、QA、ARM release还得依靠Percona社区。因此社区推动工作目前已暂停。等待公司层面的推动。



# 总结

1. C++项目需要编译才能运行在指定平台，C++项目是否支持ARM平台则需要实际的测试。
2. OpenLab CI只负责Test测试。而Test的修复则需要Mysql的专业知识，不是短期能完成的，要么Percona社区的人帮忙修复，要么OpenLab投人参与Percona社区。一个项目想要支持ARM， CI只是第一步，我们还需要投入后续的CI看护、测试修复等工作。
3. 个人在商业公司把控的开源社区很难推送需求，需要公司从商业层面考虑合作。
