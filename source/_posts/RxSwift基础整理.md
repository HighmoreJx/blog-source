---
title: RxSwift基础整理
date: 2017-12-13 13:48:00
tags:
  - RxSwift
---
**初学rxswift，记录一些相关知识点，加深印象，如果有错误地方，麻烦指教，谢谢。**

### RxSwift 背景相关

Rx即响应式编程，是建立在观察者设计模式上的异步响应方式.  
iOS开发中,说到异步API可能会想到通知,委托,GCD, 闭包.不过多种方式结合,有时往往导致写出清晰明了的异步代码变得比较困难.Rx很好的解决了这个问题,而且Rx家族涵盖了很多流行语言,因此也会极大降低学习其他平台的成本.

_RxSwift只是语言层面上实现了Rx API,而其对Cocoa和UIKit相关类是一无所知的,RxCocoa为开发者搭建了两者间的桥梁._

### RxSwift 总览

被观察者（observables）是Rx的核心,它可以理解为一个管道（sequence）,最多可以发出三种种类的事件给观察者（observer）,三种事件分别为next,error,completed.

下面的网址是Rx具体操作符的在线图,如果对某些操作的作用不清楚的话,可以看看.

> <http://rxmarbles.com/>

### DisposeBag

> <http://adamborek.com/memory-managment-rxswift/>

DisposeBag是针对iOS平台所生成的产物,其本身是不属于Rx大家庭的.  
因为其异步的特性,所以使用者需要有决定的权限来控制什么时机来释放内存.  
```
public protocol Disposable {
    func dispose()
}
```

```
final class MyViewController: UIViewController {
    var subscription: Disposable?
    override func viewDidLoad() {
        super.viewDidLoad()
        subscription = theObservable().subscribe(onNext: {
            // handle your subscription
        })
    }
    deinit {
        subscription?.dispose()
    }
}
``` 

根据如上代码,订阅了被观察者时,Disposable就会强持有一个Observable,而Observable也会强持有Disposable,这就形成了一个循环引用.而解除这个循环引用的方式就是调用dispose.这就给使用者绝对的权限来释放内存.

当不只一个被观察者出现时,就可能需要写如下代码.  
```
final class MyViewController: UIViewController {
    var subscriptions = [Disposable]()
    override func viewDidLoad() {
        super.viewDidLoad()
        subscriptions.append(theObservable().subscribe(onNext: {
            // handle your subscription
        }))
    }
    deinit {
        subscriptions.forEach { $0.dispose() }
    }
}
```

按照上述.其实已经满足内存管理的需求了,不过仍然有很多改进点.  
```
final class MyViewController: UIViewController {
    let disposeBag = DisposeBag()
    override func viewDidLoad() {
        super.viewDidLoad()
        theObservable().subscribe(onNext: {
                // handle your subscription
        })
        .disposed(by: disposeBag)
    }
}
```

上述就是最终的改进模式.但值得注意的是,只有确保init会调用的时候,这种解决循环引用的方式才是正确的.但如下情况就还需特殊处理.  
```
final class MyViewController: UIViewController {
    private let disposeBag = DisposeBag()
    private let parser = MyModelParser()

    override func viewDidLoad() {
        super.viewDidLoad()
        let parsedObject = theObservable
            .map { json in
                return self.parser.parse(json)
            }
        parsedObject.subscribe(onNext:{ _ in 
            //do something
        })
        .disposed(by: disposeBag)
    }
}
```

上述情况,controller->DisposeBag->Disposable->Observable->controller就形成了循环引用.因此得特殊处理.  
```
//1.weak打破循环
let parsedObject = theObservable
    .map { [weak self] json in
        return self?.parser.parse(json) //compile-time error. What should be returned if `self` is nil?
    }

//2.传入对应变量到capturelist,不持有self
let parsedObject = theObservable
    .map { [parser] json in
        return parser.parse(json)
    }
```

总之,disposebag不是万金油,weak也不是万金油,需要自己根据实际情况规避循环引用.

### 创建和订阅被观察者

#### subscrible

订阅被观察者(sequence)所发出的事件,subscribe(onNext:),subscribe(onError:) ,subscribe(onCompleted:).

#### never

创建一个不发出任何事件的sequence.可用来表示无限的时间段.  
```
let disposeBag = DisposeBag()
let sequence = Observable<Any>.never()
let sequenceSub = sequence
    .subscribe { _ in
        print("event happen")
}.addDisposableTo(disposeBag)
```

#### empty

空的sequence,只能发送completed事件.  
使用场景一般可用来表示马上就结束的sequence.  
```
let disposeBag = DisposeBag()
   Observable<Void>.empty()
       .subscribe { event in
           print(event)
       }
       .addDisposableTo(disposeBag)
```

```
completed
``` 

#### just

发出特定事件的sequence  
```
let disposeBag = DisposeBag()
Observable.just("test")
    .subscribe { event in
        print(event)
    }
    .addDisposableTo(disposeBag)
```

```
next(test)
completed
``` 

#### of

接受一个可变参数的操作符,会根据数据类型自动推导出Observer  
也可用做数组,不过最好用from,那样意图更加明确  
```
let disposeBag = DisposeBag()
Observable.of(1, 2, 3)
    .subscribe(onNext: { element in
        print(element)
    })
    .addDisposableTo(disposeBag)
```

#### from

从集合创建sequence.  
```
let disposeBag = DisposeBag()
Observable.from([1, 2, 3])
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

#### create

自定义可观察的sequence,用of或者from等操作符都是用特定的.next事件来创建的被观察者,要灵活定制被观察者在不同情形下发出不同的事件类型,就需要用create这个快捷操作符.  
```
let disposeBag = DisposeBag()
let cusObserver = { (element: String) -> Observable<String> in
    return Observable.create { observer in
        observer.on(.next(element))
        observer.on(.completed)
        return Disposables.create()
    }
} 
cusObserver("test")
    .subscribe { print($0) }
    .addDisposableTo(disposeBag)
```

#### range

发出指定范围内事件的sequence  
```
let disposeBag = DisposeBag()
Observable.range(start: 1, count: 10)
    .subscribe { print($0) }
    .addDisposableTo(disposeBag)
```

#### repeatElement

无止境的重复发送单个元素的sequence  
```
let disposeBag = DisposeBag()
Observable.repeatElement("test")
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

#### generate

由状态驱动的循环发出事件的sequence,初始条件为真的时候开始发出事件,条件为假的时候结束.  
```
let disposeBag = DisposeBag()
Observable.generate(
        initialState: 0,
        condition: { $0 < 3 },
        iterate: { $0 + 1 }
    )
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

#### deferred

每当有新的订阅者订阅时,就创建一个新的队列.  
```
let disposeBag = DisposeBag()
var count = 1
let deferredSequence = Observable<String>.deferred {
    print("Creating \(count)")
    count += 1
    return Observable.create { observer in
        print("Emitting...")
        observer.onNext("test")
        return Disposables.create()
    }
}
deferredSequence
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
deferredSequence
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

#### error

只发出error事件的序列  
```
let error: Error = ...
let id = Observable<Int>.error(error)
```

#### doOn

doOnNext是在subscribe(onNext:)之前预先调用的方法,其余以此类推.  
```
let disposeBag = DisposeBag()
Observable.of("test")
    .do(onNext: { print("Intercepted:", $0) }, onError: { print("Intercepted error:", $0) }, onCompleted: { print("Completed")  })
    .subscribe(onNext: { print($0) },onCompleted: { print("结束") })
    .addDisposableTo(disposeBag)
```

### Subjects

Subjects既是观察者也是被观察者.

#### PublishSubject

订阅PublishSubject时,只能订阅到之后发生的事件.  
```
let disposeBag = DisposeBag()
let subject = PublishSubject<String>()
subject.onNext = "knock knock"
let subscriptionOne = subject.subscrible(onNext: {string in
	print(string)
})
subject.onNext("1")
let subscriptionTwo = subject.subscrible(onNext: {string in
	print("2)\(string)")
}) 
subject.onNext("2")
```

#### BehaviorSubject

有初始值,会接收到初始值赋予的事件或者最后一个事件.  
```
let disposeBag = DisposeBag()
let subject = BehaviorSubject(value: "knock knock")
let subscriptionOne = subject.subscrible(onNext: {string in
	print(string)
})
subject.onNext("1")
let subscriptionTwo = subject.subscrible(onNext: {string in
	print("2)\(string)")
}) 
subject.onNext("2")
```

#### ReplaySubject

缓存指定数量的事件,新的观察者来订阅时即接收缓存的时间.  
```
let disposeBag = DisposeBag()
let subject = ReplaySubject<String>.create(bufferSize: 1)
subject.addObserver("1").addDisposableTo(disposeBag)
subject.onNext("🐶")
subject.onNext("🐱")
subject.addObserver("2").addDisposableTo(disposeBag)
subject.onNext("🅰️")
subject.onNext("🅱️")
```

```
Subscription: 1 Event: next(🐶)
Subscription: 1 Event: next(🐱)
Subscription: 2 Event: next(🐱) //订阅之后还可以接受一次前面发出的事件
Subscription: 1 Event: next(🅰️)
Subscription: 2 Event: next(🅰️)
Subscription: 1 Event: next(🅱️)
Subscription: 2 Event: next(🅱️)
``` 

#### Variable

对BehaviorSubject的封装,使用时asObservable()来拆箱,不接受error或者completed事件,完成后会自动发出completed事件  
```
let disposeBag = DisposeBag()
let variable = Variable("🔴")
variable.asObservable().addObserver("1").addDisposableTo(disposeBag)
variable.value = "🐶"
variable.value = "🐱"
variable.asObservable().addObserver("2").addDisposableTo(disposeBag)
variable.value = "🅰️"
variable.value = "🅱️"
```

```
Subscription: 1 Event: next(🔴)
Subscription: 1 Event: next(🐶)
Subscription: 1 Event: next(🐱)
Subscription: 2 Event: next(🐱)
Subscription: 1 Event: next(🅰️)
Subscription: 2 Event: next(🅰️)
Subscription: 1 Event: next(🅱️)
Subscription: 2 Event: next(🅱️)
Subscription: 1 Event: completed
Subscription: 2 Event: completed
``` 

### 联合操作

#### startWith

发出事件前,先发出某个指定的事件.  
```
let disposeBag = DisposeBag()
Observable.of("2", "3")
    .startWith("1")
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

#### merge

合并两个sequence形成单个sequence,按照时间轴发出事件  
```
let disposeBag = DisposeBag()
let subject1 = PublishSubject<String>()
let subject2 = PublishSubject<String>()
Observable.of(subject1, subject2)
    .merge()
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

#### zip

将多个sequence的的事件一一绑定后发出.  
```
let disposeBag = DisposeBag()
let stringSubject = PublishSubject<String>()
let intSubject = PublishSubject<Int>()
Observable.zip(stringSubject, intSubject) { stringElement, intElement in
    "\(stringElement) \(intElement)"
    }
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

#### combineLatest

与zip类似,区别在于绑定的事件都是最新的事件.  
```
let disposeBag = DisposeBag()
let stringSubject = PublishSubject<String>()
let intSubject = PublishSubject<Int>()
Observable.combineLatest(stringSubject, intSubject) { stringElement, intElement in
        "\(stringElement) \(intElement)"
    }
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
stringSubject.onNext("🅰️")
stringSubject.onNext("🅱️")
intSubject.onNext(1)
intSubject.onNext(2)
stringSubject.onNext("🆎")
```

```
🅱️ 1
🅱️ 2
🆎 2
``` 

#### switchLatest

可以对观察的事件源进行切换  
```
let disposeBag = DisposeBag()
let subject1 = BehaviorSubject(value: "1")
let subject2 = BehaviorSubject(value: "2")
let variable = Variable(subject1)
variable.asObservable()
    .switchLatest()
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
subject1.onNext("3")
subject1.onNext("4")
variable.value = subject2
subject1.onNext("5")
subject2.onNext("6")
variable.value = subject1
```

```
1
3
4
2
6
5
``` 

#### concat

合并多个sequence为一个,并且只有当前一个sequence的事件发出了completed事件.才会开始下一个sequence的事件.且当前一个sequence事件完成前,第二个sequence发出的除了最后一个事件外,都将会被忽略.  
```
let disposeBag = DisposeBag()
let subject1 = BehaviorSubject(value: "1")
let subject2 = BehaviorSubject(value: "2")
let variable = Variable(subject1)
variable.asObservable()
    .concat()
    .subscribe { print($0) }
    .addDisposableTo(disposeBag)
subject1.onNext("3")
subject1.onNext("4")
variable.value = subject2
subject2.onNext("5")	//1完成前，会被忽略
subject2.onNext("6") //1完成前，会被忽略
subject2.onNext("7")	//1完成前的最后一个，会被接收
subject1.onCompleted()
subject2.onNext("8")
```

```
1
3
4
7
8
``` 

### 变换操作

#### map

用一个函数闭包讲原来的sequence变换为一个新的sequence的操作符  
```
let disposeBag = DisposeBag()
Observable.of(1, 2, 3)
    .map { $0 * $0 }
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
1
4
9
``` 

#### mapWithIndex

和map类似,只是多了一个index来标识当前是第几个元素  
```
let disposeBag = DisposeBag()
Observable.of(1, 2, 3)
    .mapWithIndex { integer, index in
		index * interger
	}
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
0
2
6
``` 

#### flatMap

将源sequence的每一个元素应用flatMap提供转换方法,转换成多个sequence,然后将多个sequnce合并成一个sequence发出.  
```
let disposeBag = DisposeBag()
let first = BehaviorSubject(value: "👦🏻")
let second = BehaviorSubject(value: "🅰️")
let variable = Variable(first)
variable.asObservable()
        .flatMap { $0 }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
first.onNext("🐱")
variable.value = second
second.onNext("🅱️")
first.onNext("🐶")
```

```
👦🏻
🐱
🅰️
🅱️
🐶
``` 

#### flatMapLatest

和flatMap类似,但只保留最新的  
```
let disposeBag = DisposeBag()
let first = BehaviorSubject(value: "👦🏻")
let second = BehaviorSubject(value: "🅰️")
let variable = Variable(first)

variable.asObservable()
        .flatMapLatest { $0 }
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)

first.onNext("🐱")
variable.value = second
second.onNext("🅱️")
first.onNext("🐶")
```

```
👦🏻
🐱
🅰️
🅱️
``` 

### 过滤和约束

#### elementAt

过滤处在指定位置的事件  
```
Observable.of("1", "2", "3", "4", "5", "6")
    .elementAt(3)
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
4
``` 

#### filter

过滤掉不符合条件的事件  
```
let disposeBag = DisposeBag()
    
Observable.of(1, 2, 3)
    .filter {
        $0 > 2 
    }
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
3
``` 

#### distinctUntilChanged

去掉相临且重复的事件  
```
let disposeBag = DisposeBag()
Observable.of("1", "2", "3", "3", "3", "4", "3")
    .distinctUntilChanged()
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
1
2
3
4
3
``` 

#### skip

调过前面指定数量的事件  
```
let disposeBag = DisposeBag()
    
Observable.of(1, 2, 3)
    .skip(2)
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
3
``` 

#### skipWhile

调过满足条件的事件  
```
let disposeBag = DisposeBag()  
Observable.of(1, 2, 3, 4, 5, 6)
    .skipWhile { $0 < 4 }
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
4
5
6
``` 

#### skipWhileWithIndex

和skipWhile类似,只是可以获取到元素的index  
```
let disposeBag = DisposeBag()
Observable.of(1, 2, 3, 4, 5, 6)
    .skipWhileWithIndex { element, index in
        index < 3
    }
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
5
6
``` 

#### skipUntil

sequenceB作为sequenceA的触发器,当sequenceB发出事件后,sequenceA才会开始接收事件  
```
let disposeBag = DisposeBag()  
let sequenceA = PublishSubject<String>()
let sequenceB = PublishSubject<String>()
sequenceA
    .skipUntil(sequenceB)
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
sequenceA.onNext("1")
sequenceA.onNext("2")
sequenceA.onNext("3")
sequenceB.onNext("4")
sequenceA.onNext("5")
sequenceA.onNext("6")
}
```

```
5
6
``` 

#### take

只获取前几个事件  
```
let disposeBag = DisposeBag()
Observable.of(1, 2, 3, 4)
    .take(2)
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
1
2
``` 

#### takeLast

只获取后几个事件  
```
let disposeBag = DisposeBag()
Observable.of(1, 2, 3, 4)
    .takeLast(2)
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
3
4
``` 

#### takeWhile

只获取满足条件的事件  
```
let disposeBag = DisposeBag()
Observable.of(1, 2, 3, 4, 5, 6)
    .takeWhile { $0 < 3 }
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
1
2
``` 

#### takeUntil

sequenceB作为sequenceA的中止器,当sequenceB发出事件后,sequenceA停止接收事件  
```
let disposeBag = DisposeBag()  
let sequenceA = PublishSubject<String>()
let sequenceB = PublishSubject<String>()
sequenceA
    .takeUntil(sequenceB)
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
sequenceA.onNext("1")
sequenceA.onNext("2")
sequenceA.onNext("3")
sequenceB.onNext("4")
sequenceA.onNext("5")
sequenceA.onNext("6")
}
```

```
1
2
3
``` 

### 数学操作

#### toArray

将sequence转换成一个array,并转换成单一事件,然后结束  
```
let disposeBag = DisposeBag()
Observable.range(start: 1, count: 10)
    .toArray()
    .subscribe { print($0) }
    .addDisposableTo(disposeBag)
```

```
next([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
completed
``` 

#### reduce

用一个初始值对事件进行累计操作  
```
let disposeBag = DisposeBag()
Observable.of(10, 100, 1000)
    .reduce(1, accumulator: +)
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
1111
``` 

#### scan

与reduce类似, 但是每次操作以后就会发出一个事件  
```
let disposeBag = DisposeBag()
Observable.of(10, 100, 1000)
    .scan(1) { aggregateValue, newValue in
        aggregateValue + newValue
    }
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

```
11
111
1111
``` 

### 连接性操作

Connectable Observable出现订阅者时并不立即发送事件,只有当手动调用connect()时才开始发出事件.

#### publish

将sequence转换成connetable sequence  
```
print("Create observable")  
let observable = Observable.just("highmore").publish()  
print("start subscribe")  
observable.subscribe(onNext: {  
    print("first subscribe = \($0)")  
}).addDisposableTo(disposeBag)  
observable.subscribe(onNext: {  
    print("second subscrible = \($0)")  
}).addDisposableTo(disposeBag)  
delay(3) {  
    print("Calling connect after 3 seconds")  
    _ = observable.connect()  
}
```

```
Create observable
start subscribe
Calling connect after 3 seconds
first subscribe = highmore
second subscrible = highmore
``` 

不过publish+connect的组合有个问题是即使所有的subscription都被dispose,observable依然处于hot状态.  
而refcount()就可替代connect()来应对这个问题.  
```
print("create observable")  
   let observable = Observable<Int>.interval(1, scheduler: MainScheduler.instance)  
       .publish().refCount()  
   print("start subscribe")  
   let firstSubscribe = observable.subscribe(onNext: {  
       print("next = \($0)")  
   })  
   delay(3) {  
       print("dispose at 3 seconds")  
       firstSubscribe.dispose()  
   }  
   delay(6) {  
       print("subscribe again at 6 seconds")  
       observable.subscribe(onNext: {  
           print("next = \($0)")  
       }).addDisposableTo(self.disposeBag)  
   }
```

```
create observable
start subscribe
next = 0
next = 1
next = 2
dispose at 3 seconds
subscribe again at 6 seconds
next = 0
next = 1
next = 2
...
``` 

#### replay

将sequence转换成connetable sequence, 和push不同的是,reply会缓存指定数量的事件.  
```
let intSequence = Observable<Int>.interval(1, scheduler: MainScheduler.instance)
    .replay(5)	//接收到订阅之前的5条事件消息
_ = intSequence
    .subscribe(onNext: { print("Subscription 1:, Event: \($0)") })
delay(2) { _ = intSequence.connect() } 
delay(4) {
    _ = intSequence
        .subscribe(onNext: { print("Subscription 2:, Event: \($0)") })
}
delay(8) {
    _ = intSequence
        .subscribe(onNext: { print("Subscription 3:, Event: \($0)") })
}
```

#### multicast

将一个正常的sequence转换成一个connectable sequence，并且通过特性的subject发送出去，比如PublishSubject，或者replaySubject，behaviorSubject等。不同的Subject会有不同的结果。  
```
let subject = PublishSubject<Int>()
_ = subject
    .subscribe(onNext: { print("Subject: \($0)") })
let intSequence = Observable<Int>.interval(1, scheduler: MainScheduler.instance)
    .multicast(subject)
_ = intSequence
    .subscribe(onNext: { print("\tSubscription 1:, Event: \($0)") })
delay(2) { _ = intSequence.connect() }
delay(4) {
    _ = intSequence
        .subscribe(onNext: { print("\tSubscription 2:, Event: \($0)") })
}
```

### 错误处理

#### catchErrorJustReturn

发送error时,将error转换为指定的值,然后结束  
```
let disposeBag = DisposeBag()
let sequenceThatFails = PublishSubject<String>()
sequenceThatFails
    .catchErrorJustReturn("😊")
    .subscribe { print($0) }
    .addDisposableTo(disposeBag)
sequenceThatFails.onNext("😬")
sequenceThatFails.onNext("😨")
sequenceThatFails.onNext("😡")
sequenceThatFails.onNext("🔴")
sequenceThatFails.onError(TestError.test)
```

```
next(😬)
next(😨)
next(😡)
next(🔴)
next(😊)
completed
``` 

#### catchError

捕获error进行处理,可返回另一个sequence进行订阅  
```
let disposeBag = DisposeBag()
let sequenceThatFails = PublishSubject<String>()
let recoverySequence = PublishSubject<String>()
sequenceThatFails
    .catchError {
        print("Error:", $0)
        return recoverySequence
    }
    .subscribe { print($0) }
    .addDisposableTo(disposeBag)
sequenceThatFails.onNext("😬")
sequenceThatFails.onNext("😨")
sequenceThatFails.onNext("😡")
sequenceThatFails.onNext("🔴")
sequenceThatFails.onError(TestError.test)
    
recoverySequence.onNext("😊")
```

```
next(😬)
next(😨)
next(😡)
next(🔴)
Error: test
next(😊)
``` 

#### retry

发生error时进行重试  
```
let disposeBag = DisposeBag()
var count = 1
let sequenceThatErrors = Observable<String>.create { observer in
    observer.onNext("🍎")
    observer.onNext("🍐")
    observer.onNext("🍊")
    if count == 1 {
        observer.onError(TestError.test)
        print("Error encountered")
        count += 1
    }
    observer.onNext("🐶")
    observer.onNext("🐱")
    observer.onNext("🐭")
    observer.onCompleted()
    return Disposables.create()
}
sequenceThatErrors
    .retry(3)		//不传入数字的话，只会重试一次
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

想进一步控制重试,比如控制重试次数及重试间隔.就可以用retryWhen

### 调试信息

#### debug

打印所有的订阅, 事件和disposals  
```
sequenceThatErrors
    .retry(3)
    .debug()
    .subscribe(onNext: { print($0) })
    .addDisposableTo(disposeBag)
```

#### RxSwift.Resources.total

查看资源占用  
```
print(RxSwift.Resources.total)
```

只是先记录部分知识点,后续会逐步完善.
