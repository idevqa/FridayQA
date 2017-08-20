---
title: "Friday Q&A 2014-11-07: 实现 NSZombie"
description: Zombies 是用来调试内存管理问题的有效工具。我之前讨论过 Zombies 的实现，这次我要更进一步通过代码来构造他们，这个话题的发起者是 Шпирко Алексей。
header: "Friday Q&A 2014-11-07: 实现 NSZombie"
author: 冬瓜
---

原文：[Let's Build NSZombie](https://www.mikeash.com/pyblog/friday-qa-2014-11-07-lets-build-nszombie.html)

---

Zombies 是用来调试内存管理问题的有效工具。我之前讨论过 [Zombies 的实现](https://www.mikeash.com/pyblog/friday-qa-2011-05-20-the-inner-life-of-zombies.html)，这次我要更进一步通过代码来构造他们，这个话题的发起者是 Шпирко Алексей。

## 回顾

Zombies 可以检测内存管理错误，具体来讲，当对一个被释放的 Objective-C 对象发送消息时会被检测到。这就是 "Use After Free" 问题的一个真实场景。

通常情况下，向某个对象发送消息时，该对象可能已经被覆写或是回收至 kernel。不论被复写还是已经被回收都会导致 crash。由于其他 Objective-C 对象的内存覆盖，这个消息将发送到与目的实例对象无关的个体，将会使得 selector 抛出异常，如果这个消息同名存在，则会导致结果的异常不可控。

也有可能该段内存还未被使用，仍旧是一个初始状态，即我们所说的 `dealloc` 状态。这会导致各种崩溃【注：原文是 'lead to other interesting and bizarre falures'】。举个例子，如果对象包括一个 UNIX 文件 handle【注：传统翻译为 '文件句柄'，译者任务 handle 翻译成 '句柄' 很奇怪】，也许会对文件的描述对象调用两次 `close` 方法，这可能会导致其他文件对象关闭，从而导致错误而引起 bug 情况。

ARC 在一定程度上大大减少了这些错误的发生，但是并没有完全解决。这些问题还可能是由于多线程引起、与非 ARC 协同运行、错误的方法声明或者类型滥用都会导致这些问题，这些可能性事件都会影响 ARC 存储器的变动。

Zombies hook 了对象的释放动作。使它们不进行释放操作取而代之的是将其存储，Zombie 会将对象以一个新的 Zombie 类的实例进行存储，它也会拦截所以向其发送的消息。发送到 Zombie 对象的任何消息都会诊断为错误消息而不是像以前一样，执行不可控的行为动作。另外还有一种重写类的模式，但这并不实用，因为内存通常会快速重用，所以我们可以忽略这个模式选项。

下面我们要对实现的 Zombie 类，我们需要 hook  住对象的释放状态并构建恰当的 Zombie 类。由此开始吧！

## 捕捉所有消息

如果我们的基类没有任何的方法，之后这个类的所有实例在发送消息时都会进入 runtime 的消息转发机制。这看起来像是调用 `forwardInvaocation:` 方法来截取消息。这个时机其实有点迟。在 `forwardInvacation:` 方法执行之前，runtime 需要创建一个 `NSInvocation` 对象，这说明我们消息转发的流程已经经过了 `forwardingTargetForSelector:` 的备援接受过程。这个时机也是我们捕捉消息并发送 Zombie 对象。

## 动态分配(Dynamically Allocated)类

除了被发送消息的对象，Zombie 还会记录发送者。但是在类中没有任何空间来存储消息发出者的对象引用了。如果发送者没有可扩展的实例变量，那么久没有空间用于存储。因此，消息发送类必须存储在 Zombie 类中而不是 Zombie  的对象中，这意味着需要动态分配 Zombie 类。每个 Zombie 实例都可以获得 Zombie 类。

另外一个问题是在什么地方来存储原始类的引用。虽然外面可以为其开辟额外的存储空间来解决，但是这样十分麻烦。比较容易的做法是使用类名。由于 Objective-C 不区分命名空间，所以类名在单个进程中可作为唯一标识。通过在原始类名上增加一个固定前缀来生成 Zombie 类的名称，这样我们可以将构造的 Zombie 和原始的类一一对应上。这里我们使用 `MAZombie_` 作为前缀。

## 方法实现(Method Implementations)

请注意这里所有的代码都是在非 ARC 模式下构建的，因为 ARC 会对我们的目的有一定的影响。

我们从一个简单的方法实现起步，这是一个空方法：

```ObjC
void EmptyIMP(id obj, SEL _cmd) {}
```

在 Objective-C 中，runtime 已经假定每个类都实现了 `+initialize` 方法。在将第一条消息发送至类之前将会发送各个类来执行它以完成必要的配置。如果该方法没有实现，则 runtime 将会启动转发机制，如此是我们不需要的。我们通过增加一个空的 `+initialize` 方法来避免这个问题。`EmptyIMP` 方法将会用于 Zombie 类 `+initialize` 的实现上。

```ObjC
NSMethodSignature *ZombieMethodSignatureForSelector(id obj, SEL _cmd, SEL selector) {
```

它将会检索对象的类将其名称。然后自动生成 Zombie 类：

```ObjC
	Class class = object_getClass(obj);
	NSString *className = NSStringFromClass(class);
```


可以用截取字符串前缀来获取原始类名：

```ObjC
	className = [className substringFromIndex: [@"MAZombie_" length]];
```

之后再记录错误并调用 `about()` 方法来提示：

```ObjC
	NSLog(@"Selector %@ sent to deallocated instance %p of class %@", NSStringFromSelector(selector), obj, className);
	abort();
    }
```

## 创建类

`ZombifyClass` 方法有一个正常的类并返回一个 Zombie 类，必要时将会创建：

```ObjC
Class ZombifyClass(Class class) {
```

Zombie 类的命名对于检测该 Zombie 类是有用的，如果不存在则直接创建：

```ObjC
	NSString *className = NSStringFromClass(class);
	NSString *zombieClassName = [@"MAZombie_" stringByAppendingString: className];
```

可以使用 `NSClassFromString` 来检测某个 Zombie 类是否存在。如果存在直接返回即可：

```ObjC
	Class zombieClass = NSClassFromString(zombieClassName);
	if(zombieClass) return zombieClass;
```

此处需要注意的是，这里会引起一个选择竞争的问题：如果同一个类的两个实例同时在两个线程中执行 `zombified` ，那么都将会尝试去创建 Zombie 类。这里需要为其加锁，来解决这个问题。

之后我们调用 `objc_allocateClassPair` 方法来为 Zombie 类分配空间：

```ObjC
	zombieClass = objc_allocateClassPair(nil, [zombieClassName UTF8String], 0);
```

我们来增加 `-methodSignatureForSelector:` 的实现，通过调用 `class_addMethod` 方法。`@@::` 这个符号表示它会返回一个对象，并且要传入三个参数：一个对象（`self`），一个选择子（`_cmd`），一个函数指针（这是一个明确的函数参数），

```ObjC
	class_addMethod(zombieClass, @selector(methodSignatureForSelector:), (IMP)ZombieMethodSignatureForSelector, "@@::");
```

 `+intialize` 也会加入方法列表中。原生库中没有添加类方法的函数，我们通过向其添加一个 metaclass 来解决这个问题：
 
```ObjC
	class_addMethod(object_getClass(zombieClass), @selector(initialize), (IMP)EmptyIMP, "v@:");
```

现在该类已经构建完成，我们在 runtime 中对其注册即可。

```ObjC
	objc_registerClassPair(zombieClass);
	return zombieClass;
}
```

## 对于对象的 Zombie 化方案

为了将对象编程指定的 Zombie 模式，我们替换了 `NSObject` 的 `dealloc` 方法。子类的 `dealloc` 方法仍旧可以运行，一旦与 `NSObject` 产生继承关联就会执行 Zombie 代码。这样可以防止对象被破坏，而且提供了生成 Zombie 类的绝佳时机。这个操作我们封装成一个方法，来决策对象是否启动 Zombie 化：

```ObjC
void EnableZombies(void) {
   Method m = class_getInstanceMethod([NSObject class], @selector(dealloc));
   method_setImplementation(m, (IMP)ZombieDealloc);
}
```

## 测试

我们来确保其能正常执行：

```ObjC
	obj = [[NSIndexSet alloc] init];
	[obj release];
	[obj count];
```

我使用 `NSIndexSet` 来进行测试，其好处是我们不用考虑使用 Core Foundation 的桥接问题。运行这段代码，Zombie 将会输出日志：

```ObjC
a.out[5796:527741] Selector count sent to deallocated instance 0x100111240 of class NSIndexSet
```

## 结论

Zombie 是十分易实现的。通过对类的动态分配，可以轻松的跟做原始类，并不依赖于 Zombie 中来存储。`methodSignatureForSelector: ` 方法为拦截发送到 Zombie 对象的消息提供了一个简单的阻塞时机。通过在 `-[NSObject dealloc]`构造一个简单的 hook 从而将对象转换成对应的 Zombie 类，而不是在引用计数为0的时候直接销毁它们。
