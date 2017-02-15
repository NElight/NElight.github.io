# iOS热更新
iOS热更新主要是利用oc动态语言的特性来完成消息的运行时更改。常用的有WaxPatch和JSPatch，但是现在一般都用JSPatch。

**JSPatch有以下优点：**
1. 使用js语言作为修复脚本。
2. 更加符合apple的规定。
3. wax使用lua语言，并且停止更新。对oc一些语法支持的不好，比如block。

### 集成热更新
**集成热更新的方法主要有两种：**
1. github上下载开源的JSPatch库，利用自己的服务器来下发js脚本。
2. 利用JSPatch平台来进行热更新的集成。其SDK核心也是github上的JSPatch开源库，在其基础上封装了向平台请求脚本/传输解密/版本管理等功能。基础版免费，标准版收费399，定制版需要联系官方平台。

使用第一种方法需要下载JSPatch库，将JSPatch文件夹下的三个文件拷入目录，引入javaScriptCore框架。
在appDelegate的``didfinshlaunch:withOption:``方法返回之前加入下面的代码。
```
[JPEngine startEngine];
    NSString *sourcePath = [[NSBundle mainBundle] pathForResource:@"demo" ofType:@"js"];
    NSString *script = [NSString stringWithContentsOfFile:sourcePath encoding:NSUTF8StringEncoding error:nil];
    [JPEngine evaluateScript:script];
```
在demo.js文件中书写js代码用来动态修改代码。此处加载的是本地bundle种的js文件，实际应用中应该从服务器请求js文件。利用第二种方法的参照文档进行。
### JSPatch基础用法
##### 1.require  
     在使用Objective-C类之前需要调用`` require('className’) ``有一下三种使用方式：

```
require('UIView')
var view = UIView.alloc().init()

require('UIView, UIColor')
var view = UIView.alloc().init()
var red = UIColor.redColor()

require('UIView').alloc().init()
```

##### 2.方法名转换
多参数方法名使用``_``分隔：
```
var indexPath = require('NSIndexPath').indexPathForRow_inSection(0, 1);
```
若原 OC 方法名里包含下划线 ``_``，在 JS 使用双下划线 ``__`` 代替：
```
// Obj-C: [JPObject _privateMethod];
JPObject.__privateMethod()
```
**3.覆盖方法**  
主要调用的api方法为：
```
defineClass(classDeclaration, [properties,] instanceMethods, classMethods)

@param classDeclaration: 字符串，类名/父类名和Protocol
@param properties: 新增property，字符串数组，可省略
@param instanceMethods: 要添加或覆盖的实例方法
@param classMethods: 要添加或覆盖的类方法
```
下面给出一个实现的模板
```
// OC
@implementation JPTableViewController

+ (void)sharedInstance{
}

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
{
}
@end
```
```
// JS
defineClass("JPTableViewController: UIViewController<UIScrollViewDelegate, UITextViewDelegate>", ['data', 'totalCount'], {
    //实例方法
  tableView_didSelectRowAtIndexPath: function(tableView, indexPath) {
  self.ORIGviewDidLoad();
  self.super().viewDidLoad();
    ...
  },
}, {
    //类方法
  shareInstance: function() {
    ...  
  }
})
```
**注：**  
1. defineClass中方法名的规则与调用时相同。
2. 在方法名前面增加``ORIG``关键字就可以调用修改前的方法。
3. 修改Category中的方法使用相同的api。
4. 使用``self.super()``接口来代替``super``关键字。
5. 调用getter/setter的方式来获取/修改Property。
6. 私有成员变量使用``valueForKey()`` 和 ``setValue_forKey() ``来获取/更改。
7. 实现协议的时候和OC写法一致。


**4.增加方法**
可以随意给一个类增加OC未定义的方法，但是所有的参数类型都是``id``
```
// OC
NSString *data = [self dataAtIndex:@(1)];
// JS
var data = ["JS", "Patch"]
defineClass("JPTableViewController", {
  dataAtIndex: function(idx) {
     return idx < data.length ? data[idx]: ""
  }
})
```

**5.特殊类型**  
JSPatch原生支持 CGRect / CGPoint / CGSize / NSRange 这四个 struct 类型，用 JS 对象表示：
```
// JS
var view = UIView.alloc().initWithFrame({x:20, y:20, width:100, height:100})
view.setCenter({x: 10, y: 10})
view.sizeThatFits({width: 100, height:100})

var x = view.frame().x
var range = {location: 0, length: 1}
```

在JS中使用字符串代表Selector。
```
//Obj-C
[self performSelector:@selector(viewWillAppear:) withObject:@(YES)];

//JS
self.performSelector_withObject("viewWillAppear:", 1)
```
JS 上的 null 和 undefined 都代表 OC 的 nil，如果要表示 NSNull, 用 nsnull 代替，如果要表示 NULL, 也用 null 代替, 如果要判断是否为空请使用``!``运算符:
```
//Obj-C
@implemention JPTestObject
+ (BOOL)testNull(NSNull *null) {
    return [null isKindOfClass:[NSNull class]]
}
@end

//JS
require('JPTestObject').testNull(nsnull) //return 1
require('JPTestObject').testNull(null) //return 0
```
NSArray，NSString，NSDictionary当做普通的对象使用。如果需要转成js对象请使用``toJS()``。  
**6.Block**  
当要把 JS 函数作为 block 参数给 OC时，需要先使用 block(paramTypes, function) 接口包装:
```
// Obj-C
@implementation JPObject
+ (void)request:(void(^)(NSString *content, BOOL success))callback
{
  callback(@"I'm content", YES);
}
@end

// JS
require('JPObject').request(block("NSString *, BOOL", function(ctn, succ) {
  if (succ) log(ctn)  //output: I'm content
}))
```
这里 block 里的参数类型用字符串表示，写上这个 block 各个参数的类型，用逗号分隔。NSObject 对象如 NSString *, NSArray *等可以用 id 表示，但 block 对象要用 NSBlock* 表示。
从 OC 返回给 JS 的 block 会自动转为 JS function，直接调用即可。
