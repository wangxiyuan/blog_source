---
title: 交叉编译概述
date: 2020-01-19 14:13:00
tags:
  - Compile
categories: Other
---
# 1. 概述

## 1.1 什么是交叉编译？

简而言之，交叉编译（Cross Compile）是指在一种平台上编译出另一种平台上的可执行代码或目标文件的过程。
<!-- more -->
## 1.2 为什么需要交叉编译？

1. 目标平台的机器性能不足以支持本地编译过程。

   例如早期的安卓手机、小型单片机或片上系统，这种机器的CPU、内存、存储有限，不能满足编译过程的要求，只能在其他机器（如X86服务器）上编译。

2. 成本考虑、资源受限，手中没有目标平台的设备。

   这种情况往往发生在测试中，比如手头只有一个A平台的机器，但要求编译出可以在B平台执行的代码。

3. 维护易用性。

   X86平台已经深入人心，在X86平台操作可以避免各种奇怪的问题。

## 1.3 什么语言需要交叉编译？

Java、Python等语言，它们有专门的底层运行时，如JRE、CPython等，天生支持多平台。而C/C++则通过编译生成可在目标平台上执行的二进制文件，因此需要针对不同平台编译出不同结果。因此本文主要讲C/C++的交叉编译。

# 2. 编译器

C/C++常用的编译器有Gcc和Clang。Gcc是我常使用的编译器。Clang暂时还不了解，以后再补充。

## 2.1 Gcc

Gcc本身有很多二进制可执行文件组成，它也是由编译生成的。因此一个已经编译好的Gcc软件，它能编译出的目标平台也是固定的。例如在一台已经安装了gcc的X86机器上执行`gcc -v`，可以看到类似结果：

```
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/lib/gcc/x86_64-linux-gnu/5/lto-wrapper
Target: x86_64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Ubuntu 5.4.0-6ubuntu1~16.04.12' --with-bugurl=file:///usr/share/doc/gcc-5/README.Bugs --enable-languages=c,ada,c++,java,go,d,fortran,objc,obj-c++ --prefix=/usr --program-suffix=-5 --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --with-sysroot=/ --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-vtable-verify --enable-libmpx --enable-plugin --with-system-zlib --disable-browser-plugin --enable-java-awt=gtk --enable-gtk-cairo --with-java-home=/usr/lib/jvm/java-1.5.0-gcj-5-amd64/jre --enable-java-home --with-jvm-root-dir=/usr/lib/jvm/java-1.5.0-gcj-5-amd64 --with-jvm-jar-dir=/usr/lib/jvm-exports/java-1.5.0-gcj-5-amd64 --with-arch-directory=amd64 --with-ecj-jar=/usr/share/java/eclipse-ecj.jar --enable-objc-gc --enable-multiarch --disable-werror --with-arch-32=i686 --with-abi=m64 --with-multilib-list=m32,m64,mx32 --enable-multilib --with-tune=generic --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=x86_64-linux-gnu
Thread model: posix
gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.12)
```

注意看`Configured with`中的`--build`、`--host`和`--target`，需要重点说明一下。

* `--build`是指编译该软件所使用的平台，即这个gcc软件是在x86平台下编译出来的。
* `--host`指该软件将运行的平台。即这个gcc软件是运行在x86平台的。
* `--target`指该软件所处理的目标平台。即gcc编译出的软件可以运行在x86平台上。

因此这个gcc是一个被X86平台编译出来、运行在x86平台上并支持编译出X86平台软件的编译器。

显然这个gcc满足不了我们交叉编译的要求。当我们想在X86平台上编译出arm平台的目标文件时，我们就需要一个`--host=X86` 、`--target=arm`的gcc。这样的gcc可以通过gcc源码编译出来，也可以直接下载别人编译好的可执行文件。在ubuntu中执行`apt search aarch64`，会发现这样两个软件` gcc-aarch64-linux-gnu `和` g++-aarch64-linux-gnu `。它们就是满足我们要求的gcc。安装后执行` aarch64-linux-gnu-gcc -v `，会发现` Configured with `中的build、host和target信息：

```
--build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=aarch64-linux-gnu
```

OK，这个gcc是一个被X86平台编译出来、运行在x86平台上并支持编译出aarch64平台软件的编译器。使用`aarch64-linux-gnu-gcc`命令就可以在X86平台编译出aarch64平台上可执行的软件啦。



补充：

1. `--target`一般只用在 gcc、binutils等于平台指令相关软件中，多数软件此参数无用处。

## 2.2 Clang

To Be Done。

# 3. 设置

一般在大型项目中没人会手动调用gcc等编译器，而是通过各种工具封装，由工具自动调用gcc执行编译。下面介绍几个常用的工具。

## 3.1 make

一般C/C++语言的项目使用`make`软件来编译。而`make`通过Makefile获取编译信息。C/C++项目中通常会配一个`configure`文件来生成Makefile或编译需要的具体参数值。在交叉编译场景下，使用命令`./configure --build xxx --host xxx`来指定执行机和目标机器，`--build`一般可以省略，默认指当前机器。在第二章提到的x86环境中，若想要编译出aarch64平台的目标文件，命令可以写成`./configure --host aarch64-linux-gnu`。



`--host`为什么是`aarch64-linux-gnu`，而不是其他的值，比如``aarch64`？其实`aarch64-linux-gnu`是指将要调用的gcc命令的前缀。真正编译时make调用的底层gcc命令全称就是`aarch64-linux-gnu-gcc`。在上述机器中，其实可以发现有一组以` x86_64-linux-gnu `开头的命令，这就是默认非交叉编译场景下实际调用的命令，与`--host=x86_64-linux-gnu`相对应。

## 3.2 Cmake

Cmake是对make的再一层封装，用来简单方便的快速生成Makefile。Cmake提供了一系列系统变量，通过修改这些变量可以快速实现交叉编译配置。Cmake命令一般形如`cmake -D key1=value1`，一般使用 `CMAKE_TOOLCHAIN_FILE`指定编译时需要用的文件，然后在该文件中具体指定编译细节。例如：

```
cmake -D CMAKE_TOOLCHAIN_FILE=./my_cross_compile_conf
```

在my_cross_compile_conf中，可以使用set命令指定一些参数，例如：

```
CMAKE_SYSTEM_NAME - 目标机target所在的操作系统名称
CMAKE_C_COMPILER - C语言编译器，交叉编译时指定为aarch64-linux-gnu-gcc
CMAKE_CXX_COMPILER - C++编译器，交叉编译时指定为aarch64-linux-gnu-g++
...

格式：
set(CMAKE_C_COMPILER /usr/bin/aarch64-linux-gnu-gcc)
```

## 3.3 Java

Java项目中有时会集成C/C++语言代码，因此这些Java项目在不同平台上使用时，是需要特殊编译的。这些C/C++代码可以手动使用gcc编译后集成到java中，也可以在java项目中统一处理。尤其是在大型java项目中，往往集成了编译、打包、发布的工具。Maven、Gradel是其中比较常见的工具。下面我们看看如何在java项目的工具中编译C/C++。

### 3.3.1 Maven

Java中有两种C/C++的编译场景：

1. 直接编译C/C++并使用。

   针对这种场景，使用` maven-antrun-plugin `插件。该插件支持用户自定义命令，使用它提供的 exec 命令，直接调用gcc或make命令即可。交叉编译的话，就调用aarch64-linux-gnu-gcc或make中指定。例子：

   ```
   <plugins>
     <plugin>
   	<artifactId>maven-antrun-plugin</artifactId>
   	<executions>
   	  <execution>
   		<id>build-native-lib</id>
   		<phase>generate-sources</phase>
   		<goals>
   		  <goal>run</goal>
   		</goals>
   		<configuration>
   		  <target>
   			<exec executable="make" failonerror="true" resolveexecutable="true">
   			  <env key="CC" value="gcc" />
   			  <env key="AR" value="ar" />
   			</exec>
   		  </target>
   		</configuration>
   	  </execution>
   	</executions>
     </plugin>
   </plugins>
   
   交叉编译的话，改下value即可
   <env key="CC" value="aarch64-linux-gnu-gcc" />
   <env key="AR" value="aarch64-linux-gnu-ar" />
   ```

2. 把C/C++编译打包成jar使用。

   Maven中有个插件`maven-hawtjni-plugin`，该插件用来编译C/C++成.so文件，并打包生成java可调用的jar。例如：

   ```
   <plugin>
   	<groupId>org.fusesource.hawtjni</groupId>
   	<artifactId>maven-hawtjni-plugin</artifactId>
   	<executions>
   	  <execution>
   		<id>build-native-lib</id>
   		<configuration>
   		  <configureArgs>
   			<arg>${jni.compiler.args.ldflags}</arg>
   			<arg>${jni.compiler.args.cflags}</arg>
   			<configureArg>--libdir=${project.build.directory}/native-build/target/lib</configureArg>
   		  </configureArgs>
   		</configuration>
   		<goals>
   		  <goal>generate</goal>
   		  <goal>build</goal>
   		</goals>
   	  </execution>
   	</executions>
   </plugin>
   
   交叉编译的话，configureArg处加一行：
   <configureArg>--host=aarch64-linux-gnu</configureArg>
   ```

### 3.3.2 Gradel

不太了解，以后补充。To Be Done。
