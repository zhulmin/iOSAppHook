   
### iOS 逆向开发学习笔记
   
   
#### 查看静态库文件(.a 编译文件?)工具
nm 命令查看符号表信息   
otool -hv xxx.a 查看支持的架构   
otool -l [file] 查看app是否被砸壳过 cryptid 0(已经砸壳) 1(未砸壳)
lipo -info \[file]  查看支持的架构  
file \[file] 查看架构  
   
#### 查看.dylib等文件工具
使用otool -l \[file]   查看库支持的版本信息


### 开始逆向  
砸壳：必须使用脱壳的ipa， 脱壳的ipa不能是从App Store下载的， 必须是用越狱手机砸壳，或者被人砸好的（xx助手等第三方平台获取😆）  
签名: 使用[ios-app-signer](https://github.com/DanTheMan827/ios-app-signer)或者Cydia大神的impactor
  
使用Theos制作插件 (也可以使用基于Theos的可视化Xcode插件iOSOpenDev)   
  
先通过Theos制作.dylib文件, 网上很多资料  
然后把这个target导入到app的二进制文件  
  
通过 otool -L命令查看生成的.dylib文件  
结果中的一段/Library/Frameworks/CydiaSubstrate.framework/CydiaSubstrate (offset 24)  
可以看到这里还有对CydiaSubstrate的依赖，这是不行的 , 这个是theos在越狱机上特有的, 在非越狱机上需要更改此依赖  
修改依赖，将libsubstrate.dylib文件(该文件应该在/opt/thoes/lib/目录下),拷贝到与你生成的的.dylib一个目录下,通过下面的指令修改依赖,  
```
cd /Users/rain/Desktop/appName/Project/targetName/.theos/obj/debug 
```
```
install_name_tool -change /Library/Frameworks/CydiaSubstrate.framework/CydiaSubstrate @loader_path/libsubstrate.dylib targetName.dylib
```  
然后重新查看targetName.dylib，会发现依赖已经修改成@loader_path/libsubstrate.dylib (offset 24)  
重新签名自制的targetName.dylib 和libsubstrate.dylib(很重要)  
  
我们需要把生成的dylib和libsubstrate.dylib文件copy到appName.app中,然后用codesign开始签名  
```
codesign -f -s 'iPhone Developer: zhulmin1458@gmail.com (29H47J82NP)' (自己的证书名, 钥匙串里复制信息)  
```
添加可执行文件的依赖动态注入  
  
此处用到是insert_dylib(也可以使用[yololib](https://github.com/KJCracks/yololib))，先从gitHUb下载insert_dylib，编译后将 insert_dylib 放到/usr/local/bin目录中（不放到此目录中需要使用./insert_dylib，放在目录中后只需要使用insert_dylib）
再将自制的dylib和libsubstrate.dylib拷贝到appName.app包中，执行以下命令:  
```
cd /Users/rain/Desktop/appName/appName/Payload/appName.app/
insert_dylib @executable_path/xxx.dylib appName
```
以下为执行步骤  
```
localhost:debug apple$ insert_dylib @executable_path/targetName.dylib /Users/rain/Desktop/appName/appName.app/appName
Binary is a fat binary with 2 archs.
LC_CODE_SIGNATURE load command found. Remove it? [y/n] n
LC_CODE_SIGNATURE load command found. Remove it? [y/n] n
Added LC_LOAD_DYLIB to all archs in /Users/rain/Desktop/appName/appName.app/appName_patched
```
会生成一个appName_patched 这个就是修改了依赖关系的二进制文件，  

注意替换  
注意: 别忘了之前将 自制的.dylib libsubstrate.dylib 拷贝进appName.app  
如果appName_patched在appName.app中，还要将appName_patched拷贝进appName.app中 替换原来的appName, 把appName_patched的名字改回来appName   
  
  
  
  
   
#### 关于ARM架构
###### ARM的版本
ARMv7, ARM11, Cortex A8 and A4
随着时间的推移ARM架构推出了一系列不同的版本，每个版本都添加了一些新指令和改进，同时向后兼容前一个版本。第一台iPhone包含了一个实现了ARMv6的处理器，后来的设备包含了支持ARMv7的处理器。所以当你编译代码时，根据你指定目标架构版本，编译器会生成架构版本相对应的指令。汇编程序也是一样，编译器会检查所使用的指令是否包含在所指定的架构版本中。最终，你会获得针对指定架构生成的目标代码。目标文件和可执行文件实际上会被标记上他们所针对的架构，可以使用如下命令来检查目标文件所使用的架构版本。
   
###### 处理器核心和芯片系统(Soc)

然而，我们并不能说iPhone有一个"ARMv6的处理器"，因为ARMv6并不是指一个特定的处理器，只是说处理器能够运行这个架构的指令集，而并没有做任何特定的实现。用于最早的iPhone的处理器核心是通过ARM11实现的。

```
特别要注意一点，模拟器不会执行ARM代码，因为用模拟器的时候编译的是x86的代码，是用于在mac上本地执行的。
```
   
### iOS 安全  
  
```
###### Q: 证书绑定（pinning certificates）能否防止中间人攻击？

Conrad：证书绑定是一个用来防止中间人攻击的方法，在实践中确实很有用，前提是你的手机未越狱。比如 Twitter 就在用证书绑定技术。然而，用逆向工具 Cycript 能够轻松破解。AFNetworking 支持证书绑定，只要设置一下 SSLPinningMode 这个属性。然而… 我们可以用 Cycript，再把它修改成 none，如果你的 iPhone 越狱了，或者被人控制了。所以，并没有一种方案能够彻底防止你的流量被检测，如果没有越狱，倒是一种很好的保护方法。

###### Q: 是否推荐类似 cocoapods-keys 这样的工具来混淆字符串？

Conrad: 字符串混淆有以下的好处：

－ 如果你用苹果框架里的私有 API，能很容易的躲避过苹果审查的自动扫描工具。 － 如果那个人并没有一部越狱的 iPhone，或者它不知道如何解析混淆，他可能很难找到 app 里的字符串。

然而，你无法永远的把字符串藏起来。比如，在 Cycript 中，你可以轻易地解开混淆。所以字符串混淆也不是没法破的保护方案。

###### Q: class-dump 和 dumpdecryped 的区别是啥？

Conrad: 他们事实上是不同的工具。dumpecrypted 能将一个 App Store 加密的工具解密。而 class-dump 能将一个解密后得二进制文件去除他的 Objective-C 接口文件，跟 IDA 有点像，不过很遗憾的是：它不支持 Swift。

###### Q: 导出 Apple framework 的库，比如：this one，是不是也是用的 class-dump？

Conrad: 是的，苹果的框架没有加密，可以被轻易地导出所有的class。这意味着那些把苹果的私有interface 放到GitHub上的，你可以轻易地搞定他们。

###### Q: 逆向工程是否存在法律问题？Lyft 会不会不同意调用他们的私有代码和 api？

Conrad: 从法律上讲，我觉得没有太多问题。我们通常会跟所有的合作伙伴去聊到我们在做的事情。比如：我刚刚发现的这些东西我都会和 Lyft 讨论一下，在他们允许后集成到 Workflow 里。这些技术其实也有很多道德上的考虑。不过话说回来，这些技术也能阻止开发者们胡乱搞，就比如之前逆向 Twitter 发现他们上传用户手机里装的 App 列表到他们的服务器上。所以凡事总有两面，逆向工程只是个工具，善用就好了。

###### Q: 如何查看苹果框架的反汇编代码？

Conrad: 这个的过程跟我之前展示的一个很像，只不过你不再需要一个越狱后的设备。当你把线插到设备上的时候，iTunes 实际上获取到了设备的符号，你可以在 Xcode 里找到一个大概叫 “device symbol” 文件夹，在 IDA 或者 class-dump 里打开 Apple 的 frameworks，然后展开分析。这些都是没加密的，而且可以直接拿来用。在修复一些 beta 版本的 UIKit 的 bug 的时候很好使。
```

  
  
#### 遇到的问题
  
  
theos 执行make package时遇到的问题
```
dpkg-deb: error: obsolete compression type 'lzma'; use xz instead
```
找到该文件:   
```
/opt/theos/makefiles/package/deb.mk  
```
第六行  
```
_THEOS_PLATFORM_DPKG_DEB_COMPRESSION ?= lzma
改为
_THEOS_PLATFORM_DPKG_DEB_COMPRESSION ?= xz
```
  
  
使用ios-app-signer签名成功, 使用xcode安装时遇到的问题
```
"This app contains an app extension with an illegal bundle identifier. App extension bundle identifiers must have a prefix consisting of their containing application's bundle identifier followed by a '.'."
```
```
"This app contains an app extension with an illegal bundle identifier. App extension bundle identifiers must have a prefix consisting of their containing application&#39;s bundle identifier followed by a &#39;.&#39;."
```
这是因为 target的bundle id 必须包含app 的bundle id, 作为前缀 (target是theos做的tweak包xxx.dylib)  
比如app bundle id 是com.xxx.xx, target 的id应该是com.xxx.xx.???  
  
  



#### 参考资料   
[logos语法](http://iphonedevwiki.net/index.php/Logos)  
[两个 Xcode 的实用工具： otool 和 install_name_tool](http://www.jianshu.com/p/193ba07dadcf)  
[非越狱theos的Tweak创建的dylib安装到iOS设备](http://cdn2.jianshu.io/p/5d353d6db145)  
[iOS 善意破解简书APP(非越狱篇)实现一键点赞](http://www.jianshu.com/p/ab8d6db22e0f)  

[Urinx/iOSAppHook](https://github.com/Urinx/iOSAppHook#%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84CaptainHook%E8%BD%BD%E5%85%A5Cycript)  
##### iOS 安全  
[iOS App 的逆向工程: Hacking on Lyft](https://academy.realm.io/cn/posts/conrad-kramer-reverse-engineering-ios-apps-lyft/)  


###### 已经越狱从砸壳开始  
[ios微信逆向实战--自动抢红包、修改步数、防止消息撤回](http://www.jianshu.com/p/ec0a682e6d83)  
  
  
