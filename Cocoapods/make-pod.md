
## podspec
Cocoapods使用.podspec配置文件存储三方库的信息。一个.podspec的内容大致如下

```
Pod::Spec.new do |s|
  s.name         = "URWKWebViewController"
  s.version      = "1.2.8.3"
  s.summary      = "URWKWebViewController"
  s.description  = <<-DESC
                   DESC
  s.homepage     = "http://192.168.0.116/monkey/iOS-URWKWebViewController"
  s.license      = {
    :type => 'CpoyRight',
    :text => 'LICENSE  ©2016 ZhuoJian, Inc. All rights reserved'
 }
  s.author             = { "ChenYong" => "chenyong@zhuojianchina.com" }
  s.platform     = :ios, "8.0"

  s.source       = { :git => "http://192.168.0.116/monkey/iOS-URWKWebViewController.git", :tag => s.version }
  s.libraries = "z", "c++"
  s.source_files = 'URWKWebViewController/URWKWebViewController/**/*.{h,m}'
  s.resources = 'URWKWebViewController/URWKWebViewController/**/*.{xcassets,txt,xib}'
  s.requires_arc = true
  s.dependency 'SDWebImage', '~> 4.0.0'
  s.dependency 'iOS-URAlert', '~> 2.0.1'
  s.dependency 'iOS-URLoadIndicator', '~> 2.0.1'
  s.dependency 'iOS-URBaseMacro', '~> 2.1.1'
  s.dependency 'iOS-URNetworking', '~> 1.0.2'
  s.dependency 'URAliPay', '~> 1.0.4'
  s.dependency 'WeChatSDKPods', '~> 1.6.1'
end
```

还可以添加子模块：

```
s.subspec 'SubProject' do |ss|  
  ss.source_files = 'AFNetworking/AFSecurityPolicy.{h,m}'
  ss.public_header_files = 'AFNetworking/AFSecurityPolicy.h'
  ss.frameworks = 'Security'
end  
```

## Cocoapods make pod
开始提交项目到Cocoapods之前，需要一定的身份表示这个项目是你的。Cocoapods提供了trunk。

```
pod trunk register abc@xxx.com 'C Y' --description='My own computer'  
```

只要运行上面命令则会像Cocoapods方面注册一个账号。不过他的账号没有类似登陆的机制，所以在你切换设备后，需要再次使用这个命令进行“登陆”操作。其语法为pod trunk register 邮箱 '昵称' --description='设备信息'，其中的昵称和--description是可有可无的。对于昵称来说，第一次"注册"时最好填写一下。第二次"登陆"时，则可不比填写，而对于--description来说的话，其实就是为了让你区分不同设备"登陆"，因为Cocoapods是以Session的形式将用户信息缓存在机子中，而可能出现多台设备"登陆"的情况，所以在register的时候带上--description方面之后查看哪些设备“登陆”过. 只要只要通过pod trunk me来查看是否"注册"成功。 

#### 已有项目
如果你已有项目，可以在项目路径下输入：

```
pod spec create 'xxx'  
```

Cocoapods会自动帮你产生一个包含一部分基础内容的Podspec文件


#### 新建项目

```
pod lib create 'xxx'
```

Cocoapods会创建好一个Cocoapdos定义好的一个项目模板。


#### 校验

发布的项目是需要先校验是否合格的，使用下面的命令：

```
pod lib lint xxx.podspec 
```

常用的一些配置：
1）--allow-warnings
是否允许警告，在用到第三方框架的时候，有的时候是自带会有warmings的代码，用这参数可以屏蔽警告

2）--fail-fast
在出现第一个错误的时候就停止校验

3）--use-libraries 
如果用到的第三方中需要使用库文件的话，则会用到这个参数

4）--sources
如果一个库的podspec包含了除了Cocoapods仓库以外的其他库的引用，则需要该参数指明，用逗号分隔

#### git提交，并打tag

#### 上传

```
pod trunk push name.podspec
```


## 私有pod

私有pod在开发阶段，不建议制作成pod。可以先用路径引用的方式测试：

```
pod 'xxx', :path => '../xxx'  
```

#### 1.创建仓库

```
pod repo add '仓库名' '仓库地址'  
```

#### 2.podfile中添加私有源地址

```
# Podfile文件
# 公有仓库
source 'https://github.com/CocoaPods/Specs.git'  
# 私有仓库
source 'xxx'
```

#### 3.校验

```
pod lib lint 项目名.podspec --sources=https://github.com/CocoaPods/Specs.git,xxx
```

#### 4.上传

```
pod repo push --source=https://github.com/CocoaPods/Specs.git,xxx
```

 








