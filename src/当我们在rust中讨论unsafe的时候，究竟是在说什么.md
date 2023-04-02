# 当我们在Rust中讨论"unsafe"的时候，究竟是在说什么



> 这篇文章主要参考了这些资料：
> 
> * [Rustonomicon](https://doc.rust-lang.org/nomicon/intro.html) 《rust死灵书》rust unsafe必读
> * [Rust reference](https://doc.rust-lang.org/reference/unsafety.html) unsafety一节
> * [unsafe code guide line](https://github.com/rust-lang/unsafe-code-guidelines) rust unsafe代码指引工作组
> * [llvm pointer-aliasing-rules](https://llvm.org/docs/LangRef.html#pointer-aliasing-rules) 目前rust的引用的别明规则遵守llvm的指针别明规则
> * [RFC3128](https://rust-lang.github.io/rfcs/3128-io-safety.html) io safety
> * [Aliasing rules for Box<T>](https://github.com/rust-lang/unsafe-code-guidelines/issues/258) box的别名规则
> * [`MutexGuard<Cell<i32>>`must not be Sync](https://github.com/rust-lang/rust/issues/41622) 标准库中曾发生过的因错误实现`Sync`导致的事故
> * [PartialEq, Eq, PartialOrd and Ord should be unsafe traits](https://github.com/rust-lang/rfcs/issues/926) skiplist需要正确的全序关系
> * [RFC1066](https://rust-lang.github.io/rfcs/1066-safe-mem-forget.html) `std::mem::forget`将不再标记为`unsafe fn`
> 
> 
> 
> 知乎用户@Nugine的几篇关于unsafe的文章：
> 
> * [Unsafe Rust 随堂小测（一）](https://zhuanlan.zhihu.com/p/532496013)
> * [Unsafe Rust 随堂小测（二）](https://zhuanlan.zhihu.com/p/532937544)
> * [Unsafe Rust 随堂小测（三）](https://zhuanlan.zhihu.com/p/533259640)
> * [Unsafe Rust 随堂小测参考答案](https://zhuanlan.zhihu.com/p/533815162)





***



在往下读之前，我们先忽略掉“safe”和“unsafe”的字面意思，只关注于他们实际的使用的场景与作用。unsafe并不意味着“不安全”；以及这里的safe也不是secure所指的那种安全。



## safe与unsafe的分离

rust 通过`unsafe`关键字，把rust语言分成safe/unsafe两部分，**希望通过类型系统保证在safe rust下不会产生未定义行为(*U*ndefined *B*ehavior)**。加粗部分便是rust所追求的最根本的目标（也是一种理想），在用safe rust时永远不需要担心空指针，UAF等等错误，这种性质通常叫做类型系统的**可靠性(soundness)**。

但现实是，我们通常要与运行时或者与rust以外的系统打交道，rust的类型系统并不能验证这部分的正确性，这时就需要编写unsafe的代码了，而且需要程序员自己保证安全性。另外目前的类型检查会出现假阳性的情况，即事实上不会造成ub，但类型检查器却将代码拦了下来，这时候也可以通过unsafe的代码去绕过类型系统的检查。

其实这就意味着，rust把类型系统检查不了的问题丢给了unsafe rust，或者说，丢给了程序员。



~~在我有限的认知内，在语言内划分safe/unsafe的，rust应该是独一份。（孤陋寡闻了，其实Java/C#这俩语言都有）~~ 这使得rust在可以无压力使用高级抽象（通过safe rust）的同时也可以像C/C++那样无门槛对接bare metal（通过unsafe rust）。

在对待unsafe这件事情上，现有语言上其实有两种极端，一种是C/C++的“只有”unsafe，类型系统不会保证任何安全性；另一种类似是Java这种“全safe”，将几乎所有安全问题丢给运行时去处理。后者对比前者就缺乏了很多做低级抽象的能力——这里点名批评Haskell，不靠ffi不靠运行时内置实现，都使用不了“线性内存”，都定义不出数组。。。



## unsafe 的语法以及用法

于是`unsafe`关键字的含义便是：

* 对于接口声明方：声明该接口有一些编译器无法检查的契约，反过来接口的实现则可以无条件使用这些假设。含义像"assume that xxx"
* 对于接口的适用方：声明程序员已经检查过满足了unsafe接口的契约。这里的含义更像是“trust me”

这两种含义总是成对出现的，目前rust中有两对这样的`unsafe`：

* `unsafe fn`(与一些unsafe的操作) 与 `unsafe {}`

  unsafe的操作，目前只有几种：

  * 裸指针解引用
  * 读写可变全局变量或者外部全局变量
  * 访问`union`的字段

* `unsafe trait`与`unsafe impl`



### `unsafe fn`/`unsafe {}`

对于`unsafe fn`/`unsafe {}`，这里举个简单的例子：

* `ptr::read`是个`unsafe fn`，会将指针指向的内存按位拷贝出来
  
  ```rust
  pub unsafe fn read<T>(src: *const T) -> T;
  ```
  
  调用该函数需要满足以下条件（契约），如果违反了这些规则则是ub
  
  * 指针`src`是合法(valid)的
  
    不过指针合法的规则并没有完全决定下来，但有几个必要条件
  
    * 空指针是不合法的（包括对于ZST来说）
    * 指针地址开始到给定大小(`size_of::<T>()`)的内存范围都是已分配好的内存
  
  * 指针`src`是对齐了的
  
  * 指针`src`指向的值是已经初始化了的
  
* 简单地调用一下这个函数：

  ```rust
  let x = 12;
  let y = &x as *const i32;
   
  unsafe {
      assert_eq!(std::ptr::read(y), 12);
  }
  ```

  这里的调用是安全的，因为`y`满足了`read`的三个安全契约，即合法、对齐、值已初始化。



### `unsafe trait`/`unsafe impl`

而`unsafe trait`/`unsafe impl`这一对则微妙一点。但意思也是类似：`unsafe trait`声明需要满足某些契约，而`unsafe impl`则声明程序员已经检查并满足了这些契约。`unsafe impl`一个坏的实现，可能就会引起某个依赖了该trait的安全契约的`unsafe {}`发生ub（但ub不一定立刻发生）。

比如声明一个`unsafe trait Foo {}`引入一个安全契约xxx，那么`T: Foo`相当于引入了这条安全契约，`unsafe {}`就安全地使用这条规则去调用`unsafe fn`：

```rust
/// # Safety
/// * xxx
unsafe trait Foo { /*可以有实现，也可以没有*/ }

// `foo`可以是一个自由函数，也可以是`Foo`里的方法
fn foo<T>(t: T) 
where
    T: Foo, // 引入安全契约xxx
{
    // 使用安全契约xxx
    unsafe {
        // unsafe_foo这个函数需要安全契约xxx，否则是ub
        unsafe_foo(x);
    }
}
```

那么当用户为某个类型实现`Foo`的时候，就需要检查并保证满足安全契约xxx。否则就会导致`foo`产生ub，因为相当于破坏了`foo`内部`unsafe_foo`的调用的安全契约。

```rust
struct Bar;

// 这里需要实现者自己保证满足安全契约xxx
unsafe impl Foo for Bar {}

fn main() {
    // 如果Bar正确实现Foo，没有问题
    // 否则这里就会ub
    foo(Bar);
}
```





`Send`/`Sync`是便是标准库中比较经典的一组unsafe trait，需要结合标准库中的一些并发原语来理解。

* `Send`要求实现者保证`T`可以跨线程移动

* `Sync`要求实现者保证`&T`可以跨线程共享（即`&T: Send`）——必要条件是，通过`&T`对`T`的操作（读和写）需要进行线程间同步

* `thread::spawn()`是rust标准库中的并发的原语，会创建一个新线程执行闭包，要求闭包满足`Send`。这里会信任闭包里捕获的数据，程序员已经正确实现了`Send`。

  ```rust
  pub fn spawn<F, T>(f: F) -> JoinHandle<T>
  where
      F: FnOnce() -> T, 
      // spawn内部某个unsafe block依赖了 
      // 由`F: Send`引入的 “F可在线程间安全移动” 的安全契约
      F: Send + 'static,
      T: Send + 'static,
  ```



如果用户实现了一个坏的`Sync`就可能使得`spawn`产生ub。虽然下面的调用都是safe的，但是坏的实现破坏了调用`spawn`的契约，使得`spawn`内部某个需要到这个契约的`unsafe {}`发生ub。

```rust
#[derive(Debug, Default)]
struct Inc(Cell<i32>);

impl Inc {
    fn inc(&self) {
        // 没有做线程间数据同步
        self.0.set(self.0.get() + 1)
    } 
}

// `Inc`提供的接口没有做数据同步，违反`Sync`的契约
unsafe impl Sync for Inc {}

fn main() {
    let cell = Inc::default();
    
    // spawn的scoped版本，不要求'static
    thread::scope(|s| {  
        // 线程1
        s.spawn(|| {
            for _ in 0..100 {
                // Inc因为实现了Sync，于是&cell可以在两个线程共享
                cell.inc();
            }
        });
        
        // 线程2
        s.spawn(|| {
            for _ in 0..100 {
                // 与线程1中的数据竞争 产生ub
                cell.inc();
            }
        });
    });
    
    // 可能是任意值！
    dbg!(cell);
}
```

其实标准库历史上也出现过错误实现`Send`/`Sync`导致接口unsound的事故，[`MutexGuard<Cell<i32>>`must not be Sync](https://github.com/rust-lang/rust/issues/41622)。



实践上，我们还是会尽可能少地为一个trait标记unsafe，这会为实现者带来比较大的心智负担（比如说刚刚提到的事故），而且unsafe的范围很难评估。我们大多数情况会允许trait的实现者给出一个坏的实现，转而在依赖该契约的地方做对这种情况进行防御（不过`Send`/`Sync`这种性质没法防御，就只能提供为unsafe trait了），**只要保证库提供出去的api是可靠的就可以了。**

非得标记unsafe的时候也尽可能缩小unsafe的范围，比如在具体会发生ub的函数上标记unsafe。

比如说标准库中其实有些trait会带上一些额外的要求，`Ord`/`Hash` trait等。比如`Ord`*建议*实现的类型是*严格全序*的，需要满足反对称性、三歧性和完全性（必要条件）：

1. `a.cmp(&b) == Greater` 当且仅当 `b.cmp(&a) == Less`

2. `a.cmp(&b) == Equal` 且 `Equal == b.cmp(&a)`

3. `a.partial_cmp(&b) == Some(a.cmp(&b))`

1和2等价于反对称性 + 三歧性，3等价于完全性。除此之外还有好些规则，这里不一一列举。但这些规则并没有由类型系统所保障，也没有标记为`unsafe trait`让实现者严格保证规则，实现者便可以随意实现一个`Ord`不满足严格全序的约定。

有些算法是依赖全序的一些性质的，比如说排序，B树，跳表等。排序姑且一个坏的`Ord`实现不会导致ub，但B树和跳表会。标准库里的`BTreeMap`就对坏的`Ord`实现进行了防御，保证了`BTreeMap`提供的接口不会产生ub，是可靠的。

最终来看，标准库内提供的与`Ord`相关的接口都不会产生ub，于是`Ord`最终便提供为了safe trait。不过标准库外的实现就没那么幸运了，比如说跳表的实现，依赖了全序的性质，而其实现又没办法完全防御不满足性质的情况，就只能将其接口标记为`unsafe`了：[`skiplist::OrderedSkipList::sort_by`](https://docs.rs/skiplist/0.4.0/skiplist/ordered_skiplist/struct.OrderedSkipList.html#method.sort_by)。（不过现在这个接口没有直接依赖`Ord` trait了）

```rust
/// The ordered skiplist relies on a well-behaved comparison function. 
/// Specifically, given some ordering function f(a, b), it must satisfy the following properties:
///
/// * Be well defined: f(a, b) should always return the same value
/// * Be anti-symmetric: f(a, b) == Greater if and only if f(b, a) == Less, and f(a, b) == Equal == f(b, a).
/// * By transitive: If f(a, b) == Greater and f(b, c) == Greater then f(a, c) == Greater.
///
/// Failure to satisfy these properties can result in unexpected behavior at best, 
/// and at worst will cause a segfault, null deref, or some other bad behavior.
pub unsafe fn sort_by<F>(&mut self, cmp: F) 
where
    F: 'static + Fn(&T, &T) -> Ordering, 
```





## 目前rust认为哪些东西是unsafe的

虽然从概念上，我们可以将任何编译器检查不了的条件往`unsafe`里套，但历史上，rust曾划了一条界限，声明`unsafe`只管内存安全，其它的都不管。不过前不久，标准库里引入了一个新的概念，**IO安全**，也通过`unsafe`来划定界限。

我们之所以要保证内存安全，不仅仅是因为要避免ub，本质上是也在避免ub带来的*非局部性*问题。我们总是希望一段代码的行为总是有限的、可预测的；如果更进一步的话就是要求在条件相同的情况下，同一段代码的行为是一致的。这样的性质，我姑且称之为代码的局部性原则。而如果违反内存安全的约定的话，则就会轻松破坏掉代码的局部性原则。正如上面一节所举的`Inc`一样，一旦发生数据竞争，程序的行为将无法预测，甚至可以执行任意代码（这就是所谓内存漏洞），这正是我们要避免的问题。

而IO同样也会产生类似的问题。比如说直接通过原式句柄读写程序其它部分的所需要的数据，就会产生“鬼魅的超距作用”，甚至执行任意代码，这破坏掉了代码的局部性，这便是我们想要避免的情况。而且在某些情况IO的错误也可能导致内存安全事故（比如通过mmap与别的进程打交道）。这种bug的严重程度并不亚于内存安全带来的问题。于是现在标准库便引入了IO安全的概念，并通过类似“所有权”和“借用”的方式来管理文件句柄。

（说实话我觉得halt safety也该给unsafe管）

![memory safety](./images/safety.svg)





## 目前rust不认为哪些东西是unsafe的

不过目前有一些行为在rust中不被认为是`unsafe`的，比如：

1. 死锁
2. 泄露：主要指`std::mem::forget`
3. 取裸指针：比如`std::ptr::addr_of!`或者引用转裸指针
4. 算术溢出：加减乘除模左移右移等
5. 代码逻辑错误：比如`Ord`, `Hash`等错误实现导致排序或者哈希等算法产生奇怪现象甚至panic等

他们在rust中都不是ub，而且他们与标准库中其它safe api组合起来也不会造成ub，所以最终他们还是获得了rust世界中的绿码——safe了，没有必要unsafe的地方便无需unsafe。尽管他们有一些确实是bug，也是我们应该去避免的。

虽然这几种情况现在都是safe的，但我想澄清一下，现实却可以“因为这些错误”而导致ub，比如旧版的[scoped thread](https://github.com/rust-lang/rust/issues/24292)会因为泄露而产生ub，于是这个api便被踢出了标准库，并且在头上“刺字”`unsafe`；又或者上面提到的跳表，也需要加上`unsafe`，让程序员去保证所需要的条件，否则也会产生ub。



值得注意的是算术溢出，事实上在C++中一些算术溢出是[ub](https://en.cppreference.com/w/c/language/operator_arithmetic#Overflows)。不过在rust中，算术溢出的行为[是有定义的](https://rust-lang.github.io/rfcs/0560-integer-overflow.html)。分为开启`debug_assert`选项和不开启两种情况：

* 开启`debug_assert`情况下算术溢出会产生panic
* 关闭选项时（比如说release）
  * `+`, `-`, `*`会按补码的方式来工作
  * `x/0`, `x%0`, `MIN/-1`, `MIN&-1`会panic
  * `x<<n`或`x>>n`，当`n`大于`x`的位数`N`时，会先对`n`模`N`再进行运算

同时，编译器会尽可能地在编译时检查出来并报错：

```rust
fn main() {
    i32::MIN%-1;
}
/* 编译报错，当然只是lint级别的报错，类型系统检查不了
error: this operation will panic at runtime
 --> src/main.rs:4:5
  |
4 |     i32::MIN%-1;
  |     ^^^^^^^^^^^ attempt to compute the remainder of `i32::MIN % -1_i32`, which would overflow
  |
  = note: `#[deny(unconditional_panic)]` on by default
*/
```

而且，如果确实明确希望通过补码的方式来处理溢出的，std也提供一套[`Wrapping`](https://doc.rust-lang.org/stable/std/num/struct.Wrapping.html)api来处理；或者明确需要知道是否溢出，也可以通过类似`i32::checked_add`或者`i32::overflowing_add`这样的api来进行算术运算。

panic或者编译报错，体现了算术溢出确实是个错误，需要程序员去避免；而明确规定溢出的错误行为，从而使得接口无需使用unsafe，则体现rust会尽量去提高接口的易用性；但当你真的需要到某一种行为的时候，rust也提供给你。这一波倒腾，其实很rusty——同时兼顾[Reliable](https://rustacean-principles.netlify.app/how_rust_empowers/reliable.html), [Supportive](https://rustacean-principles.netlify.app/how_rust_empowers/supportive.html), [Versatile](https://rustacean-principles.netlify.app/how_rust_empowers/versatile.html).



最后来谈谈讨论度很高的死锁和泄露两个问题。。。首先，从理论上我们没有算法能静态检查出任意代码是否发生死锁或泄露，根据rice theorem，这都等价于停机问题；其次，rust的类型系统没办法限制这两种情况的出现，比如对于泄露，只通过当前的RAII机制，无法保证所有值的析构函数运行（比如造一个成环引用计数），而且通过safe api泄露的方式还不止这种（死锁同理）。——rust最终将`std::mem::forget`的`unsafe`给去掉了。

可以怪目前编程语言研究对这些问题研究得还不够深入，也可以怪rust目前的机制还比较菜，但我个人倾向于认为死锁和泄露是逻辑问题。比如，检查泄露，相当于在问“一些资源何时不再使用”，这其实是需要通过代码逻辑去描述的，而当程序员自己也不清楚一个资源应该什么时候释放的时候，编译器便更不清楚了。

（热知识，有GC也会有泄露）





## 什么是UB，目前有哪些？

 **划分safe/unsafe最根本的目的，就是在safe rust中避免ub。** 那么ub作为这里最核心的概念，究竟是什么呢？


<div class="note">

UB其实是一个对所有语言都通用的概念：
<br/>
<br/>
首先，所有程序语言（包括rust）都有一台抽象的机器来运行代码，这台机器将解释语言的每一个语句的行为，以及行为如何改变这台机器的一些状态。我们称这种抽象的机器为程序语言的**操作语义**。一般我们会以定义好的操作语义作为一个程序语言的标准行为，然后所有的编译器以及解释器都要对齐这个标准。
<br/>
<br/>
然后，编译器和程序员之间会约定一些*契约*，这种契约一般会定义在在语言标准中（不过rust还没有语言标准）。如果程序员遵守了这些契约，编译器则保证编译出来的代码在真实硬件上的行为与原代码在抽象机器的行为保持一致。反过来，如果程序员没有保证契约，编译器则什么都不保证，生成的东西就是垃圾。可能会恰巧看似正确执行，可能会执行任意代码，甚至不需要是个可执行文件。对于不遵守契约的情况，我们就称之为**未定义行为(*U*ndefined *B*ehavior)**，简称UB。
<br/>
<br/>
不过值得注意的是，这些契约在不同平台上不一样，这也导致不同平台上的UB也不完全相同。

</div>



让我们回到rust，rust目前定义了以下[未定义行为](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)，不过因为rust的操作语义还没完全定义下来（[在努力了](https://github.com/rust-lang/rfcs/pull/3346)，目前的工作有stacked borrow，是目前最好的一个借用模型；MIRI，基于MIR的解释器；现在又正在研究MiniRust），以下列表还不完整，而且里面的一些规则还有一些修改的空间：

1. 数据竞争
2. 在悬垂或未对齐的裸指针上解引用(`*expr`)
3. 破坏指针别名规则（如果两个指针或引用指向的内存有重叠，则两者互为**别名(alias)**）。目前`Box<T>`, `&mut T` 和 `&T`基本遵守llvm的别名模型([scoped noalias](https://llvm.org/docs/LangRef.html#noalias))，除非`&T`中包含了`UnsafeCell`。`Box`和引用在他们还“活着”的时候不允许悬垂，不过“活着”的这个范围还没有定义，但至少满足：
   1. 对于引用，他们不会活得比他们的作用域长，具体长度由borrow checker给出
   2. `Box`和引用传入函数参数或从函数返回时，是活着的
   3. 引用（非`Box`），作为函数参数时在整个函数体内是活着的（同样的除非`&T`包含`UnsafeCell`）
4. 修改不可变数据，比如说没有`mut`模式修饰的变量，或者从`&T`访问的数据（包含从`&T`转化的裸指针，其指向的数据也是不可变的），除非这些数据被包在了`UnsafeCell`中。
5. 违反一些编译器的内建函数的约定（类似`transmute`这些）
6. 执行平台不支持的代码
7. 调用一个没有遵守ABI的函数，或者在不支持unwind的ABI中unwind
8. 产生一个不合法的值（类型中不存在的值），比如说
   1. `bool`中除了`true`(1)或`false`(0)以外的值都是非法值
   2. A discriminant in an `enum` not included in the type definition.（这个不知道怎么翻译。。）
   3. 空的函数指针
   4. `char`超过`char::MAX`的值（友情提示，`char`虽然占4个字节但只有21位有效）
   5. 产生`!`类型的任何值都是非法的
   6. 未初始化内存中的整数、浮点数、裸指针、`str`都是非法值
   7. 悬垂、未对齐、指向非法值的引用或`Box`都是非法的
   8. 元数据非法的宽指针都是非法的
   9. 包含不合法值的自定义类型的值也是非法的。

9. 错误使用内联汇编



这份UB列表很大一部分都是内存相关的问题，而在safe rust中则保证了(almost)不会产生这些UB，于是便声称——rust解决了内存安全问题。（热知识，Java也（几乎）没有内存安全问题）

这里的规则其实在实践起来还是挺微妙的，标准库数次翻车。更不用说那种以为很了解底层原理就以为不遵守规则的行为了。（话说历史上`actix`的原作者因为代码里有ub，但作者因为觉得反对者举不出来ub的用例于是不肯做修改，被rust社区喷到直接退圈……当然里面有人的语言比十分有攻击性）





## 编写 unsafe rust还要注意什么

其实刚刚多多少少都提到了一些，这里再强调一下（只是我个人能想到的一些）



* safe和unsafe的不对称信任

  * 一方面safe会无条件信任unsafe的代码都已经正确编写，比如`spawn`会相信你的`unsafe` impl是正确的
  * 另一方面unsafe不可以信任safe的声称但没有保证的性质，比如`BTreeMap`中unsafe代码不能信任`Ord`的实现一定是全序的，需要做防御。

  其实safe需要信任unsafe，unsafe不能信任safe挺反直觉的2333

* 如果不遵守unsafe的安全契约，带来的后果是不可预测的，影响范围是非局部的，甚至ub发生的地方不一定在`unsafe {}`中。

* 某些unsafe api安全的契约会互斥，当你将其中一个封装为safe api的时候，另外一个就不能封装成safe了。比如，pin project，这俩函数只能选一个封装为safe，否则api就unsound了

  * `Pin<&mut Struct> -> Pin<&mut Field>`
  * `Pin<&mut Struct> -> &mut Field`

* 进一步，在将一个unsafe api封装为safe api的时候，就要考虑是否和已有接口相悖，否则封装的api也是unsound的
* 有些unsafe api的安全契约，无法通过类型系统或者运行时检查保证的，就不能封装为safe api。（比如说目前RDMA，在rust中暂时无法封装为safe api）





## 补充：可靠性(soundness)和完备性(compeleteness)

这一小节请允许我以比较民科的口吻去描述。。

刚刚除了讲到了`unsafe`和ub，其实也引入了一个概念 **可靠性(soundness)**，这个单词也会经常出现在rust的讨论中。这个概念其实来自于数理逻辑学，用来描述形式系统中语法与语义之间的关系，大概说的是“满足语法的语句，语义也正确”；与之对应的一个概念叫**完备性(compeletenenss)**，表示“满足语义的语句，也满足语法”。



对于rust来说，我们一般讨论的是safe rust下的可靠性与完备性。“满足语法”指的是那些类型检查通过的程序，“满足语义”则指的是在rust抽象机中可解释的的程序。前者我们称之为Type check的程序，后者我们称之为Type safe的程序。那么可靠性和完备性在safe rust中大概的关系就是：

* soundness: type check -> type safe
* compeleteness: type safe -> type check

目前的rust做到了"amost soundness"，尽可能去做到"compeleteness"（比如通过提高类型系统的表达力，尽可能减少类型检查误报的情况）

不过这里有个比较悲观的理论，表达力足够强大的形式系统都无法同时做到可靠且完备（这其实就是数学中的哥德尔不完备定理了）





## 补充：Rust“虚假的”安全

> 参考资料
>
> - [`/proc/self/mem` and similar OS features](https://doc.rust-lang.org/stable/std/os/unix/io/index.html#procselfmem-and-similar-os-features)
> - [A soundness bug in std::fs](https://github.com/rust-lang/rust/issues/32670)


请品鉴

```rust
use std::fs;
use std::io;
use std::io::prelude::*;

fn main() {
    let i = 0;
    let j = &i as *const i32 as u64;
    // 通过文件"/proc/self/mem"访问内存
    let mut f = fs::OpenOptions::new().write(true).open("/proc/self/mem").unwrap();
    // 找到`i`的内存
    f.seek(io::SeekFrom::Start(j)).unwrap();
    // 向`i`写入[0xFF, 0xFF, 0xFF, 0xFF]
    f.write(&[0xFF; 4]).unwrap(); 
     // 输出-1
    println!("{i:?}");
}
```

这个例子展示了如何通过系统提供的api，读写任意内存，并且无需使用任意的unsafe操作。这对于rust来说显然是个ub，因为这样的程序破坏了对i这片内存的假设。
那是不是意味着文件系统的api就都要标记成unsafe呢？目前讨论下来是不需要的。这种通过“外部”方式造成的内存安全问题并不是rust的管辖范围，在程序的内部也不可能防范来自外部的“攻击”，可能来自操作系统，可能来自其它进程，甚至物理层面上的修改。

于是，就有这么一条结论：Rust的安全性仅承诺其自身的操作不会导致内存安全问题。rust只为rust编译器编译出来的代码负责（

> Rust的安全性，构建在一个不安全的世界之上，或者是一种“粉饰的太平”。


## 习题

给下面的impl一个合理的解释：

```rust
unsafe impl<T: Send + Sync + ?Sized> Send for Arc<T> {}
unsafe impl<T: Send + Sync + ?Sized> Sync for Arc<T> {}
```



大家有兴趣可以做一做知乎用户@Nugine的那几篇unsafe随堂测 2333，反正我在不看答案的时候都不能完全做出来。。