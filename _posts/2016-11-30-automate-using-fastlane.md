---
title: 使用 fastlane 自动化 iOS 部署
author: 孙耀珠
tags: 运维
---

对于 iOS 开发者来说，应用发布和代码签名证书大概是最令人头疼的两个环节了，这倒不是因为技术上有多难，而是它们的操作流程相当麻烦，尤其是在中国的网络环境下。

一般来讲，手动发布应用更新大致有以下流程：修改所有 Target 的版本号、用 Xcode 给项目 Archive、在 Xcode Organizer 中上传到 App Store、到 iTunes Connect 更新相关信息、提交给苹果审核。而其中上传那一步在不翻墙的情况下成功率极低，经常会卡在「Authenticating with the iTunes Store…」，而且系统 SOCKS 代理（如 Shadowsocks）在此时似乎并不起作用，只有使用 Proxifier 或者 VPN 才有效果。也是基于这个原因，我一般不会直接在 Organizer 中直接上传，而是先导出为 .ipa 文件，再使用 Xcode 附带的 Application Loader 上传，这样就免去了上传失败的话每次直接上传时将 .xcarchive 转为 .ipa 的时间。

当然以上还没考虑第一次发布时配置证书的流程，一个初学者面对苹果开发者中心琳琅满目的 Certificates / Identifiers / Provisioning Profiles 多半是一脸懵逼，不过幸运的是从 Xcode 8 开始已经能够比较完美地自动管理代码签名了，不再像以前一样需要自己去 Fix issues。

对于 iOS 应用的部署，如果你也像我一样饱受折磨，fastlane 也许是你的救星。

<!--more-->

## fastlane

[![](https://raw.githubusercontent.com/fastlane/fastlane/master/fastlane/assets/fastlane_text.png)](https://fastlane.tools)

简而言之，fastlane 是一套用 Ruby 编写的 iOS 命令行工具集（后来也支持了 Android），主要组件包括：

- `match` / `cert` / `sigh` 协助管理代码签名
- `pem` 自动生成 APNs 证书
- `scan` 自动化测试
- `gym` 自动化编译并打包生成签名的 .ipa 文件
- `snapshot` / `frameit` 协助处理 iOS 屏幕快照
- `pilot` 上传和管理 TestFlight
- `deliver` 将应用及其它信息上传到 App Store


而通过 fastlane 我们可以将上面这些独立的小工具有机地结合起来，从管理证书到单元测试，从编译打包到上传发布，都能在命令行轻松完成，乃至一键部署，听起来是不是生产力大增呢。

因为 macOS 自带了 Ruby，所以 fastlane 的安装非常简单，`gem install fastlane` 就搞定了，初始化也只需要在工程目录下 `fastlane init` 即可。初始化成功后你会得到 `Fastfile`、`Appfile` 和 `Deliverfile`，前者是最核心的流程配置文件，后两者则分别用于访问 Apple Developer Portal 和 iTunes Connect，另外在 `metadata` 目录下会自动拉取 App Store 的相关信息，以供日后 `deliver` 使用。

顺带一提，Terminal 也不读取系统代理设置，而是通过环境变量来设置代理的。譬如对于 ShadowsocksX-NG 的默认设置，`export ALL_PROXY="socks5://127.0.0.1:1086"` 即可。

## Fastfile

接下来最核心的部分就是来配置 `Fastfile` 了，虽然自动生成的 `Fastfile` 已经囊括了 `test` / `beta` / `release` 这三支流程（在这里称为 lane），不过它们的默认操作只是简单的 `scan`  / `gym` + `pilot` / `gym` + `deliver`，尚未发挥出 fastlane 的全部威力。fastlane 除了上述的子工具之外，还内建了非常多实用的操作，在 <https://docs.fastlane.tools/actions/> 可以查询 fastlane 所有支持的操作和已有的插件。因为 `Fastfile` 本质上就是 Ruby DSL，所以你也可以直接在其中写 Ruby 代码，甚至为其创建插件。

以我自己的项目（QSCMobileV3）为例，在常规的编译打包和上传之前，我需要用 `agvtool` 增加 Build Number 并修改版本号，`git commit` 之后再打上包含更新日志的标签并推送到远程仓库。这一整套流程实际上都可以由 fastlane 代劳，而我需要做的只是预先书写一下 `release_notes.txt`，接着敲下 `fastlane release` 便能一键部署，剩下就是静静等待苹果审核了。

```ruby
platform :ios do
  before_all do
    increment_build_number
    increment_version_number(
      bump_type: prompt(text: "Bump type: (patch/minor/major)")
    )
    
    git_commit(
      path: ".",
      message: "Version bump to #{lane_context[:VERSION_NUMBER]}"
    )
    lane_context[:CHANGE_LOG] = File.read "./metadata/zh-Hans/release_notes.txt"
    add_git_tag(
      tag: lane_context[:VERSION_NUMBER],
      message: lane_context[:CHANGE_LOG]
    )
    push_to_git_remote
    
    gym(
      scheme: "QSCMobileV3",
      include_bitcode: true
    )
  end
  
  desc "Submit a new Beta Build to Apple TestFlight"
  lane :beta do
    pilot(
      changelog: lane_context[:CHANGE_LOG],
      distribute_external: true
    )
  end
  
  desc "Deploy a new version to the App Store"
  lane :release do
    deliver(
      force: true,
      submit_for_review: true
    )
  end
end
```

## boarding

[![](https://raw.githubusercontent.com/fastlane/boarding/master/assets/BoardingHuge.png)](https://github.com/fastlane/boarding)

除了命令行工具之外，fastlane 还提供了一个自动建站工具 boarding，通过这个网站用户可以一键注册成为 TestFlight 外部测试员。也许你已经猜到了，boarding 用的是 Ruby on Rails 框架，你所需要做的就是将网站部署到 Heroku 上，填写好模板信息，就可以通过 xxx.herokuapp.com 来访问它了。

之前在求是潮手机站 V3 公测时，是通过回复微信公众号来申请的，需要先在后台将用户数据处理为 CSV，然而再定期由我手动添加到 iTunes Connect，这相当不优雅。因为接下来还会推出 Today Widget / Box / Discover 等新功能，所以我现在用 boarding 搭建了「[求是潮手机站 iOS 测试体验计划](https://ios.zjuqsc.com)」，完全不需要手动干预，是不是优雅多了呢？
