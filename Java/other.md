# 1 常量池
基本类型的包装类的大部分都实现了常量池技术，Byte,Short,Integer,Long,Character,Boolean，另外两种**浮点**数类型的包装类则没有实现。**-128~127**
# 2 SPI(Service Provider Interface)
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