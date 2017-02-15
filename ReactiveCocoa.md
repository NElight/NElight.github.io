# ReactiveCocoa
ReactiveCocoa(RAC)是一个开源的应用于iOS和OS开发的框架。它是一个函数式响应变成框架，它的作用主要就是将响应事件的不通表现形式整合，提供一个统一的事件处理接口。将代码高度内聚，降低耦合程度。
### 导入ReactiveCocoa框架
RAC经过几个版本的发展，已经和原来大不相同，5.0版本做出了重要改变，主要是将原来和swift相关的代码提取出来，形成了多个库。
- 如果你只是纯 swift 项目，你继续使用 ReactiveCocoa 。但是 RAC 依赖于 ReactiveSwift ，等于你引入了两个库。

- 如果你的项目是纯 OC 项目，你需要使用的是 ReactiveObjC 。这个库里面包含原来 RAC 2 的全部代码。

- 如果你的项目是 swift 和 OC 混编，你需要同时引用 ReactiveCocoa 和 ReactiveObjCBridge 。但是 ReactiveObjCBridge 依赖于 ReactiveObjC ，所以你就等于引入了 4 个库。

### RAC中重要的类
#### RACSignal
这是RAC中核心的一个类，任何事情都是通过信号传递的。
```
 // RACSignal使用步骤：  
    // 1.创建信号 + (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe  
    // 2.订阅信号,才会激活信号. - (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock  
    // 3.发送信号 - (void)sendNext:(id)value

 // RACSignal底层实现：  
    // 1.创建信号，首先把didSubscribe保存到信号中，还不会触发。  
    // 2.当信号被订阅，也就是调用signal的subscribeNext:nextBlock  
    // 2.2 subscribeNext内部会创建订阅者subscriber，并且把nextBlock保存到subscriber中。  
    // 2.1 subscribeNext内部会调用siganl的didSubscribe  
    // 3.siganl的didSubscribe中调用[subscriber sendNext:@1];  
    // 3.1 sendNext底层其实就是执行subscriber的nextBlock
```  
#### RACSubscriber
表示订阅者的意思，用于发送信号，这是一个协议，不是一个类，只要遵守这个协议，并且实现方法才能成为订阅者。通过create创建的信号，都有一个订阅者，帮助他发送数据。
#### RACDisposable
用于取消订阅或者清理资源，当信号发送完成或者发送错误的时候，就会自动触发它。当需要取消监听摸个信号时，可以通过它主动取消订阅信号。
#### RACSubject  RACReplaySubject
RACSubject信号提供者，可以自己充当信号，又能发送信号。  
RACReplaySubject是RACSubject子类，重复提供信号。  
**区别和用途：  **
- RACReplaySubject可以先发送信号，在订阅信号，RACSubject就不可以。
-  使用场景一:如果一个信号每被订阅一次，就需要把之前的值重复发送一遍，使用重复提供信号类。
-  使用场景二:可以设置capacity数量来限制缓存的value的数量,即只缓充最新的几个值。
### RACTuple
元组类，用来包装值。
### RACSequence
RAC中的集合类，用于代替NSArray,NSDictionary,可以使用它来快速遍历数组和字典。  
**用法：**
```
// 第一步: 把数组转换成集合RACSequence numbers.rac_sequence
    // 第二步: 把集合RACSequence转换RACSignal信号类,numbers.rac_sequence.signal
    // 第三步: 订阅信号，激活信号，会自动把集合中的所有值，遍历出来。
NSArray *numbers = @[@1, @2, @3, @4];
    [numbers.rac_sequence.signal subscribeNext:^(id  _Nullable x) {
        NSLog(@"%@", x);
    }];
    
    NSDictionary *dic = @{@"name":@"xmg", @"age" : @18};
    [dic.rac_sequence.signal subscribeNext:^(RACTuple * _Nullable x) {
        RACTupleUnpack(NSString *key, NSString *value) = x;
        NSLog(@"%@ : %@", key, value);
    }];
```
### RACCommand
处理事件的类，可以把事件如何处理,事件中的数据如何传递，包装到这个类中，他可以很方便的监控事件的执行过程。
```
 // 一、RACCommand使用步骤:
    // 1.创建命令 initWithSignalBlock:(RACSignal * (^)(id input))signalBlock
    // 2.在signalBlock中，创建RACSignal，并且作为signalBlock的返回值
    // 3.执行命令 - (RACSignal *)execute:(id)input

    // 二、RACCommand使用注意:
    // 1.signalBlock必须要返回一个信号，不能传nil.
    // 2.如果不想要传递信号，直接创建空的信号[RACSignal empty];
    // 3.RACCommand中信号如果数据传递完，必须调用[subscriber sendCompleted]，这时命令才会执行完毕，否则永远处于执行中。
    // 4.RACCommand需要被强引用，否则接收不到RACCommand中的信号，因此RACCommand中的信号是延迟发送的。

    // 三、RACCommand设计思想：内部signalBlock为什么要返回一个信号，这个信号有什么用。
    // 1.在RAC开发中，通常会把网络请求封装到RACCommand，直接执行某个RACCommand就能发送请求。
    // 2.当RACCommand内部请求到数据的时候，需要把请求的数据传递给外界，这时候就需要通过signalBlock返回的信号传递了。

    // 四、如何拿到RACCommand中返回信号发出的数据。
    // 1.RACCommand有个执行信号源executionSignals，这个是signal of signals(信号的信号),意思是信号发出的数据是信号，不是普通的类型。
    // 2.订阅executionSignals就能拿到RACCommand中返回的信号，然后订阅signalBlock返回的信号，就能获取发出的值。

    // 五、监听当前命令是否正在执行executing

    // 六、使用场景,监听按钮点击，网络请求
```
### RACMulticastConnection  
用于当一个信号，被多次订阅时，为了保证创建信号时，避免多次调用创建信号中的block，造成副作用，可以使用这个类处理。  
使用注意:RACMulticastConnection通过RACSignal的-publish或者-muticast:方法创建.

```
// RACMulticastConnection使用步骤:
    // 1.创建信号 + (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe
    // 2.创建连接 RACMulticastConnection *connect = [signal publish];
    // 3.订阅信号,注意：订阅的不在是之前的信号，而是连接的信号。 [connect.signal subscribeNext:nextBlock]
    // 4.连接 [connect connect]

    // RACMulticastConnection底层原理:
    // 1.创建connect，connect.sourceSignal -> RACSignal(原始信号)  connect.signal -> RACSubject
    // 2.订阅connect.signal，会调用RACSubject的subscribeNext，创建订阅者，而且把订阅者保存起来，不会执行block。
    // 3.[connect connect]内部会订阅RACSignal(原始信号)，并且订阅者是RACSubject
    // 3.1.订阅原始信号，就会调用原始信号中的didSubscribe
    // 3.2 didSubscribe，拿到订阅者调用sendNext，其实是调用RACSubject的sendNext
    // 4.RACSubject的sendNext,会遍历RACSubject所有订阅者发送信号。
    // 4.1 因为刚刚第二步，都是在订阅RACSubject，因此会拿到第二步所有的订阅者，调用他们的nextBlock
    // 需求：假设在一个信号中发送请求，每次订阅一次都会发送请求，这样就会导致多次请求。
    // 解决：使用RACMulticastConnection就能解决.
```
### 用法
1 代替代理:

    rac_signalForSelector：用于替代代理。

2 代替KVO :

    rac_valuesAndChangesForKeyPath：用于监听某个对象的属性改变。

3 监听事件:

    rac_signalForControlEvents：用于监听某个事件。

4 代替通知:

    rac_addObserverForName:用于监听某个通知。

5 监听文本框文字改变:

    rac_textSignal:只要文本框发出改变就会发出这个信号。

6 处理当界面有多次请求时，需要都获取到数据时，才能展示界面

    rac_liftSelector:withSignalsFromArray:Signals:当传入的Signals(信号数组)，每一个signal都至少sendNext过一次，就会去触发第一个selector参数的方法。
    使用注意：几个信号，参数一的方法就几个参数，每个参数对应信号发出的数据。
