# 当我们在rRust中讨论“所有权”的时候，究竟是在说什么

> 本文主要参考：
> 
> - [rust reference](https://doc.rust-lang.org/reference/introduction.html)
> - [rust std](https://doc.rust-lang.org/stable/std/)
> - [rustc dev guide](https://rustc-dev-guide.rust-lang.org/about-this-guide.html)

不过这篇文章不是正式的文档，有些可能理解的不够深的地方就变成脑补了。如果有理解出现偏差的地方，还请多多包涵，欢迎pr。


## 值



在rust中一个**值(Value)**的一生大概会经历这些事情：

1. 从一个**表达式(Expression)**中**求值(Evaluation)**

2. 求值后值会被放在一个**位置(Place)**中

3. 这个值可能会被**借用(borrow)**，被修改

4. 最后这个值可能会被

   1. **移动(move)**——即从当前位置移动到另一个位置中

   2. **拷贝(copy)**——即从当前位置拷贝一份到另一个位置中

   3. **丢弃(drop)**——比如一个变量超出作用域的时候，就会调用析构函数把当前值给丢弃掉

   4. **泄露(leak)**——一个值到程序结束时既没有被丢弃，也没有失效，那么我们就可以说这个值被泄露了。

   5. **失效(invalidate)**：

      其实移动和丢弃也是使原值失效的手段之一，当值被移走时，在当前位置原来的值就失效了；当值被丢弃之后值也会失效。
   
      但还有其它手段，比如直接通过`ptr::write()`往当前位置覆写新值的时候，旧值就失效了；位置对应的内存直接被 **释放(deallocate)** 时，位置上的值也同样会失效了。
   
      总之当我们试图去访问一个失效的值时，就是**未定义行为(*U*ndefined *B*ehavior)**。




比如：

```rust
{
    let mut s = String::new(); // 求值得到一个空字符串，放到变量`s`中
    
    // 这里发生了几件事：
    // 1. 借用了`s`中的值，得到一个独占借用，放到临时的位置1中；
    // 2. `1`求值放到了一个临时的位置2中；
    // 3. `i32::to_string()` 借用了在临时位置1中的`1`去求值得到一个新的字符串，放到一个临时位置3中；
    // 4. `String::assign_add()` 移走了位置1和位置3中的值进行求值，得到`()`。
    //     而在求值过程中则修改了s的值。
    s += 1.to_string();
} // `s`超出了作用域，于是调用其析构函数，丢弃了其中的值。

```



## 位置



刚刚提到一个位置(place)的概念，对应着“内存里的位置”(location in memory)，其自身还有一些额外的属性（类型、可变性等）和状态。位置和值，对应C语言中“左值”和“右值”的概念。



常见的一些位置：

- **变量(Variable)** 声明会创建一个带名字的位置，
  - 局部变量的话则对应一片栈内存，
  - 全局变量对应一段静态存储区

- 表达式求值的过程中会产生一些**临时位置(Temporary)**，一般会对应一片栈内存（也可能会做静态提升）。
- 由 **全局分配器(Global Allocator)** 分配一片内存，作为位置。比如说`Box::new()`会分配一片堆内存作为位置。
- 函数参数、函数返回值也有对应的位置
- 一些值里又有一些小的位置，比如说一个结构体里的字段，数组中某个元素等。这些位置也会有自身的属性与状态。



而一个位置在运行的时候有几种状态：

1. 未初始化
2. 初始化了
3. 值被移走了（Copy类型的位置不存在这种情况）
4. 初始化了，其值被共享借用着
5. 初始化了，其值被独占借用着

编译器会跟踪这些状态用于分析或者插入一些代码。比如分析是否未初始化就使用（初始化分析）、是否尝试移动一个正在被借用的值（借用检查）等。如果这些状态静态时无法分析出来，则会插入一些代码在运行时来跟踪这些状态——这里引入了一些隐形的开销，所以建议还是不要写这样的代码：

```rust
let v; // v 未初始化
if condition {
    v = vec![1, 2, 3]; 
    // v 已初始化
}
// v 的状态初始化了或者还没有初始化


```

那么大概就会转换成这样的代码

```rust 
// -> 伪代码

let v;
let mut v初始化了 = false;

if condition {
    v = vec![1, 2, 3];
    v初始化了 = true;
}

if v初始化了 { // 当v超出作用域时会插入类似这样的代码
    mem::drop_in_place(&mut v as _);
}
```



除了状态，可变性也是一个位置的重要属性。只要一个位置可变的时候，我们就可以修改该位置中的值（与类型无关），否则就是ub（也有例外，UnsafeCell）：

* mut pattern修饰的变量，比如说`let mut x = 1` 我们就可以通过`x`的可变借用修改里面的值
* 一个结构体所在的位置可变，其字段的位置也是可变的





## 表达式



一开始说道，一个值是从一个表达式中求值得到的，那么接下来我们再聊聊表达式。

rust继承了一些函数式语言的一些特点，比如说“万物皆表达式”——除了常规的一些字面量是表达式、函数调用是表达式以外，`break`,`continue`,`return`这些控制流表达式是表达式，`{ let x = ...; ... }`,循环也是表达式。（注意：语句是块表达式组成部分，但是其本身并不是表达式）。但也有一些不那么函数式的地方，所有表达式求值都会得到一个值，但求值的过程中还可能会产生一些副作用，比如修改了其他值、求值时跳转了又或者是做了一些IO操作等。

 <br /> 

尽管在rust中有各种各样的表达式，但主要可以划分成两类表达式（其实还有第三种，叫Assignee Expression）：

- **位置表达式(Place Expression)**，求值后得到存放在该位置上的值。目前包括：

  * 局部变量
  * 全局变量
  * 解引用`*p`
  * 数组索引`arr[index]`
  * 字段引用`s.field`

  这些从一个值访问到某个位置的方式也叫**路径(Path)**。

- **值表达式(Value Expression)** ，求值后会得到一个新的值，然后这个值就会被存放在某个位置上。

  这里值的注意的是`const`定义的常量和`fn`定义的函数都是一个值表达式，每次使用的时候都会重新求值，并放在一个新的位置上。

 <br /> 

一些运算符可能会要求其操作数是某种分类的表达式，或者是不同分类的表达式行为会不同，比如说：

* 赋值运算符`=`要求左操作数是一个位置表达式

* 解引用运算符`*`要求操作数是一个位置表达式

* 借用运算符`&`和`&mut`在操作数为位置表达式时，就会直接构造一个该位置的引用，借用了该位置的值；

  而如果操作数为值表达式时，就会先创建一个临时位置来保存求值，再引用这个临时的位置，借用求值后的值。

* 如果一个位置表达式出现在函数调用的参数中时，就需要移走或拷贝该位置中的值（是否能移动和拷贝又有一些额外的规则）

* etc

这些导致表达式求值行为有差异地方称之为求值上下文，其实大概也可以分为 **位置上下文(place context)** 和 **值上下文(value context)** 两类，这里就不赘述了。



## 析构



刚刚讲了一个值的出生（表达式的求值），生活（在某个位置上被借用被修改），现在来讲讲它的死亡（析构）。



首先，析构函数行为由值的类型所决定，一个类型`T`的**析构函数(Destructor)**将会做这些事情：

1. 如果`T: Drop`，则先对其值调用`<T as Drop>::drop`
2. 然后递归地调用其自身所有字段的析构函数，比如
   * `T`是结构体，那么则按照字段的声明顺序调用字段的析构函数
   * `T`是数组，则按顺序调用元素的析构函数
   * `T`是Trait Object，则对其值调用内部实际类型的析构函数
   * ...

其中编译器将步骤1和步骤2粘起来成为完整的析构函数的操作叫做drop glue。

 <br /> 

目前自动调用析构函数的时机只有两个：

* 当一个变量或者临时位置超出其作用域时，就会对其值调用析构函数，将内部的值给丢弃掉；
  * 其中变量的作用域一般是从声明到`{}`结束
  * 临时位置的作用域一般从表达式的位置到`;`结束
  * 还有一些更复杂的作用域构成，可以接着看reference destructors这一节。
* 对一个位置进行赋值的时候，如果该位置已经有值的时候，则先调用析构函数，再将右操作数的值移动到该位置。

但仅有这两条规则*不能确保所有值的析构函数在其失效前调用*。这就是为什么rust事实上对内存泄露毫无办法的原因。。

强调一点，这些析构函数调用的时机，在进行分析检查之前就已经确定下来了，所以借用检查分析出来的lifetime不会影响一个值的析构时机。而且应该反过来说，借用检查器需要根据析构顺序去分析借用关系。



除了自动析构的时机以外，当编译器决定不下来的时候，或者想自己丢弃一个值的时候，可以通过`std::mem::drop_in_place`来手动调用其析构函数（一般搭配`std::mem::forget`, `ManuallyDrop`来使用）。





## 借用



最具rust的特色值莫过于 **借用(borrow 此处作名词)** 了，借用的值*是一段内存的引用，借用了对应位置里的值*。



借用检查器，会根据你的代码，分析出每个借用的借用范围（第二个借用作动词-. -），这个借用范围就是所谓的**生命期(lifetime)**，rust中用`'a`来表示。不过这个分析出来的借用范围并不是实际的范围，而是会稍微“长”一点，历史上rust有着几种生命期的推导方式：

1. 在1.31之前，生命期被推导成与借用本身的作用域同样的范围，也就是所谓的"lexical lifetime"，当时稍微写点复杂的代码都会被借用检查器给蠢到（
2. 而在1.31至今，借用检查器则根据rust的控制流图MIR推导出的 **非词法生命期(*N*on-*L*exical *L*ifetime)**，和作用域再无瓜葛（也还有，有个包含关系）。比以往精确不少，大大降低了rust的编码难度。虽然大多数时候都可以无视生命期了，但对于某些更抽象点的代码还是无能为力（比如经典的`Entry` API，和目前写不出来的`LendingIterator`的Filter组合子）
3. 未来（？），目前rust的类型系统工作组还在研究下一代的借用检查器polonius(polinius.next)，都能很好地覆盖到NLL解决不了的case。所以我还是蛮期待的。

这里一个常见的误解，很多初学者都会将生命期和作用域混为一谈。那么我希望大家看到这里应该要能将思维转变过来了。

 <br /> 


借用检查除了检查生命期期间值是否失效以外，对于目前的两种共享借用`&`和独占借用`&mut`，还有不同的检查项：

* 允许存在多个共享借用
* 被共享借用的值不允许修改
  * 有特例，`UnsafeCell`。允许通过共享借用来修改值。这种共享借用可变性也叫 **内部可变性(Interior Mutability)**。`UnsafeCell`的值本身允许修改，与其所在的位置的可变性无关，Interior的含义不是内外的内，而是本质、内在的内。
* 只有标记为mut的变量才能被独占借用
* 在独占借用期间，不允许同时存在其它借用——noalias
  
  目前也有特例，但似乎并没有公开，有兴趣可以参考：
  - 目前的Unpin hack: [exclude mutable references to !Unpin types from uniqueness guarantees](https://github.com/rust-lang/miri/pull/1952)
  - Share Aliasing?: [unsafe_cell_mut](https://hackmd.io/@CV5q1SRASEuY8WfOgd_3iQ/BkmQIn7Bs)

 <br /> 

虽然大多数生命期都能在编译期间推导出来，但在rust中我们可以写出两种生命期，一种是`'static`，一种在泛型参数列表中`'a`。后者是接口上的一些必要信息，甚至能引入一些额外的检查。

```rust
// 喜闻乐见的例子
// 如果去掉接口名，参数名，实现，调用者能判断这三个借用之间的关系吗？
fn longer(long: &str, short: &str) -> &str { long }

// 这时候则需要通过手动标记生命期，来表示几个借用之间的关系
fn longer<'long, 'short>(long: &'long str, short: &'short str) -> &'long str
where
    'long: 'short // 'long outlive 'short
{ long } // 编译器再检查实现是否满足接口约束
```

rust是个严格区分接口和实现的语言

* 调用者从接口中得知参数和返回值之间的关系，编译器检查调用是否满足接口所描述的条款
* 实现者通过接口声明，额外引入一些条件来实现逻辑。

当然，其实在实际编码的过程中，就我体验是比较少需要自己手动标记生命期的。因为有一些明确的**生命期消除规则(lifetime elision)**，在不标记生命期的情况下约定了一些参数和返回值的关系。





## 类型

我们再来讲讲类型。在rust中，所有的表达式、值与位置都会指定一个类型，类型赋予了他们意义，他们都不能脱离于类型来讨论：

* 类型之于表达式来说，决定了其求值的行为，得到何值。
  比如当我们写下`a + b`的时候，只有知道了`a`和`b`的类型，才能得知这个表达式的含义以及如何求值。

* 类型之于一个值来说，决定了其使用方式，以及析构行为。
  比如当我们使用一个值`a`的时候，只有知道了`a`的类型，才能知道`a`能否使用，能否作为参数应用于某个函数中。

* 类型之于一个位置来说，决定了其在内存中的**布局(Layout)**，包括**大小(Sized)**与**对齐(Align)**。
  比如当我们声明一个局部变量`x`的时候，只有知道`x`的类型，才知道如何分配内存给`x`。

 <br /> 

rust中有三种比较特殊的类型，一种是零大小类型(ZST)，一种是`!`和“无变体枚举”(Zero-variant Enums)，一种是动态大小类型(DST)

* 零大小类型指的是那些大小为0的类型。分配器和一些数据结构常常需要为ZST进行特化，比如`GlobalAlloc::alloc`就要求只能为非零大小的那些布局分配内存；而数据结构则要考虑ZST是否要进行分配内存的操作。值得注意的是，ZST虽然0大小，但对齐不一定是0，比如`[T; 0]`的对齐是`T`的对齐，意味着`[T; 0]`的地址也要是`align_of::<T>()`的倍数。

  ```rust
  println!("{:p}", Box::new([1i32, 0])); // 输出 0x4
  ```

* `!`和“无变体枚举”，是一种“求值永远都拿不到结果”的类型，比如`panic!()`, `break`等表达式，他们总会在求值的中途中断，永远得不到结果——当然也是一种零长类型。

* 动态大小类型，我们也会经常遇到，比如trait object类型`dyn Trait`，和切片类型`[T]`（组合之后得到的`Mutex<[u8]>`之类的也是DST）。

  这些类型的布局甚至析构函数在运行时才确定，在访问其值的位置时，就需要带上一些**元数据(Metadata)**，这些元数据就和指针放在一起：

  ```rust
  // std::mem::Pointee
  pub trait Pointee {
      // pointee的元数据的类型
      type Metadata: Copy + Send + Sync + Ord + Hash + Unpin;
  }
  
  impl<T> *const T {
      // rust中指针都可以分为两部分，一部分是数据指针，一部分是Pointee的元数据
      // 
      //  ┌─────────────────────────┐
      //  │ the value of DST in mem │
      //  └▲────────────────────────┘
      //   │
      //  ┌┴────────┬────────────┐
      //  │ pointer │  metadata  │
      //  └─────────┴────────────┘
      //
      pub fn to_raw_parts(self) -> (*const (), <T as Pointee>::Metadata)
  }
  ```

  那么目前就只有三种类型的元数据，`()`, `DynMetaData<dyn trait>`, `usize`:

  * 对于固定大小类型来说，他们的元数据类型为`()`，于是他们的指针就叫**窄指针(Thin Pointer)**
  * 对于trait object来说，他们的元数据类型为`DynMetadata<Self>`，是一个虚表指针，包含了大小、对齐、析构函数以及trait中方法的函数指针。
  * 对于切片类型`[T]`，他们的对齐和析构函数都是确定的，于是他们的元数据仅需要一个大小信息，类型为`usize`。

  * 而对于复合的DST来说，他们的元数据类型则继承自内部的DST，比如说`<Mutex<[T]> as Pointee>::Metadata == usize`。

  指向DST的指针因为包含了额外的元数据，所以也称之为**宽指针(Wide Pointer)**。



## trait

刚刚说到，类型决定了表达式、值、位置的含义与行为，那么如何定义这些的含义与行为呢？答案是通过trait，trait是rust类型系统中不可或缺的一部分。

* 编译器通过trait来为类型附加上一些语言内建的语法规则与语义，比如说`Copy`, `Send`, `Sync`, `Unpin`, `Fn`等。
* 用户则可以通过trait来为类型定义一些额外的行为，通过trait来对外暴露接口。



不过trait的详细用法这里就不展开了，这篇文章基本已经把我想说的都写下来了（

<br />

***



等一下，所以，所有权在哪呢？

——所有权其实并不存在，在语言里面并没有一条叫做所有权的规则，所有权的概念只存在于程序员的意识里。这种意识反映到代码里面就是一些*责权分明*的代码，会明确各个值的使用范围，以及值与值之间的关系。

rust的语言规则和编译器则会强迫你思考清晰这些关系，否则整个程序就会变得很糟糕，很难用——当然rust也会帮助你去思考这些事情。


<br />

>
> 
> 其实在drop check的[rfcs](https://github.com/rust-lang/rfcs/blob/master/text/0769-sound-generic-drop.md#when-does-one-type-own-another)中提到了所有权规则，
>
> > The definition of the Drop-Check Rule used the phrase "if the type owns data of type D".
> > This criteria is based on recursive descent of the structure of an input type E.
> > - If E itself has a Drop implementation that satisfies either condition (A.) or (B.) then add, for all relevant 'a, the constraint that 'a must outlive the scope of the value that caused >  the recursive descent.
> > - Otherwise, if we have previously seen E during the descent then skip it (i.e. we assume a type has no destructor of interest until we see evidence saying otherwise). This check prevents infinite-looping when we encounter recursive references to a type, which can arise in e.g. `Option<Box<Type>>`.
> > - Otherwise, if E is a struct (or tuple), for each of the struct's fields, recurse on the field's type (i.e., a struct owns its fields).
> > - Otherwise, if E is an enum, for each of the enum's variants, and for each field of each variant, recurse on the field's type (i.e., an enum owns its fields).
> > - Otherwise, if E is of the form `& T`, `&mut T`, `* T`, or `fn (T, ...) -> T`, then skip this E (i.e., references, native pointers, and bare functions do not own the types they refer to).
> > - Otherwise, recurse on any immediate type substructure of E. (i.e., an instantiation of a polymorphic type `Poly<T_1, T_2>` is assumed to own T_1 and T_2; note that structs and enums do not fall into this category, as they are handled up above; but this does cover cases like `Box<Trait<T_1, T_2>+'a>`).
>
>
> 以及[`PhantomData`](https://doc.rust-lang.org/stable/std/marker/struct.PhantomData.html)中也有所谓own的说法。
>
> > Zero-sized type used to mark things that “act like” they own a T.
> 
> 不过由于各种历史原因，事实上Rustc目前并没有真正的实际应用这些规则。（目前有个[PR](https://github.com/rust-lang/rust/pull/103413)正在跟进这个事，merge之后我再来修改这篇文章吧）
>