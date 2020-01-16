#KVO
##KVO实现机制
**当你观察一个对象时，一个新的类会动态被创建。这个类继承自该对象的原本的类，并重写了被观察属性的 setter 方法。重写的 setter 方法会负责在调用原 setter 方法之前和之后，通知所有观察对象值得更改。最后把这个对象的 isa（isa指针告诉运行时系统这个对象的类是什么） 指针只想这个新创建的子类，对象就变成了新创建的子类的实例。**

![pic](/Users/admin/Desktop/鲍琨/笔记/resource/屏幕快照 2017-09-05 上午9.47.06.png)

键值观察通知依赖于 NSObject 的两个方法：**willChangValueForKey:**和 **didChangeValueForKey:**。

在被观察属性发生改变之前,**willChangeValueForKey:**一定会被调用，会记录旧的值。而当改变发生后，**observeValueForKey:ofObject:change:context:**会被调用，继而会调用 **didChangeValueForKey:**。（手动插入这两个方法实现手动调用 KVO）



























