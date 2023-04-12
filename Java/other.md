# 1 常量池
基本类型的包装类的大部分都实现了常量池技术（java类），Byte,Short,Integer,Long,Character,Boolean，另外两种**浮点**数类型的包装类则没有实现。**-128~127**
# 2 SPI(Service Provider Interface)
[Java SPI机制详解](https://juejin.cn/post/6844903605695152142)
> 简单的总结下 java SPI 机制的思想。我们系统里抽象的各个模块，往往有很多不同的实现方案，比如日志模块的方案，xml解析模块、jdbc模块的方案等。面向的对象的设计里，我们一般推荐模块之间基于接口编程，模块之间不对实现类进行硬编码。一旦代码里涉及具体的实现类，就违反了可拔插的原则，如果需要替换一种实现，就需要修改代码。为了实现在模块装配的时候能不在程序里动态指明，这就需要一种服务发现机制。
java SPI 就是提供这样的一个机制：为某个接口寻找服务实现的机制。有点类似IOC的思想，就是将装配的控制权移到程序之外，在模块化设计中这个机制尤其重要。
- 在ClassPath路径下的 META-INF/services 文件夹添加一个文件：文件名是**接口**的全限定类名，内容是**实现类**的全限定类名，多个实现类用换行符分隔
- 使用
    ```java
    ServiceLoader<SpiService> load = ServiceLoader.load(SpiService.class);
    Iterator<SpiService> iterator = load.iterator();
    while (iterator.hasNext()){
        SpiService service = iterator.next();
        service.println();
    }
    ```