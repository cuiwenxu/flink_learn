# 用idea读代码时的一个技巧

用idea看代码时的一点困惑：

点开一个方法的代码想看方法的具体实现时，发现跳转到最基础的接口或者父类中，没有具体的函数体，此时点ctrl+h也仅弹出一级子类或者接口，为了查看具体是调用的哪个类中的方法，要用idea debug模式下的step into，仅往前执行一步，此时就可以看到具体调用的哪个类中的方法了。

![idea read code](idea_read_code.png)

