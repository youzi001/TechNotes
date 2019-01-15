# iOS启动时间优化逻辑整理

本次只讨论程序冷启动逻辑

####APP冷启动共分三个阶段		
#####一、main()之前 简称pr-main阶段
iOS10之后通过Xcode8可以通过以下方式查看并打印启动耗时：

Product --> Scheme -->  Edit Scheme --> Run --> Arguments --> Environment Variables

* 通过添加环境变量可以打印出APP的启动时间分析			
1、``DYLD_PRINT_STATISTICS``设置为 1 	
2、如果需要更详细的信息，那就将``DYLD_PRINT_STATISTICS_DETAILS``设置为 1

APP
  
```
Total pre-main time: 911.53 milliseconds (100.0%)  
	dylib loading time: 167.58 milliseconds (18.3%)  (动态库加载)
	rebase/binding time:  26.01 milliseconds (2.8%)  (修正、重新绑定程序指针)
	ObjC setup time: 142.80 milliseconds (15.6%)     (runtime启动加载)
	initializer time: 575.05 milliseconds (63.0%)    (类的初始化才做 C++类相关操作)
```

#####pr-main阶段系统做的事  
**dyld**  
Apple的动态链接器，可以用来装载Mach-O文件（可执行文件，动态库等）  
启动APP时，dyld所做的事情有  
1。装载APP的可执行文件，同时会递归加载所有依赖的动态库  
2.当dyld把可执行文件，动态库都装载完毕后，会通知运行时进行下一步的处理  

**runtime**  
启动APP时，runtime所做的事情有		
1.调用``map_images``进行可执行文件内容的解析和处理  
2.在``load_images``中调用``call_load_methods``，调用所有类和类的+加载方法  
3.进行各种objc结构的初始化（注册Objc类，初始化类对象等等）  
4.调用C++静态初始化器和属性（（构造函数））修饰的函数  
到此为止，可执行文件和动态库中所有的符号（Class，Protocol，Selector，IMP，...）都已经按格式成功加载到内存中，被runtime管理

**main**   
1.APP的启动由dyld的主导，将可执行文件加载到内存，顺便加载所有依赖的动态库  
2.并由运行时负责加载成objc定义的结构  
3.所有初始化工作结束后，使dyld就会调用主函数  
4.接下来就是UIApplication Main函数 

####由此可以看出我们能做的优化有：  
* 移除不需要用到的动态库  
* 移除不需要用到的类, 减少selector、category的数量  
* 尽量避免在+load方法里执行的操作，可以推迟到+initialize方法中。  

#####二、main() 到 APPDelegate ``application:didFinishLaunchingWithOptions:``方法执行之前

这个阶段我们能做的不多，但有一点值得注意  
如果你info.Plist文件里面设置了启动如果为Main  
程序会在这个阶段加载Main.storyBoard里面的控制器如控制为空则会增加 50ms--100ms 的启动时间   
如控制器不为空且控制器``awakeFromNib`` 里面代码较多，则会较大情况下影响到启动的时间 

####由此可以看出我们能做的优化有：  
* 代码初始化控制器并调整控制器初始化时机 

#####三、APPDelegate ``application:didFinishLaunchingWithOptions:``的执行过程

这个过程也是代码优化比较明显的地方，不同的启动函数对代码的影响是不同的，我们应该对启动代码进行分析分级、操作延迟等操作，部分代码做到按需加载。

####由此可以看出我们能做的优化有：
* 进行代码分级分时  
1、日志、统计等必须在 APP 一启动就最先配置的事件  
2、项目配置、环境配置、用户信息的初始化 、推送、IM等事件  
3、其他 SDK 和配置事件  

