   
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
   
   
  
  
