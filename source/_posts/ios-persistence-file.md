title: iOS持久化（二）：File
date: 2017-04-18 21:26:58
tags: [iOS, 持久化, NSUserDefaults]
categories: iOS
---

理论上所有的本地持久化都是存的文件，在这里指的是一些简单的文件存储，包括但不限于：

+ NSUserDefaults
+ Property List
+ Raw File

## NSUserDefaults

NSUserDefaults应该是使用频率最高的简单持久化方式了，没办法，用起来太方便了，只需要一个key就搞定了，也正是因为使用的太方便，导致很多工程里各种滥用，这一点在使用中一定要注意！一般的使用方式如`get`、`set`，很常用也很简单，这里就不多说了，但它的功能远不止这些，下面主要说下经常会被忽略的地方。

### 基础
可能对于NSUserDefaults一般的理解是：把保存的配置以Key-Value的方式，用plist作为存储格式保存到沙盒中。这么理解没错，但是不够全面，这样的话有点太小看NSUserDefaults了。

NSUserDefaults可以理解为对一系列配置文件的管理，并非只有app的配置文件，以*MVC*的角度来看，NSUserDefaults是*C*，每个配置文件可以认为是*Model*，NSUserDefaults使用**Domain**的方式进行管理。

### Domain

Domain可以理解为一类配置，是NSUserDefaults对于配置文件的组织方式，它的类型可以分为`volatile`和`persistent`，也就是临时保存在内存中和持久化到磁盘中。

默认有5种Domain，按照搜索顺序排序依次为：

+ NSArgumentDomain (volatile)
+ Application (persistent)
+ NSGlobalDomain (persistent)
+ Languages (volatile)
+ NSRegistrationDomain (volatile)

当我们进行`get`操作时，会按照上述顺序依次进行查询，直到找到对应的key，如果都没有找到，就会返回`nil`。

#### Argument Domain

这里用来保存程序启动的参数，在iOS上实际意义不大，启动参数可以在Xcode的*Edit Schemes*中进行配置，但是需要注意参数需要以`-key value`的方式传入。

获取方式如下：
```
UserDefaults.standard.volatileDomain(forName: UserDefaults.argumentDomain)
```

#### Application Domain

这里就是大家比较熟悉的了，我们的一般操作都是对于的这个Domain，用来存储App相关配置，plist文件存放在`Library/Preferences/$BundleId.plist`

#### Global Domain

这里用来给系统级`frameworks`保存一些系统相关的配置项，一般App不会用到。

#### Languages Domains

这里用来保存语言相关的配置项，App不会用到。

#### Registration Domain

这里用来定义一些`key`的默认值，当没有找到给定的`key`的value时，会使用这个Domain下面的`value`，因为它在NSUserDefaults搜索list的最下层。

registration domain是可以被设置的，方式如下：

```
let dict = ["key": "value"]
UserDefaults.standard.register(defaults: dict)
```

#### 自定义Domain

如果需要的话，可以向NSUserDefaults加入自定的Domain，但是不会加到它的搜索list中，只能通过获取Domain的方式来取到内容。

主要是这几个api，区分`volatile`和`persistent`方式:

```
open func volatileDomain(forName domainName: String) -> [String : Any]
open func setVolatileDomain(_ domain: [String : Any], forName domainName: String)
open func removeVolatileDomain(forName domainName: String)

open func persistentDomain(forName domainName: String) -> [String : Any]?
open func setPersistentDomain(_ domain: [String : Any], forName domainName: String)
open func removePersistentDomain(forName domainName: String)
```

### Suite

通过Suite可以让不同的App共享数据，只有Suite的名字相同就可以。

创建Suite

```
UserDefaults(suiteName: "suite")
```

将Suite加入到搜索树（注意该Suite在搜索树中的位置是Application Domain之后，Global Domain之前）

```
UserDefaults.standard.addSuite(named: "suite")
```

移除Suite

```
UserDefaults.standard.removeSuite(named: "suite")
```


到这里NSUserDefaults相关的内容就说完了（还没有包含iCloud部分），最开始我也以为只是一个单纯的App配置文件，没想到里面还有这么多道道，看来没事读读官方文档还是很有必要的。

## Property List

Property List就是我们俗称的plist，上面也提到过，以key-value的方式保存数据，其格式是基于xml的

### 读取
plist文件的读取方式如下

```
class func propertyList(from data: Data, options opt: PropertyListSerialization.ReadOptions = [], format: UnsafeMutablePointer<PropertyListSerialization.PropertyListFormat>?) throws -> Any
```

example:

```
let path = Bundle.main.path(forResource: "Info", ofType: "plist")
let data = FileManager.default.contents(atPath: path!)
var format = PropertyListSerialization.PropertyListFormat.xml
do {
	let dict = try PropertyListSerialization.propertyList(from: data!, options: PropertyListSerialization.ReadOptions.mutableContainers, format: &format) as? Dictionary<String, Any>
} catch  {
	print(error)
}
```

有一种比较简便的方式，使用plist文件路径构造一个`NSDictionary`

```
public convenience init?(contentsOfFile path: String)
public convenience init?(contentsOf url: URL)
```

example:

```
let path = Bundle.main.path(forResource: "Info", ofType: "plist")
let dict = NSDictionary(contentsOfFile: path!)
```

### 保存

可以先用下面的api转换为Data，再把Data保存到文件

```
class func data(fromPropertyList plist: Any, format: PropertyListSerialization.PropertyListFormat, options opt: PropertyListSerialization.WriteOptions) throws -> Data
```

这里的`PropertyListSerialization.WriteOptions`没有用到，随便写一个数就可以了

example:

```
let dict = ["key": "value"]
do {
	let data = try PropertyListSerialization.data(fromPropertyList: dict, format: PropertyListSerialization.PropertyListFormat.xml, options: 0)
	try data.write(to: URL(fileURLWithPath: "path_save_plist"))
} catch {
	print(error)
}
```

和读取一样，也可以使用`NSDictionary`的简便方法

```
func write(toFile path: String, atomically useAuxiliaryFile: Bool) -> Bool
func write(to url: URL, atomically: Bool) -> Bool
```

example:

```
let dict = ["key": "value"] as NSDictionary
dict.write(toFile: "path_save_plist", atomically: true)
```


## Raw File

最后要说的是纯文件操作，保存最原始的数据，不使用任何格式封装，类似c语言的`FILE`和c++的`fstream`，在iOS上使用的是`FileHandle`

### FileHandle

FileHandle是对file descriptor的面向对象封装，可以用它来存储files，sockets，pipes和devices，在这里只说file的方式，下面列举了一下会用到的api。

#### 打开文件

大家都知道，打开文件的方式一般有三种：**只读**，**只写** 和 **读写**

只读打开文件：

```
public convenience init?(forReadingAtPath path: String)
public convenience init(forReadingFrom url: URL) throws
```

只写打开文件：

```
public convenience init?(forWritingAtPath path: String)
public convenience init(forWritingTo url: URL) throws
```

读写打开文件：

```
public convenience init?(forUpdatingAtPath path: String)
public convenience init(forUpdating url: URL) throws
```

当然，也可以使用fd来打开文件：

```
public init(fileDescriptor fd: Int32, closeOnDealloc closeopt: Bool)
public convenience init(fileDescriptor fd: Int32)
```

#### 读写文件

read：

```
open func readDataToEndOfFile() -> Data
open func readData(ofLength length: Int) -> Data
```

write：

```
open func write(_ data: Data)
```

seek：

```
open func seekToEndOfFile() -> UInt64
open func seek(toFileOffset offset: UInt64)
```


到这里关于文件的三种持久化方式就说完了，也许并不全面，但应该是比较常用的了，这些都是比较轻量的，后面会说到功能强大的`CoreData`