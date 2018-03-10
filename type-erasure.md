常用的 type erasure 方法就是使用协议和继承。

### 继承

对于继承来说，Foundation 里的 NSString/NSArray 等类型，使用类簇（cluster）的方式，抽象出父类接口（NSArray），使用的时候，根据不同的初始化方法生成内部的类型（__NSPlacehodlerArray）。

### 协议
Swift 里定义一个类型时，遵循某个协议或者继承自某个类，写起来都一样。但实际上 Swift 不允许把协议当成一个类型一样使用（当这个协议有关联类型或使用了 Self 关键字的时候）。例如：

```swift
func f(seq: Sequence<Int>) {} // error: cannot specialize non-generic type 'Sequence'
```

有些情况下可以使用泛型的方式（泛型类型仅用作函数参数类型）：

```swift
func f2<S: Sequence>(seq: S) where S.Element == Int { }
```

但是如果泛型类型是用来做函数返回值类型：

```swift
func f3<S: Sequence>() -> S where S.Element == Int { ... } // error: contextual type 'S' cannot be used with array literal
```

这个方法允许调用方自己选择想要的返回值类型（只需要遵循 Sequence 协议并且元素类型为 Int），但是这个方法的实现并不能事先知道调用方想要的类型，所以无法提供一种实现来满足所有调用方的需求。例如

```swift
class LeftSequence: Sequence {} ...
class RightSequence: Sequence {} ...


let left: LeftSequence<Int> = f3()
// or
let right: RightSequence<Int> = f3()
```

由于 LeftSequence 和 RightSequence 的实现可能天差地别，因此无法实现这么一个 f3 函数满足这两个类型。

1. 使用类（类簇）的方式进行类型擦除

```swift
class MAnySequence<Element>: Sequence {
    class Iterator: IteratorProtocol {
        func next() -> Element? {
            fatalError("Must Override next()")
        }
    }

    func makeIterator() -> Iterator {
        fatalError("Must override makeIterator()")
    }
}

extension MAnySequence {
    static func make<Seq: Sequence>(_ seq: Seq) -> MAnySequence<Element> where Seq.Element == Element {
        return MAnySequenceImpl<Seq>(seq)
    }
}

private class MAnySequenceImpl<Seq: Sequence>: MAnySequence<Seq.Element> {
    class IteratorImpl: Iterator {
        var wrapped: Seq.Iterator

        init(_ wrapped: Seq.Iterator) {
            self.wrapped = wrapped
        }

        override func next() -> Seq.Element? {
            return wrapped.next()
        }
    }

    var seq: Seq

    init(_ seq: Seq) {
        self.seq = seq
    }

    override func makeIterator() -> MAnySequence<Seq.Element>.Iterator {
        return IteratorImpl(seq.makeIterator())
    }
}
```

这种方法本质上是使用一个 public 的接口来接受方法调用，内部使用（不同的）子类作为容器，父类则根据初始化参数生成不同的子类；当接口接收到方法调用的时候”转发“给子类容器中的元素。

2. 使用函数的方式进行类型擦除

```swift
struct MAnySequence<Element>: Sequence {
    struct Iterator: IteratorProtocol {
        let _next: () -> Element?
        
        func next() -> Element? {
            return _next()
        }
    }
    
    let _makeIterator: () -> Iterator
    
    func makeIterator() -> MAnySequence<Element>.Iterator {
        return _makeIterator()
    }
    
    init<Seq: Sequence>(_ seq: Seq) where Seq.Element == Element {
        _makeIterator = {
            var i = seq.makeIterator()
            return Iterator(_next: { i.next() })
        }
    }
}
```

这种方法与 1 类似，但他是将元素的方法包装成 closure.


### 其他语言

* C 语言里的`void *`用法在一定程度上就是一种类型擦除。
* C++ 的模板（template）也是一种类型擦除
    * 优点：C++ 使用的是`Code Specialization`的方式，它为泛型类的每一种实例都生成独立的代码，不需要运行时判断具体类型，有利于加快代码的执行速度
    * 缺点：代码膨胀，编译时间长
* C#: 无论是源代码，还是编译后的中间语言 IL 还是运行时的 CLR，泛型类型符号都是存在的，例如 List<int> 与 List<String>
* Java: 通过类型擦除，Java 为每个泛型类创建唯一的字节码，也就是说 ArrayList<Int> 和 ArrayList<String> 是同一个类。泛型只是 Java 的语法糖。

### 总结

类型擦除这个概念在不同的语言中有着不同的含义，同时也有不同的实现方式。例如，Java 的类型擦除是针对将泛型代码编译为字节码的时候消除泛型实例类型，这使得泛型只是 Java 的一个语法糖功能，而 C++ 则是在编译的时候为泛型实例的每一种可能单独生成一份代码，在编译后的二进制代码里同样没有了泛型信息，而对于 Swift 来说，类型消除这个概念主要用于泛型协议（generic protocol），也就是有关联类型的协议。而且实现的主要方式就是使用泛型，将关联类型转化为泛型类型，从而达到消除关联类型的目的。

### 延伸阅读

* [Generics in Swift 4](https://theswiftpost.co/generics-swift-4/)
* [Java 的类型擦除](http://www.hollischuang.com/archives/226)
* [Friday Q&A 2015-11-20: Covariance and Contravariance](https://www.mikeash.com/pyblog/friday-qa-2015-11-20-covariance-and-contravariance.html)
* [Friday Q&A 2017-12-08: Type Erasure in Swift](https://www.mikeash.com/pyblog/friday-qa-2017-12-08-type-erasure-in-swift.html)
* [神奇的类型擦除](https://academy.realm.io/cn/posts/altconf-hector-matos-type-erasure-magic/)
* [Generics](https://github.com/apple/swift/blob/master/docs%2FGenerics.rst)
* [Generics Manifesto](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md)
* [关于序列和集合需要知道的二三事](https://academy.realm.io/cn/posts/try-swift-soroush-khanlou-sequence-collection/)
* [Breaking Down Type Erasure in Swift](https://www.bignerdranch.com/blog/breaking-down-type-erasure-in-swift/)
* [GENERIC PROTOCOLS & THEIR SHORTCOMINGS](https://krakendev.io/blog/generic-protocols-and-their-shortcomings)

