# +load VS +initialize

翻译原文地址：https://medium.com/@kostiakoval/load-vs-initialize-a1b3dc7ad6eb

### load和initialize方法有什么区别呢?
已经有很多优秀的文章讨论过这个话题了， 比如Mike Ash 写的[这一篇文章](https://www.mikeash.com/pyblog/friday-qa-2009-05-22-objective-c-class-loading-and-initialization.html)。我就不对细节做过多描述了，来看看它们的使用案例吧。

## 共同点
父类总是比它们的子类先收到调用消息。

### +initialize
1、每个类只调用一次。
2、如果不去调用这个类，那它就不会被调用。
3、+initialize方法可能会被调用多次。
例如如果一个子类没有实现initialize方法，那么父类的initialize会被调用两次。
4、第一次调用这个类的时候，它会被调用。
5、调用它是线程安全的。
6、父类总是比它们的子类先收到调用消息。

### +load
1、在app启动以后，一个类或者Category被加入到Objective-C runtime的时候会去调用。
2、即使你不调用一个类的任何方法，也会被调用。
3、一个类或者Category的load方法只会被系统调用一次（如果一个Category实现了+load方法）。
4、Framework里的类在这时候已经加载完成了。
5、C++ static initializers 这时候还没有加载。
6、在main方法前边被调用。
7、在这里使用autorelease对象是安全的，因为ARC会自动创建autorelease pool 。

## 用例
### +load
* 方法调用的非常早，可以对类进行配置。
* Swizzle方法
``` 
+ (void)load
{
    NSLog(@"Load KKObject");
    [KKObject jr_swizzleMethod:@selector(description) withMethod:@selector(kkk_description) error:nil];
}
```
> Note：在load方法中也建议使用dispatch_once之类的方式来保障这里的内容只被执行一次，因为load方法虽然系统只调用一次，但是也可能被手动调用。

### initialize
* 不要在+initialize方法里做太重的工作。
* 总是需要检查+initialize是否是被当前类调用。
```
+ (void)initialize
{
    if (self == [KKObject self]) {
        // ... do the initialization ...
        k_objectName = @"a Name";
    }
}
```
因为如果子类没有实现+initialize方法，父类的+initialize方法会被调用。
* 初始化类的静态变量
使用dispatch_once来初始化，是个好方法。
如果你想实现“懒加载”方式，创建一个property方法并且总是通过这property方法来访问这个变量。使用dispatch_once来实现这个property的访问。
```
- (NSString *)localName
{
    static NSString *k_localName;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        k_localName = @"Cool local name";
    });
    return k_localName;
}
```
