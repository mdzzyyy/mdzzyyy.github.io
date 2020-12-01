# Effective Objective-c 2.0

## 理解 objc_msgSend 的使用

由于 Objective-C 是 C 的超级，C 语言使用 "静态绑定" 也就是说，在编译器就能决定运行时所调用的函数。

```
#import <stdio.h>

void printHello() {
	printf("Hello, world!\n");
}

void printGodbye() {
	printf("Godbye, world!\n");
}

void doTheThing(int type) {
	if (type == 0) {
		printHello();
	} else {
		printGodbye();
	}
	return 0;
}

```

如果不考虑 "内联"，那么编译器在编译代码的时候就已经知道程序中有 printHello 与 printGodbye 两个函数，于是会直接生成调用这些函数的指令。而函数地地址实际上是硬编码在指令之中的。

```
#import <stdio.h>

void printHello() {
	printf("Hello, world!\n");
}

void printGodbye() {
	printf("Godbye, world!\n");
}

void doTheThing(int type) {
	void (*fnc)();
	if (type == 0) {
		fnc = printHHello;
	} else {
		fnc = printGodbye;
	}
	fnc();
	return 0;
}

```

这时就得使用 "动态绑定" 了，因为所要调用的函数直接到运行期才能确定。

在 Objective-C 中，想某对象传递消息，那就会使用动态绑定机制来决定需要调用的方法。在底层，所有方法都是普通的 C 语言函数，然而对象收到消息后，究竟调哪个方法则完全于运行期决定。

编译器看到 "消息" 之后，将其转换为一条标准的 C 语言函数调用，所调用的函数乃是消息传递机制中的核心函数，叫做objc_msgSend。

```
void objc_msgSend(id self, SEL cmd, ...)

第一参数代表接收者，第二个参数代表选择器，后续参数就是消息中的那些参数。

id returnValue = objc_msgSend(someObject, @selector(messageName:),parameter)
```
objc_msgSend 函数会根据接收者与选择器的类型来调用适当的方法。为此需要在接受者所属的类中搜索其 "方法列表"，找到选择器相符的方法，就会跳至实现代码。若找不到，则就沿着继承体系继续向上查找，等找到合适的方法之后在跳转。

同时 objc_msgSend 会将匹配结果缓存在 "快速映射表" 里面，每个类都会有一块缓存，用来缓存使用过的方法，用于之后发送相同的消息，可以迅速执行起来。


### 要点
1. 消息由接收者、选择器以及参数构成。给某对象 "发送消息"，也就相当于在该对象上 "调用方法"。
2. 发给某对象的全部消息都有由 "动态消息派发系统" 来处理，该系统会查出对应的方法，并执行其代码。








