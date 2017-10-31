   
### iOS 逆向开发学习笔记
   
      
   
#### 查看静态库文件(.a 编译文件?)工具
nm 命令查看符号表信息   
otool -hv xxx.a 查看支持的架构   
lipo -info \[file]  查看支持的架构  
file \[file] 查看架构  
   
#### 查看.dylib等文件工具
使用otool -l \[file]   查看库支持的版本信息

   
     
   
#### 关于ARM架构
###### ARM的版本
ARMv7, ARM11, Cortex A8 and A4
随着时间的推移ARM架构推出了一系列不同的版本，每个版本都添加了一些新指令和改进，同时向后兼容前一个版本。第一台iPhone包含了一个实现了ARMv6的处理器，后来的设备包含了支持ARMv7的处理器。所以当你编译代码时，根据你指定目标架构版本，编译器会生成架构版本相对应的指令。汇编程序也是一样，编译器会检查所使用的指令是否包含在所指定的架构版本中。最终，你会获得针对指定架构生成的目标代码。目标文件和可执行文件实际上会被标记上他们所针对的架构，可以使用如下命令来检查目标文件所使用的架构版本。
   
###### 处理器核心和芯片系统(Soc)

然而，我们并不能说iPhone有一个"ARMv6的处理器"，因为ARMv6并不是指一个特定的处理器，只是说处理器能够运行这个架构的指令集，而并没有做任何特定的实现。用于最早的iPhone的处理器核心是通过ARM11实现的。

```
特别要注意一点，模拟器不会执行ARM代码，因为用模拟器的时候编译的是x86的代码，是用于在mac上本地执行的。
```
   
   

#### 参考资料   
[两个 Xcode 的实用工具： otool 和 install_name_tool](http://www.jianshu.com/p/193ba07dadcf)
