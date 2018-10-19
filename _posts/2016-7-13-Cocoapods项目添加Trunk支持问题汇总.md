---
layout: post
title: "Cocoapods项目添加Trunk支持问题汇总"
subtitle: " Cocoapods project to add Trunk support problem summary"
author: "Zhaopin"
header-img: "img/post-bg-rwd.jpg"
header-mask: 0.4
tags:
  - Cocoapods
  - iOS
---

### 1、关于podspec文件

s.name 就是你的项目名了,通过之后可以使用pod search命令搜索到；

s.version 项目当前的版本号；

s.summary 项目的概要描述；

s.description 项目的详细描述；

s.license 许可文件，这个是cocoapods必须要求若没有会无法通过验证；

s.source 项目源代码位置，一般就是一个github地址；

s.default_subspec 项目默认加载的子包，由于我的项目是由多个包构成的所以我会添加这项，若项目只有一个包则不用填写这个参数；

比较重要的参数有：

core.source_files 项目主要文件；

core.public_header_files 暴露出的头文件；

core.vendored_libraries 项目封的静态库（根据项目类型：开源、闭源，没有可不填）；

core.resource 项目用的资源文件（图片之类的）；

core.frameworks 依赖的系统的framework框架 ；

core.ios.library 依赖的系统的lib库文件；

### 2、本地检查

提交之前，最好先进行本地检查，执行pod lib lint --verbose命令，verbose是为了出错时，有详细的错误描述。

执行命令后，常见的错误有：

1）- ERROR | license: Sample license type.

你的podspec文件中s.license是系统默认给你生成的那个，去掉后面的example就行

2）source: The Git source still contains the example URL.

你的s.source还是默认的那个example URL，改成你自己提交到github上的那个url就行，比如https://github.com/xxx/xxx.git

3）description: The description is empty.

添加你的描述

4）- ERROR | [OSX] unknown: Encountered an unknown error

这个错误一般是由于swift引起的（即使你的工程没有swift），The validator for Swift projects uses Swift 3.0 by default, if you are using a different version of swift you can use a `.swift-version` file to set the version for your Pod. For example to use Swift 2.3错误描述如上。这个时候你可以用touch .swift-version创建一个隐藏文件，然后打开这个隐藏文件，写入3.0即可。

5）- WARN  | summary: The summary is not meaningful.

这个是警告，意思是说你的summary是系统给生成的描述，你应该改成你自己的。

改完后，再执行一遍pod lib lint --verbose，成功后会看到下面的信息：

![pod_lint.png](https://upload-images.jianshu.io/upload_images/2005687-ea87893a94263102.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3、通过Trunk push podspec文件

如果你之前还没提交代码的话，先提交代码到github，

git add .

git commit -m"版本内容"

git push origin master

然后，这步很重要，给你的项目提交版本号标签，别人使用你的pods时，Trunk找的就是你的标签。

git tag 'tagNum' （tagNum是你的版本号，比如1.0.0）

git push --tags

所有准备工作完成之后，就可以开始最核心的工作trunk push了

pod trunk push yourProjectName.podsepc --verbose

yourProjectName是你的文件名，请修改成你自己的podspec名称

通过本地pod lib lint的文件一般而言都不会出什么问题，发生概率最大的问题可能就是由于网不给力，导致连接GitHub困难。上传成功之后，就可以使用pod search命令来搜索自己的项目了（可能得等上一段时间，有可能几个小时才能搜索到）。

### 4、后续项目的升级

当你的项目做出了修改之后，当然希望cocoapods中的版本也进行更新。此时就需要更新podspec描述文件了，将podspec文件改成符合你当前版本的需求之后。还需要给你GitHub上的版本打上tag，而且一定要和podspec中的s.version一致。

建议先修改podspec文件里的s.version，然后提交到github上。再用命令

```

git tag 'tagNum' (tagNum为你自己的版本号)

git push --tags

```

有时候会出现：

warning: Could not find remote branch 0.0.2 to clone.

fatal: Remote branch 0.0.2 not found in upstream origin

类似这种的，意思是找不到你的新版本号。这个时候可能是因为你使用了第三方git管理工具比如：sourcetree之类的，你虽然在sourcetree上标注了版本号，但是还没push上去，这个时候应该执行

git push --tags

然后再执行

pod trunk push xxx.podspec --verbose

就可以了。

### 5、执行pod trunk me时的错误

1、[!] Authentication token is invalid or unverified. Either verify it with the email that was sent or register a new session.

说明你的登录token过期了，需要执行pod trunk register 你的邮箱名，然后去你的邮箱验证一下，就可以了。
![pod_success.png](https://upload-images.jianshu.io/upload_images/6879404-5837c8382cadff12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

6、pod search 你的类库 失败的解决办法

有时候你提交成功了，也等了几小时了，依然没有search到你的库，其实不是你的类库没提交上去或者没更新，其实已经上去了。这个时候需要以下几个操作：

1、执行pod setup

会出现Setting up CocoaPods master repo，稍等几十秒，最底下会输出Setup completed。说明执行pod setup成功。

如果，依旧输出：Unable to find a pod with name, author, summary, or descriptionmatching '你的类库' 这时就需要继续下面的步骤了。

2、删除~/Library/Caches/CocoaPods目录下的search_index.json文件

终端输入：rm ~/Library/Caches/CocoaPods/search_index.json

删除成功后，再执行pod search。

3、如果search成功，但是依旧显示的老版本的类库的话。

*pod setup: 初始化（之前走过这个方法可以忽略）

*pod repo update: 更新仓库

*pod search 你的类库
