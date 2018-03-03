## 协变和逆变

### 父类/子类/override

```swift
class Animal {}

class Cat: Animal {}
```

规则：函数（方法）返回值可以是子类，函数参数可以是父类。

### 函数/属性

规则：有两个函数 f1 和 f2，当 f1 的参数是 f2 的参数的父类，f1 的返回值（如果有）是 f2 的返回值的子类，则 f1 可以看成是 f2 的子类。其实这条规则跟上面是一样的。

对于属性，可以看成是 getter/setter 方法。

### 泛型

```swift
let var1: SomeType<A> = ...
let var2: SomeType<B> = var1
```

对于 SomeType<T> 来说，T 可以用来当作属性类型（property types），方法参数类型（method parameter types）和方法返回值类型（method return value types）。

1. T 仅用于**方法返回值**和**只读类型**的属性：那么当 B 是 A 的父类时是成立的

```swift
let var1: SomeType<Cat> = ...
let var2: SomeType<Animal> = var1
```

2. T 仅用于**方法参数**：那么当 B 是 A 的子类时是成立的：

```swift
let var1: SomeType<Animal> = ...
let var2: SomeType<Cat> = var1
```

3. 其他情况下只有当 A == B 时财成立。

正因为上面的规则如此复杂，所以 Swift 的泛型类型只允许 A == B 这一种规则成立（尽管在理论上其他两种情况都是可以接受的）。

### 结论

协变和逆变通常出现在 `重写方法并且修改参数或者返回值类型` 时。

> 事实上，Swift 里的一些集合类型支持协变，例如 Array/Dictionary
> 
> import UIKit 
> 
> // invariant
class Thing<T> { // could be a struct just as well 
    var thing: T 
    init(_ thing: T) { self.thing = thing } 
} 
var foo: Thing<UIView> = Thing(UIView()) 
var bar: Thing<UIButton> = Thing(UIButton()) 
foo = bar // error: cannot assign value of type 'Thing<UIButton>' to type 'Thing<UIView>'
>
> // covariant
var views: Array<UIView> = [UIView()] 
var buttons: Array<UIButton> = [UIButton()] 
views = buttons // it's ok
>

### 参考：[Friday Q&A 2015-11-20: Covariance and Contravariance](https://www.mikeash.com/pyblog/friday-qa-2015-11-20-covariance-and-contravariance.html)

