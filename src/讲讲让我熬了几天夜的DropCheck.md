# 讲讲让我熬了几天夜的Drop Check

> 收集到关于drop check的资料：
> 
> * [RFC0769](https://rust-lang.github.io/rfcs/0769-sound-generic-drop.html) 已被 rfc1238部分替代
> * [RFC1238](https://rust-lang.github.io/rfcs/1238-nonparametric-dropck.html) 目前实现的规则，其中例外规则的语法由rfc1327替代
> * [RFC1327](https://rust-lang.github.io/rfcs/1327-dropck-param-eyepatch.html) drop check的例外规则，目前正在推动稳定
> * [nomicon](https://doc.rust-lang.org/nomicon/dropck.html) drop check章节
> * [RFC1860](https://rust-lang.github.io/rfcs/1860-manually-drop.html) ManuallyDrop不会有drop glue
> 
> * 还有一个讨论话题[confusing dropck](https://rust-lang.zulipchat.com/#narrow/stream/122651-general/topic/.E2.9C.94.20Confusing.20dropck/near/310689272)
> 
> 补充
> - [PhantomData<T> no longer dropck?](https://github.com/rust-lang/rust/issues/70841) 目前不会析构的类型不会进行drop check。
> - [PhantomData: fix documentation wrt interaction with dropck](https://github.com/rust-lang/rust/pull/103413#issuecomment-1325099615) 原来目前官方对dropck还没有共识……如果到时这个PR合入之后再修改一下文章

***



## Drop与析构函数



在讨论之前先理一下[`Drop`](https://doc.rust-lang.org/stable/std/ops/trait.Drop.html)和析构函数的关系。`Drop::drop`是析构函数的一部分，可选实现，编译器通过drop glue得到一个完整的析构函数。

> 首先，析构函数行为由值的类型所决定，一个类型`T`的**析构函数(Destructor)**将会做这些事情：
>
> 1. 如果`T: Copy`或者`T == ManuallyDrop<_>`，则没有析构函数。
>
> 2. 如果`T: Drop`，则先对其值调用`<T as Drop>::drop`
>
> 3. 然后递归地调用其自身所有字段的析构函数，比如
>
>    1. `T`是结构体，那么则按照字段的声明顺序调用字段的析构函数
>
>    2. `T`是数组，则按顺序调用元素的析构函数
>    3. `T`是Trait Object，则对其值调用内部实际类型的析构函数
>
>    1. ...
>
> 其中编译器将步骤2和步骤3粘起来成为完整的析构函数的操作叫做drop glue，其产物为[`drop_in_place`](https://doc.rust-lang.org/stable/std/ptr/fn.drop_in_place.html)

没有析构函数的类型，在MIR阶段不会插入`DROP(var)`的指令，不会进行析构，于是也不会进行drop check

## 为什么要check？

众所周知，rust通过RAII来管理内存——值会在退出作用域的时候被析构。但按照目前的规定的析构顺序，比如变量是先声明先析构，就有可能发生后析构的值仍然借用着已经析构了的值的情况，从而有可能发生UAF(use after free)。

于是需要引入一个drop check：保证**一个值在析构时，其析构函数不会*访问*到一个悬垂引用`&'a _`所引用的值**。



与borrow check中一般的规则不同，borrow check在使用一个值的时候是不允许存在悬垂引用的，但析构函数有些情况是允许的——因为大多数类型drop glue产生的析构函数不会通过悬垂引用去访问内存，比如`(&str, String)`。从这个角度来说drop check的会比borrow checker的一般规则要弱一些。

另外borrow check也不能覆盖所有drop check所检查的情况。一些类型内部的析构顺序（比如数组会从前到后析构元素），并不会在MIR上体现出来，borrow checker无法进行检查。于是从这个角度来说，drop check还是borrow check的补充。



那么drop check 的具体规则是什么呢？



## 年轻人的第一个drop check

从一个例子讲起：

```rust
//! 例子1

struct Foo<T: Debug>(T);

impl<T: Debug> Drop for Foo<T> {
    fn drop(&mut self) {
        println!("dropping Foo with {:?}", self.0);
    }
}

fn main() {
    let _foo;
    let s = String::from("123");
    _foo = Foo(&*s);
}

/* 编译报错：

error[E0597]: `s` does not live long enough
  --> src/main.rs:16:23
   |
16 |     _foo = Foo(&*s);
   |                  ^ borrowed value does not live long enough
17 | }
   | -
   | |
   | `s` dropped here while still borrowed
   | borrow might be used here, when `_foo` is dropped and runs the `Drop` code for type `Foo`
   |
   = note: values in a scope are dropped in the opposite order they are defined
*/
```

报错的原因是：

1. 按照局部变量的析构顺序，`s`先析构于`_foo`，则`_foo`在析构的时候仍包含了一个悬垂引用`&'s str`；

2. `_foo`的析构函数会调用`<Foo<T> as Drop>::drop`，编译器不知道`drop`中是否通过`T`访问到了`&'s str`所引用的值（确实访问到了，通过`Debug`的方法），于是添加了一个约束：`T` (**strictly**) outlive `_foo`的作用域（记为`'_foo`）。

   此处`T == &'s str`，那么`T: '_foo` 等价于 `'s: '_foo`。

3. 借用检查器发现`'s`比`'_foo`短，然后报错。



不过如果我们将`impl Drop`给去掉，上面的代码就能正常编译了。因为编译器能明确地知道`_foo`的析构函数不会访问`&'s str`所引用的值，不会进行第二步的检查。



## strictly outlive？



刚刚加粗了**strictly**，是因为有这种情况：

```rust
//! 例子2

struct Foo<T: Debug>(T);

// 同样，如果不写这个实现，下面的代码仍然可以编译过。
impl<T: Debug> Drop for Foo<T> {
    fn drop(&mut self) {
        println!("dropping Foo with {:?}", self.0);
    }
}

fn main() {
    let tuple = (String::from("123"), None); // 元组的析构顺序是从前到后
    tuple.1 = Some(Foo(&tuple.0));
}
```

这段代码将和第一个例子产生同一个报错。但这种情况下，`tuple.0`和`tuple.1`在同一个作用域下（记为`'tuple`），而不是像第一个例子那样`_foo`的作用域包含`s`作用域。如果仅仅要求`T: 'tuple`，这段代码是能编译过的，然后`tuple.1`析构时就会使用到悬垂引用。所以这里drop check需要strictly outlive的断言。



不过strictly outlive这种断言用户目前没法写出来，仅存在于编译器内部。



那么第一和第二个例子引入了drop check的第一个规则：

> 记`foo: Foo<..., T, ...>`，其中`T`可以是个生命周期参数，也可以是个泛型参数。然后`foo`的作用域记为`'foo`，当`foo`被析构的时候：
>
> 1. 如果`Foo<..., T, ...>`  实现了 `Drop`，都有`T` strictly outlive `'foo` 。



## 睁一只眼闭一只眼的drop check

但这个检查会错杀一些本来安全的程序：

```rust
//! 例子3

struct Foo<T>(T, &'static str);

impl<T> Drop for Foo<T> {
    fn drop(&mut self) {
        // 没有访问`T`
        println!("dropping Foo with {:?}", self.1);
    }
}

fn main() {
    let _foo;
    let s = String::from("123");
    _foo = Foo(&*s);
}
```

这个例子会报与例子1相同的编译错误，因为根据规则1，在`_foo`析构时同样会引入`T: '_foo`的约束。但`Foo`的析构函数完全不会访问到已经析构掉的`s`，所以这份代码是安全的。于是rfc1327引入了豁免dropck的语法与规则：

```rust
//! 例子4

struct Foo<T>(T, &'static str);

unsafe impl<#[may_dangle] T> Drop for Foo<T> {
    fn drop(&mut self) {
        println!("dropping Foo with {:?}", self.1);
    }
}

fn main() {
    let _foo;
    let s = String::from("123");
    _foo = Foo(&*s);
}
```

这个例子就可以通过编译了。



这里多了一个特殊的attribute`#[may_dangle]`标注在泛型`T`上，表示`T`可以是个悬垂引用（或者是 一个可能访问到一个悬垂引用的类型），在dropck时就会跳过这个参数。另外此时`Drop`需要`unsafe impl`，因为实现者需要自己保证`drop`的实现中不会使用到悬垂引用。



于是目前drop check的规则是：

> 记`foo: Foo<..., T, ...>`，其中`T`可以是个生命周期参数，也可以是个泛型参数。然后`foo`的作用域记为`'foo`，当`foo`被析构的时候：
>
> 1. 如果`Foo<..., T, ...>`  实现了 `Drop`，都有`T` strictly outlive `'foo` 。
>
> 2. 除非`T`在实现`Drop`时标记了`#[may_dangle]`





## 怎么就出错了呢

现在我们来实现一个`MyBox`

```rust
struct MyBox<T> {
    ptr: NonNull<T>,
}

impl<T> MyBox<T> {
    fn new(x: T) -> Self {
        if size_of::<T> == 0 {
            // Allocator不允许为ZST分配内存
            return Self {
                ptr: NonNull::dangling(),
            };
        }
        
        unsafe {
            let layout = Layout::new::<T>();
            let ptr = alloc(layout);
            assert!(!ptr.is_null);
            write(ptr as *mut _, x);
            
            Self {
                ptr: NonNull::new_unchecked(ptr),
            }
        }
    }
}

// Box的Drop也加了`#[may_dangle]`，那我也加！
unsafe impl<#[may_dangle] T> Drop for MyBox<T> {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.ptr.as_ptr());
            
            let layout = Layout::new::<T>();
            dealloc(self.ptr.as_ptr() as *mut _, layout);
        }
    }
}
```



这个实现其实是有问题的，虽然如果把例一中的`Foo`替换成这个`MyBox`不会报错也不会产生ub，因为这里确实不会访问到`&'s str`所引用的值，仅仅只是将`&'s str`的内存给释放了。但考虑下面的代码：

```rust
//! 例子5

fn main() {
    let _foo;
    let s = String::from("123");
    _foo = MyBox::new(Foo(&*s)); // 例子1中的Foo
}
```

这份代码是能编译过的，但是会产生ub——` drop_in_place(self.ptr.as_ptr())`这行违反了`unsafe impl Drop`的约定：

* 首先`s`仍然是比`_foo`先析构，即`&'s str`在`_foo`析构时是一个悬垂引用；
* 当`_foo`析构的时候，其`MyBox::drop`会先调用`Foo`析构函数；
* `Foo`的析构函数，调用`Foo::drop`，尝试访问了`&'s str`所引用的值，因为`&'s str`是个悬垂引用，此处产生ub。



修复这个错误可以去掉`#[may_dangle]`。但将`MyBox`换回`Box`，这份代码将无法通过编译（还是例子一的错误）。这是为什么呢？

其实是因为`Box<T>`在析构的时候，也会对`T`进行drop check，但`MyBox<T>`并不会。这里小心混淆，`#[may_dangle]`是在`Self`的drop check时跳过对应参数，而不是跳过对应参数的drop check。



下面的代码是会报错的，因为`Foo`的drop check生效了，尽管`Bar`加了`#[may_dangle]`：

```rust
//! 例子6

use 例子1::Foo;
use 例子4::Foo as Bar;

fn main() {
    let _foo;
    let s = String::from("123");
    _foo = Bar(Foo(&*s)); 
}
```



## PhantomData来救场

那怎么给`MyBox<T>`补上`T`的drop check呢。



首先我们回顾一下例子2，分析一下其报错的原因。`tuple`是个元组，其析构为什么会给`Foo`进行drop check呢？这是因为drop check的下一条规则：它会递归地给`tuple`的所有字段进行drop check。

例子6报错的原因也是一样的，`Bar`在析构的时候，不仅对`Bar`自身进行drop check，也对其字段`Foo`进行drop check，然后`Foo`不满足条件就报错了。



那么现在drop check就有第三条规则了：

> 记`foo: Foo<..., T, ...>`，其中`T`可以是个生命周期参数，也可以是个泛型参数。然后`foo`的作用域记为`'foo`，当`foo`被析构的时候：
>
> 1. 如果`Foo<..., T, ...>`  实现了 `Drop`，都有`T` strictly outlive `'foo` 。
>
> 2. 除非`T`在实现`Drop`时标记了`#[may_dangle]`
> 3. 递归地给`foo`所有字段进行drop check



一步步点开`Box`的实现我们会发现，里面有个类型为`PhantomData<T>`的字段，其定义：

```rust
pub struct Unique<T: ?Sized> {
    pointer: NonNull<T>,
    _marker: PhantomData<T>,
}

#[lang = "phantom_data"]
#[stable(feature = "rust1", since = "1.0.0")]
pub struct PhantomData<T: ?Sized>;
```

`PhantomData<T>`是rust中的一个洞（因为这个定义用户写不出），它假装自己有一个`T`，但它实际上是个ZST，在他进行drop check的时候，也会对`T`进行drop check。

破案了。



那么仿造`Box`，我们只要在`MyBox`的定义上加上一个`PhantomData`就可以让`T`的drop check回来了：

```rust
struct MyBox<T> {
    ptr: NonNull<T>,
    _marker: PhantomData<T>,
}
```



## 那些被特殊照顾的类型

这里解释一下为什么原先`MyBox<T>`，`T`的drop check不生效。这是因为`NonNull<T>`内部是个裸指针，而裸指针是不需要进行drop check的。不需要drop check的类型还有：

* 基础类型（`bool`，数字类型，`char`，`str`，`!`）
* 指针类型（借用、裸指针、函数指针）
* 函数定义类型

这些都是自然成立的，他们都没有析构函数（BTW，我觉得`ManuallyDrop`也可以不check呀）。



那么连同上面提到的`PhatomData`，我们终于可以给出一个比较完整的drop check规则了：

> 记`foo: Foo`，然后`foo`的作用域记为`'foo`，当`foo`被析构的时候：
>
> 1. 当`Foo`以下类型不需要drop check
>
>    * 基础类型（`bool`，数字类型，`char`，`str`，`!`）
>    * 指针类型（借用、裸指针、函数指针）
>    * 函数定义类型
>    * `ManuallyDrop<T>`
>
> 2. 当`Foo == PhantomData<T>`时，对`T`进行drop check
>
> 3. 当`Foo`的有一个参数列表`Foo<..., T, ...>`，`T`可以是lifetime参数，也可以是泛型参数
>
>    1. 如果`Foo<..., T, ...>`  实现了 `Drop`，都有`T` strictly outlive `'foo` 。
>
>    2. 除非`T`在`Foo`实现`Drop`时标记了`#[may_dangle]`
>    3. 递归地给`Foo`所有字段进行drop check



## 习题时间

解释下面的代码为什么无法编译：

```rust
struct Foo<T>(T);

impl<T> Drop for Foo<T> {
    fn drop(&mut self) {
    }
}

fn main() {
    let _foo;
    let s = String::from("123");
    _foo = foo(&*s);
}

fn foo<'s>(_: &'s str) -> Foo<fn() -> &'s str> {
    Foo(|| "321")
}
```



### tips

一些outlive规则（与形变规则无关）：

* 不带泛型参数的类型 outlive 'static
* `&'a T: 'a` 
* `fn() -> &'a T: 'a`
* `fn(&'a T): 'a`
* ADT `Foo<'a>: 'a`
* ADT `Foo<T>: 'a` 当且仅当`T: 'a`
* `dyn Foo + 'a: 'a` 
* `dyn Foo + 'a: 'a`
* `dyn Foo + 'a: 'a`
* `dyn Foo<Assoc = T>: 'a`当且仅当`T: 'a`
* `&'a T: 'r` 当且仅当 `'a: 'r`
* `impl Trait` 同 `dyn Trait`
* `fn foo<T>(t: T) { ...任意local的'a... }` 都有`T: 'a`
* `&'a T` 的必要条件是 `T: 'a`


## 补充：PhantomData怎么不进行drop check？

其中的代码竟然能通过编译？

```rust
//! 例子8

use std::marker::PhantomData;

struct Pending<T> {``
    phantom: PhantomData<T>,
}

fn pending<T>(value: T) -> Pending<T> {
    Pending { 
        phantom: PhantomData,
    }
}

struct Inspector<'a> {
    value: &'a String,
}

impl Drop for Inspector<'_> {
    fn drop(&mut self) {
        eprintln!("drop inspector {}", self.value)
    }
}

fn main() {
    let p: Pending<Inspector>;
    let s = "hello".to_string();
    p = pending(Inspector { value: &s });
}
```

按刚刚的drop check规则来分析一下p:
- 对`p: Pending<Inspector<'s>>`进行drop check
  - 等价于 对`p.phantom: PhantomData<Inspector<'s>>`进行drop check
    - 等价于 对`Inspector<'s>`进行drop check
      - `Inspector<'s>: Drop`，则`'s` strictly outlive `'p`
      - 与`'p: 's`产生矛盾
- 编译失败

但这里却编译成功了？

这段代码其实在NLL合入之后才能编译通过。NLL在一个变量析构的时候会往MIR里插入DROP(var)指令，在变量析构的时候才会进行drop check。而PhantomData<T>: Copy，不需要析构，rust就不会往MIR中插入析构指令，也就不进行drop check了。

回到例子中，Pending<T>是由PhantomData<T>组合成的ADT，本身也没有实现Drop，于是也不会进行drop check，也没有刚刚我们分析的一串了。

```rust
assert!(!std::mem::needs_drop::<Pending<Foo>>());
```

我这里再来点好玩的，既然不跑析构指令就能不drop check，那么下面代码也是可以编译的：

```rust
// 沿用例子1中的定义：
fn main() {
    let foo;
    let s = String::from("123");
    loop {} // NLL 会知道下面的代码不可达，包括foo的析构函数，于是这段代码也能编译通过。
    foo = Foo(&*s);
}
```

## 补充：drop check与NLL

NLL的规则可以简单概括为，将lifetime推导成借用第一次使用到最后一次使用的范围（MIR上）。但目前rust会在变量超出作用域时插入析构函数（可能带上一个drop flag），如果编译器没法确定析构函数是否访问到某个借用的时候（实现了Drop，而且没加`#[may_dangle]`；或者是`dyn trait`,`impl trait`），编译器只能假设借用的最后一次使用是到作用域结束的时候了——这时候NLL就会退化为LL。

比如说这个例子，目前是编译不过的：

```rust
//! 例子9 沿用例子1的定义

fn main() {
    let foo;
    let s = String::from("123");
    foo = Foo(&*s);
    drop(foo); // 尽管移动走了foo
} // 但是编译器还是会在这里插入DROP指令，变相延长了最后一次使用的范围。
```

其实rust是可以分析出来MIR上那些路径会析构/不会析构/可能会析构的，但目前并没有将其应用到drop check中，导致目前drop check的限制还有点大。这种代码其实在循环里经常遇到，其实挺难受的。

当然，这种限制也是可以绕过的

```rust
//! 例子10 沿用例子1的定义

fn main() {
    let foo;
    let s = String::from("123");
    foo = ManuallyDrop::new(Foo(&*s)); // 通过`ManuallyDrop`跳过drop check
    drop(ManuallyDrop::into_inner(foo));  // 再挪出来
} 
```