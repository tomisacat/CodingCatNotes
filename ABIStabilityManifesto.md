> 翻译的非常 naive，个人笔记，仅供参考。

兼容性包括源码兼容和二进制兼容。

源码兼容是指无需修改源码即可编译不同版本的 Swift 源码，尤其是指使用新版本编译器编译之前的代码。

二进制框架包含一个 Swift 模块文件，它表示这个框架暴露出来的公共接口；以及一个二进制库文件，它表示一个库被编译器编译后的二进制实现，因此二进制兼容包括两方面：

* 模块格式稳定性：包括 API 声明和内联代码，编译器可以用它进行类型检查和代码生成等工作
* ABI 稳定性：它确保不同版本 Swift 编译器编译出来的二进制代码可以互相调用，包括如何调用函数、数据在内存中如何表示，元数据存放的位置以及如何访问元数据。

**ABI 稳定仅仅影响外部公开接口和符号的不可变性**。至于内部符号、调用惯例和数据布局可以持续修改而不影响 ABI 的稳定性。比如说，未来版本的编译器可以自由地修改内部函数的调用惯例，只要公开接口不变就可以了。

ABI 稳定后，未来的 Swift 版本可以添加与当前 ABI 正交的修改，这称为 **ABI 附加** 修改。它允许我们扩展或者渐进式地锁定 ABI。

---

Swift ABI 包含以下部分：

1. **内存布局**：类型（包括 struct 和 class）实例在内存中的数据布局；
2. **类型元数据**：Swift 程序大量依赖类型的元数据信息，例如运行时，反射，调试器等工具，因此元数据必须至少满足这两个条件之一：要么有一个确定的内存布局，要么有定义好的 API 用于查询类型元数据信息；
3. **Name Mangling**：所有暴露出来的符号都必须有一个独一无二的名字。Swift 支持函数重载和基于上下文的命名空间（例如模块和类型），这意味着在代码里名字可能不是全局唯一的。为了实现外部符号唯一这个目的，我们使用一个叫做名称改编（name mangling）的技术；
4. **Calling Convention**：函数需要知道如何互相调用，这需要明确定义调用栈的布局，哪些寄存器的值需要被保护，以及所有权惯例。
5. **运行时**：Swift 自带一个运行时库，用来处理类型动态转换，引用计数，反射操作等。Swift 程序需要调用运行时库，因此，运行时的 API 是 Swift ABI 的一部分。
6. **标准库**：Swift 自带一个标准库，定义了许多通用的类型和操作。为了让多个 Swift 编译器编译出来的程序能够协同调用，标准库必须提供一个稳定的 API，因此，标准库的 API 也是 Swift ABI 的一部分。

## 1. 数据内存布局

### 背景信息

先定义几个接下来要用到的术语：

* **对象**：内存或者寄存器中的一个实体，它可以是任何类型，包括结构体，枚举，class，协议，甚至闭包。这与通常意义上的面向对象语言里的对象不太一样。
* **数据成员**：对象内部的对象，包括它的存储属性和关联属性。
* **闲置位**：一个对象在内存布局里没有用到的位，通常是因为对象内存布局中的对齐和填充而产生的。
* **extra inhabitant**：是一种无法用给定类型表示出有效值的位模式。例如一个有 3 种情况的枚举类型可以用 2 个位（bit）来表示，但是两个位可以表示 4 种情况，因此第四种位模式就表示无效的 extra inhabitant

数据布局，定义了一个对象的数据在内存中的布局，包括对象所需要的内存大小，对齐方式，如何查询数据成员。

#### 布局和属性

对于一个可以在编译期明确数据布局的类型 T 来说，ABI 由下面几个条件确定：

* 对齐（alignment）：例如类型 T 里的一个属性 x，x 的地址与对齐值的模运算结果总是 0
* 大小（size）：对象大小，不包含末尾填充，以字节为单位
* 偏移（offset）：成员属性在内存中相对对象基地址的偏移地址

基于对齐和大小还衍生出了步幅（stride）这个概念，它表示对象大小根据对齐向上取整的大小。

有些类型还有这样的属性：

* trivial 类型：如果一个类型对象仅用于存储数据，没有多余的复制，移动或释放语义，就是 trivial 类型。
* bitwise movable：一个类型如果没有基于它的实例地址的引用表就称这个类型是 bitwise movable。这种类型的对象在移动时就是简单的将原来内存地址处的值按位移动到新内存地址处并将原地址置为无效内存。一个类型只有当它的所有成员属性都是 bitwise movable 的时候才称这个类型是 bitwise movable 的，例如上面的 trivial 类型。

一个 trivial 类型的例子：Point 结构体，它有两个 Double 类型的成员属性。一个 bitwise movable 但却是非 trivial 类型的例子：一个包含 class 类型成员属性的结构体。复制这个类型的某个对象的时候你不能简单的按位复制它的内存数据，因为**复制 class 类型成员属性的时候需要进行 retain 操作**。一个既不是 trivial 类型也不是 bitwise movable 的例子：一个包含弱引用成员属性的结构体。弱引用对象存放在一个副表里，可以被置为 nil，当移动一个对象到另一个地址时，需要更新这个副表里这个弱引用对象到新的地址。

#### 不透明布局

运行时才能确定布局的情况称为不透明布局。它可以是一个未特化的泛型，在编译期这个泛型并不知道自己在内存中的数据布局。它也可以是一个 弹性布局（resilient layout）类型。

一个不透明布局对象的大小和对齐，以及它是否是 trivial 或者 bitwise movable 类型，都是通过查询它的 `value witness table` 来得到，而数据成员的偏移量则通过查询类型的**元数据**得到。不透明布局的对象在使用的时候必须通过间接传递，比如说 Swift 运行时通过指针来操作不透明布局对象，因此这些对象必须是**可寻址**的。

事实上，一个对象的内存布局在编译期可能有一部分是能够确定的。比如说一个泛型为 T 的结构体，它有一个整型成员和一个 T 成员，在这种情况下，整型的内存布局甚至它在这个结构体对象的位置都是可以确定的。但是，因为这个结构体有一个不透明布局对象，因此整体上来说，这个结构体的大小和对齐是无法确定的。我们应该将这个不透明布局的成员放在这个结构体对象的尾部来保证非不透明类型成员的偏移量是确定的。

#### 程序员库的演进

程序库的演进为公共类型（public types）引入了弹性布局（resilient layout），同时它也提供了语法标记来固定布局以便提高性能。弹性布局通过将布局不透明化来避免易被破坏的二进制问题（fragile binary problem）。它有更大的自由度来修改和演进布局，同时也不破坏库的二进制兼容性：公开的数据成员可以被重新排列，添加，甚至删除。

此外，我们还引入了一个一个叫弹性域（resilient domain）的概念来允许跨模块的优化：它将多个模块合并成一个组，并锁定他们的版本。因此这破坏了跨版本二进制兼容这个特性。

#### 抽象层级

Swift 程序里的类型从概念上来说分为好几个抽象层级。例如，整型是一个具体类型并且可以通过寄存器传递给函数。但是同样的一个值也可能传递给一个接收泛型 T （也就是不透明布局类型）的函数，因为这个函数预期接收的参数值是通过间接传递过来的，因此这个整型值必须要压入到调用栈里。抽象层级之间的转换通过一个叫做重抽象（reabstraction）的操作完成。

### 类型之旅

#### 结构体

结构体的布局算法应该高效利用内存空间，这可能会使最后的布局与声明成员属性时不同。还有一点要考虑的是：我们需要保证结构体成员属性是可寻址的，否则的话我们宁愿通过位压缩的方式来节省内存空间。

#### 元组

元组类似于一个匿名的结构体，但它们的不同之处在于元组可以表示出结构性分类：例如说一个 (Bool, Bool) 元组可以被传递给任何一个预期接收 (T, U) 的调用，但是 (T, U) 的抽象层级比 (Bool, Bool) 更高。因此，如果元组的内存布局被压缩时可能会面临更大的重抽象开销。

考虑到这一点我们可能需要给元组设计一个简单、按照定义序、不压缩的布局算法。元组通常用于保存比较小的本地变量，并且极少面临跨 ABI 调用时压缩会导致性能问题的情况。这与 C 数组被导入到 Swift 中类似（C 数组导入后成为元组）。

我们应该考察是否需要像结构体那样付出重抽象的代价对元组进行压缩。

#### 枚举

一个枚举类型有多个可能情况（case），一个枚举的值由鉴别器（discriminator）决定，鉴别器也称为标签（tag），它是用一个整型值来表示当前存储的是枚举中的哪个 case。为了节省内存空间，鉴别器可以保存在枚举的空余位（spare bits）或者 extra inhabitants

枚举可以用 `@closed` 修饰，它表示这个枚举不能后续添加新的 case，下面是几种可以用 `@closed` 修饰的情况：

* 退化的（degenerate）：枚举没有 case 或者只有一个 case，并且这个 case 没有关联值
* 朴素的（trivial）：所有 case 没有关联值
* 单负载（single payload）：只有一个 case 有关联值
* 多负载（multi-payload）：有多个 case 有关联值

退化枚举不占用内存空间，朴素枚举就是指他们的鉴别器（discriminator）。

单负载枚举会尝试将它的鉴别器放置在负载的 extra inhabitants 里，如果不行的话就将它存储在负载后面，这种情况下，如果这个枚举的值恰好是这个带负载的 case，那么可以忽略这个鉴别器的值（不会设置它的值）。由于这个负载不使用 extra inhabitants 数据，因此布局算法可以保证负载在内存中的布局与这个枚举是兼容的。由于对齐的关系，将鉴别器放置在负载后面也可能提高枚举布局的效率。

对于多负载枚举来说情况会更复杂，布局算法应该尝试重新排列负载的位置来节省内存空间，这也可以带来提高性能和减少代码尺寸的好处。

枚举的原始值不是 ABI，因为它们是用计算属性来实现的。用 `@objc` 修饰的属性是与 C 语言兼容的，因此它们只能含有朴素 case。

库的演进为枚举增加了一个 `@open` 标记（同时也表明它是弹性的 resilient），它允许库的开发者给枚举添加新的 case 或者重新排列 case 的顺序而不破坏二进制兼容性。

#### 类

对于类的布局来说有两个概念需要讨论：类的实例，它存放在堆里面；指向类的实例的引用，它是基于引用计数的指针。

##### 类的实例

类的实例布局多数情况下是不透明的，这么做是为了避免“易碎的二进制接口”问题。对于没有用 final 修饰的类来说它在运行时的类型其实是不确定的，为了支持类型的动态转换，实例对象必须要存储一个指向它的类型的指针，称为 isa，这个指针总是存储在对象的内存开头。至于这个类型如何表示以及它能够提供什么样的信息是由类的元数据决定的。没有用 fianl 标记的函数与之类似。

作为 ABI 兼容的一部分，类实例将会在 isa 指针后面添加一个以字长为单位的不透明数据，它可以给运行时提供引用计数相关的信息。但是刚开始的话，这个不透明数据的格式和调用惯例不用考虑 ABI 兼容，以便提高语言设计和实现的灵活性。在未来的版本中可以将这个数据格式和调用惯例的锁定作为 ABI 附加修改的一部分。

##### 引用

类是引用类型。也就是说，在二进制层级的话，Swift 通过指针来处理类的操作。

指向与 Objective-C 兼容的类时要在底层提供跟 Objective-C 运行时一样的引用，因此，这些引用也是不透明对象。

指向 Swift 原生、不与 Objective-C 兼容的类时就没有上面这种限制了。Swift 原生类对象的对齐是 ABI 的一部分，并且在引用的低地址处留有空余位。对于不同的平台（操作系统）来说可能也要提供空余位（在高地址处）和 extra inhabitants（在低地址处）。

我们可能会尝试在空余位存储引用计数相关的信息来提高 ARC 相关操作的效率。

#### Existential Containers

在类型论中，存在类型（existential type）表示一个抽象类型的接口，一个存在类型的值称为存在值（existential value），Swift 里的协议（protocol）就是一种存在类型。证据表（witness table）是一个存储函数指针的表，这些函数指针指向遵循协议的函数实现。

遵循协议的类型使用存在容器（existential container）将对于协议的实现保存在证据表旁边。对于非 class 类型的存在类型，这个容器需要保存：

* 这个值本身：要么存放在内部的缓冲区，要么使用一个指针指向一个外部存储
* 一个指向类型元数据的指针
* 一个指向存储遵循协议类型的 witness table 的指针

而 class 类型的存在类型不需要保存指向类型元数据的指针（因为类对象本身就有一个指向自己类型元数据的指针）

#### 声明的稳定性

除了上面提到的这些之外，在 ABI 稳定之后，我们可能还会提出更激进的算法来改进布局，例如我们可能会探索一下重新排列和压缩内嵌类型和外部类型数据成员。这些提升将作为 ABI 附加修改的一部分。

## 2. 类型元数据

数据内存布局表示一个特定类型对象的布局信息，而类型元数据保存类型本身的信息，如何访问这些信息是 Swift ABI 的一部分。

Swift 为所有的具体类型都保存了他们的元数据信息，具体类型包括：所有的非泛型类型，或者有具体类型参数的泛型类型。元数据可能在编译期或运行时（泛型实例化的时候）生成。

一个实现 ABI 稳定性的可行机制就是给运行时提供元数据的读写功能，这可以给底层结构的修改带来一些自由。这么做的话可以让大部分元数据保持不透明，但是某些特定部分需要能够提供尽量高效的访问接口，如果通过一个中间函数来访问的话可能会达不到我们想要的性能。因此，我们可能需要固定如何访问这些性能优先级较高的部分。

目前的元数据信息里有一些历史遗留信息需要去除掉，同时我们也想给元数据提供更好的语义信息以便于将来能够提供更多的功能，例如说反射功能。

### 泛型参数

在运行时，所有的对象都有了具体类型，如果在源码层面这个类型是一个泛型类型，那么这个具体类型是通过泛型的实例化生成的。泛型实例化元数据信息给每一个泛型类型参数提供了类型元数据信息。

### 值类型元数据

实名值类型保存了类型的名字信息，如果一个类型是内嵌类型的话，还会保存一个指向它的外部类型的指针。

值类型元数据还有类型特定的入口。结构体类型元数据保存它的成员信息，成员的偏移量、名字和元数据。枚举类型元数据保存它的所有 cases，负载的大小以及负载的元数据。元组类型元数据保存它的元素和标签信息。

#### 值证据表

每个具体类型都有一个相关的值证据表（value witness table），它保存着这个类型如何布局和交互的信息。如果一个值类型含有不透明布局，那么它的实际布局和属性信息在编译期是不确定的，这就是为什么需要这个表的原因。

值证据表保存了这个类型是否是朴素类型；是否可按位移动；是否含有 extra inhabitants 以及如果有的话如何读写。对于枚举类型来说，值证据表同时还会提供与 discriminator 的交互。

### class 类型元数据

对于 Apple 平台上的 Swift 环境来说，class 类型元数据与 Objective-C 是布局兼容的，在类型元数据的开头将会保存这个类型的父类指针，实例对象的大小、对齐、标记，以及与 Objective-C 运行时交互的不透明信息等。接下来是父类的成员信息、父类的元数据、泛型参数元数据、成员属性以及 vtable。

#### 方法分发

对一个没有标记为 final 的实例方法的调用就是对一个在编译期无法确定具体函数的调用：通过运行时查询 vtable （virtual method table，虚方法表）信息来确定调用哪个具体函数。vtable 是一个保存类的函数指针或者子类对可重写方法实现的表。如果将 vtable 作为 ABI 的一部分，那它也需要提供一个有可扩展性的布局算法。

或者，我们可以通过不透明调用或者编译器创建的中间函数来进行模块间的调用，这样我们可以在不破坏二进制兼容性的情况下允许内部修改类的层级来实现库的演进。当然，这样的话 vtable 就不能作为 ABI 的一部分了。

### 协议和存在类型元数据

#### 协议证据表

协议证据表是一个保存某个类型对协议接口实现的函数表。如果这个协议有关联类型，那么这个证据表同样会存储这个关联类型的元数据信息。

协议证据表既可以在运行时动态创建也可以通过编译期静态生成，它是 ABI 的一部分。

#### 存在类型元数据

存在类型元数据包含当前证据表的数量，类型是否是 class 类型，对于每个协议来说还有一个协议描述符。协议描述符描述了一个协议的约束，例如它是否只能用于 class 类型。

### 函数元数据

除了一些通用数据，函数类型元数据还保存这个函数的签名信息：参数和返回值类型元数据，调用惯例，参数所有权惯例，以及这个函数是否会抛异常。

## 3. 命名改编

命名改编用来生成全局唯一符号，外部（公开）符号，内部符号以及隐藏符号都可以应用这个方法，但只有外部符号的命名改编规则才是 ABI 的一部分。

目前命名改编规则里有一些 corner case 需要在 ABI 稳定之前解决。例如，对于泛型和协议来说，我们需要确定一个规范来允许顺序无关的命名改编。命名改编的主要目的之一是减少符号的大小。例如，在命名改编的时候我们不应该区分结构体和枚举类型，这可以给库的演进提供更大的自由度。

## 4. 调用惯例

> 标准调用惯例：这里指 C 语言在特定平台的调用惯例
> Swift 调用惯例：指 Swift 语言代码间的调用惯例

Swift 运行时使用标准调用惯例，当然在实际过程中可能会有部分修改。调用惯例稳定性与公共接口有关，对于内部调用，Swift 编译器可以自由选择调用惯例。

### 寄存器惯例

* 被调用方存储寄存器：如果一个被调用函数希望修改寄存器的值，那么它必须在返回之前恢复之前的值
* scratch 寄存器：与上面相反（？）

// To be continue...

## 5. 运行时

Swift 为编译后的代码提供了一个运行时环境，编译器会综合考虑内存惯例和运行时类型信息等情况来生成调用代码，此外，运行时还提供了底层的反射接口以便于标准库或者部分有需求的用户调用。

每个运行时函数都需要考虑到它的存在目的和运行行为：

* 如果一个函数是必须存在的，那么就应该精确的设计它的语义信息并且确保这个接口的存在
* 如果不是必须存在的，我们需要考虑修改，删除或者替换掉这个接口，并指定新的语义信息

运行时还要负责在运行时创建类型元数据访问接口，库的演进一般来说会通过使数据和元数据更不透明的方式引入新的需求类别，此外，所有权语义也可能会需要新的运行时接口或者修改当前的接口。

## 6. 标准库

任何在 ABI 稳定后发布的标准库都应该被未来的版本所支持来确保二进制库的稳定性。




