---
title: 记一次多次创建单例对象问题
date: 2019-03-11
category: [iOS, troubleshooting]
---

#### 单例不唯一 ?

在做 framework 开发联调时，发现一个有意思的 bug，一顿操作后，问题定位在本该是应用内唯一的单例对象创建了两次。

```objc
+ (instancetype)sharedInstance {
    static id singleton;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        singleton = [[self alloc] init];
    });
    return singleton;
}
```

WTF？如此标准的写法还能有什么问题不成？对 iOS 单例写法倒背如流的我马上就否定了这一想法 (｡･∀･)ﾉﾞ 一定是两次创建时的*环境*不同，遂从导入的这个 framework 上找原因。

整体结构是这样的

![Dependency]({{ site.url }}/assets/image/0311/dependency.png)

其中 App 层依赖于两个 framework，framework A 作为 debug 用模块，直接依赖于 framework B。问题就在于 B 中的单例被创建了两次。

看到这里，有经验的同学可能马上就能猜出原因来了：framework A 作为**动态库**被打到了 App 中。

#### 验证

为了确认这一问题，同时简化问题说明，[这里]()新建了一个 demo 来描述。其中 

- `Vendor` 作为静态库，包含唯一单例类 `VendorSingleton` 和唯一方法 `sharedInstance`，调用该方法会打印一条日志
- `HelloLogger` 作为动态库，包含一个方法 `startTesting` 来调用 `[VendorSingleton sharedInstance]`
- App 层同样作为 `Vendor` 的消费方调用该方法

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    // App 层通过动态库间接使用静态库
    [SimpleLogger startTesting];
    // App 层直接对静态库的依赖
    [VendorSingleton sharedInstance];
}
```

"预期": 单例应该只创建一次，log 只打印一条
实际: 打印了两条日志，同时带有一条警告

![Result]({{ site.url }}/assets/image/0311/result.png)

#### 解释

生成一份动态库需要经过编译与链接两步，其中链接过程中，如果一个动态库依赖了静态库，会将静态库中的内容合并到动态库中。这样与 App 内依赖的静态库为两份不同的代码。不同地方引用的单例对象，当然也就"此单例非彼单例"啦。

可以通过查看打出来的二进制包内的符号来进一步验证

![Result]({{ site.url }}/assets/image/0311/nm.png)

#### Fixup

知道了问题就好说了，iOS 中常见的动态库与静态库的创建有三种方式

- Cocoa Touch Static Library 创建 .a 静态库，与 framework 的差别为前者是纯二进制文件，后者包含了资源文件
- Mach-O 为 `Static` 的 Cocoa Touch Framework, 静态库
- Mach-O 为 `Dynamic` 的 Cocoa Touch Framework, 动态库

创建一个新的 framework 时默认 Mach-O 为 Dynamic，改成 static 即可。

问题本身比较简单、低级，不过现象确实有意思╮(╯▽╰)╭

