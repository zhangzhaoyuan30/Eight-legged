<!-- TOC -->

- [1 case class?](#1-case-class)
- [2模式匹配？](#2模式匹配)

<!-- /TOC -->
# 1 case class?
当一个类被声名为case class的时候，scala会帮助我们做下面几件事情：  
1. **构造器中的参数**默认为**val类型**
2. 自动创建伴生对象，同时在里面给我们实现了**apply**方法，使得我们在使用的时候可以不直接显示地new对象
3. 伴生对象中同样会帮我们实现unapply方法，从而可以将case class应用于模式匹配
4. 实现自己的toString、hashCode、copy、equals方法，并实现序列化接口，除此之此，case class与其它普通的scala类没有区别
# 2模式匹配？
[模式匹配与匿名函数](https://windor.gitbooks.io/beginners-guide-to-scala/content/chp4-pattern-matching-anonymous-functions.html)