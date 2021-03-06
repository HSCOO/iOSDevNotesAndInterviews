# 2018-4月份iOS面试经历

> 作者：Xcode_boy
> 链接：https://juejin.im/post/5adaed6a518825673123c757

## 算法题

### 请在1000万个整型数据中以最快的速度找出其中最大的1000个数？

- 这是一个经常被问到的问题，百度网上解法也很多。这里仅提供基本思路，供参考：
	- 把1000万的整型平均分到合适n个文件中，分别对每一份文件找出前1000个最大的数，最后对每份文件前1000数据用常规算法合并即可。
	- 那么，如何从每一份文件中找出前1000个最大的数呢？
	- **先取文件中前1000个数放到数组中，并排好序（假设升序），之后从文件中读取下一个数与数组第一个数比较，如果比数组中第一个数大，则替换数组第一个数，并重新排序，之后再取下一个数进行下轮比较即可。**

### 循环链表题：一个有序循的整形环链表断开了，请插入一个整形数，使得链表仍然是有序的。
- 解题思路：请百度……哈哈


### Block中可以修改全局变量，全局静态变量，局部静态变量吗？
- 可以.[深入研究Block捕获外部变量和__block实现原理](https://www.jianshu.com/p/ee9756f3d5f6)
	- 全局变量和静态全局变量的值改变，以及它们被Block捕获进去，因为是全局的，作用域很广
	- 静态变量和自动变量，被Block从外面捕获进来，成为__main_block_impl_0这个结构体的成员变量
	- 自动变量是以值传递方式传递到Block的构造函数里面去的。Block只捕获Block中会用到的变量。由于只捕获了自动变量的值，并非内存地址，所以Block内部不能改变自动变量的值。
	- Block捕获的外部变量可以**改变值的是静态变量，静态全局变量，全局变量**
	- Block就分为以下3种
		- _NSConcreteStackBlock:只用到外部局部变量、成员属性变量，且没有强指针引用的block都是StackBlock。
StackBlock的生命周期由系统控制的，一旦返回之后，就被系统销毁了,是不持有对象的
			- _NSConcreteStackBlock所属的变量域一旦结束，那么该Block就会被销毁。在ARC环境下，编译器会自动的判断，把Block自动的从栈copy到堆。比如当Block作为函数返回值的时候，肯定会copy到堆上
		- _NSConcreteMallocBlock:有强指针引用或copy修饰的成员属性引用的block会被复制一份到堆中成为MallocBlock，没有强指针引用即销毁，生命周期由程序员控制,是持有对象的
		- _NSConcreteGlobalBlock:没有用到外界变量或只用到全局变量、静态变量的block为_NSConcreteGlobalBlock，生命周期从创建到应用程序结束,也不持有对象

		
- ARC环境下，一旦Block赋值就会触发copy，__block就会copy到堆上，Block也是__NSMallocBlock。ARC环境下也是存在__NSStackBlock的时候，这种情况下，__block就在栈上
- ARC下，Block中引用id类型的数据有没有__block都一样都是retain，而对于基础变量而言，没有的话无法修改变量值，有的话就是修改其结构体令其内部的forwarding指针指向拷贝后的地址达到值的修改


###  代码输出 ?

```objc
@property (nonatomic, strong) NSString *strongString;
@property (nonatomic, weak)   NSString *weakString;

strongString =  [NSString stringWithFormat:@"%@",@"string1"];
weakString =  strongString;
strongString = nil;

NSLog(@"%@", weakString);


```
- NSString的问题，这个跟retainCount没什么太大的关系 
- 首先，stringWithFormat方法创建的字符串是autorelease的，本身就会延迟释放，直接跟log的话肯定不会输出null，如果你写个button做触发，放在方法外作log的话，才会打印出null
	- 在64位环境下，苹果对NSString做了优化，细节不说，具体表现是，当非字面值常量的数字，英文字母字符串的长度小于等于 9 的时候会自动成为 NSTaggedPointerString 类型，如果有中文或其他特殊符号存在的话则会直接成为__NSCFString 类型。而NSTaggedPointerString是个常量释放不掉的.
	- 最后，如果是使用@""或者initWithString:@""的方式创建的字符串，会被转换成__NSCFConstantString,也是个常量，释放不掉不会输出null

### SDWebImage实现原理是什么？ 它是如何解决tableView的复用时出现图片错乱问题的呢？
- 解决tableView复用错乱问题：每次都会调UIImageView+WebCache文件中的 [self sd_cancelCurrentImageLoad];
- [原理解释参考](https://www.jianshu.com/p/13c0cdc7987e)
	- SDWebImageDownloader  
	- 图片的下载操作放在一个NSOperationQueue并发操作队列中，队列默认最大并发数是6
	- 每个图片对应一些回调（下载进度，完成回调等），回调信息会存在downloader的URLCallbacks（一个字典，key是url地址，value是图片下载回调数组）中，URLCallbacks可能被多个线程访问，所以downloader把下载任务放在一个barrierQueue中，并设置屏障保证同一时间只有一个线程访问URLCallbacks。，在创建回调URLCallbacks的block中创建了一个NSOperation并添加到NSOperationQueue中
	- 下载的核心是利用NSURLSession加载数据，每个图片的下载都有一个operation操作来完成，并将这些操作放到一个操作队列中，这样可以实现图片的并发下载。
	- 内存缓存的处理由NSCache对象实现，NSCache类似一个集合的容器，它存储key-value对，类似于nsdictionary类，我们通常使用缓存来临时存储短时间使用但创建昂贵的对象，重用这些对象可以优化新能，同时这些对象对于程序来说不是紧要的，如果内存紧张就会自动释放。
	- 先在内存中放置一份缓存，如果需要缓存到磁盘，将磁盘缓存操作作为一个task放到串行队列中处理，会先检查图片格式是jpeg还是png，将其转换为响应的图片数据，最后吧数据写入磁盘中（文件名是对key值做MD5后的串）。

- [UML图](https:www.github.com/rs/SDWebImage)

## Swift题

### struct 和 class 的区别？
-  类可以继承，结构体不可以

-  可以让一个类的实例来反初始化，释放存储空间，结构体做不到

-  类的对象是引用类型，而结构体是值类型。所以类的赋值是传递引用 ，结构体则是传值。


### class与staitc关键字的区别？

- static 可以在类、结构体、或者枚举中使用。而 class 只能在类中使用。
-  static 可以修饰存储属性，static 修饰的存储属性称为静态变量(常量)。而 class 不能修饰存储属性。
-  static 修饰的计算属性不能被重写。而 class 修饰的可以被重写。
-  static 修饰的静态方法不能被重写。而 class 修饰的类方法可以被重写。
-  class 修饰的计算属性被重写时，可以使用 static 让其变为静态属性。
-  class 修饰的类方法被重写时，可以使用 static 让方法变为静态方法。

## 链接

- [面试题系列目录](../README.md)
- **上一份**: [基础问题系列](interview-iOS-6.md)
- **下一份**: [我的iOS面试之旅](interview-iOS-8-我的iOS面试之旅.md)
