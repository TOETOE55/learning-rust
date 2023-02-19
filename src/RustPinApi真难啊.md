# Rust Pin api真难啊

最近本来想写rust aliasing rules的，但这些规则并没有确定下来，相关的资料并不好找，先咕咕一段时间。先来分享一波关于Pin一些有趣的“洞”以及之后相关的一些修改。



## Unpin Hack

相关资料

* [Stacked Borrows vs self-referential structs](https://github.com/rust-lang/unsafe-code-guidelines/issues/148)
* [exclude mutable references to !Unpin types from uniqueness guarantees](https://github.com/rust-lang/miri/pull/1952)

这一条是为rustc Unpin开的洞，也是和alias相关的。没了这个洞估计Pin api都没法做到soundness，但目前RFC和std文档都没有提到这个规则，**默认这个hack暂时只提供给rust以及其标准库内使用**。

首先，rust中有两种特殊的引用要求[noalias](https://llvm.org/docs/LangRef.html#parameter-attributes)，`&mut T`和`Box<T>`，即在引用“活着”的时候不允许通过其它引用或指针访问或修改alias的内存。那么这个hack说的就是：**对于`T: !Unpin`，不要求`&mut T`和`Box<T>`noalias**。

那么根据这个规则，就可得到一个违反rust直觉的东西——*对于`T: !Unpin`的类型，允许多个`&mut T`, `&T`, `Box<T>`同时存在，甚至同时去访问和修改`T`！*不过这虽然safe但不sound，比如`mem::swap()`假设了所有`&mut T`都是noalias，内存都是不重叠的；如果允许`fn dup_mut<T: !Unpin>(&mut T) -> (&mut T, &mut T)` safe，就会与已有的`mem::swap`产生冲突，导致ub。



好了，我们为什么需要这条例外的规则呢？我们来考虑一段这代码：

```rust
async fn no_opt() {}

fn main() {
    let fut = async {
        let mut local = 42;
        let r = &mut local;
        no_opt().await; 
        // 当引用跨`.await`的时候会产生自引用
        //
        // fut(state 2)
        // ┌─────────────┐
        // │ local = 42  │◄───┐
        // ├─────────────┤    │
        // │ r=&mut local├────┘
        // └─────────────┘

        // 这里通过自引用访问`local`
        println!("{}", *r);
    };
    
    pin_mut!(fut);
    
    let waker = noop_waker();
    let mut cx = Context::from_waker(&waker);
    
    // `fut.as_mut()`创建了一个指向`fut`的独占借用
    fut.as_mut().poll(&mut cx);
}
```

如果没有unpin hack，这段代码其实违反了noalias的规则：

* `fut.as_mut()`和`r`alias了`local`这片内存
* `fut.as_mut()`和`r`都是独占借用，要求noalias
* 在`fut.as_mut()`有效的期间，`r`访问了`local`，UB



如果把上面代码解糖，并且去掉`Pin`相关的东西，大概长这样：

```rust
fn main() {
    let mut local = 42;
    // async block的自引用用的是裸指针存储
    let raw_ptr = &mut local as *mut i32; 
    // 创建了一个指向`local`的独占借用，使得`raw_ptr`非法（不能再解引用了）
    let safe_ref = &mut local; 
    // UB
    println!("{}", unsafe { *raw_ptr }); 
}
```

这个UB其实非常微妙，一个下载量特别多的库[owning_ref](https://crates.io/crates/owning_ref)也同样犯了这个错误，但这个库竟然有一千三百万的下载量。。。（这个库其实还有UAF的问题。。）



于是Unpin hack便为这种情况给rust的alias rules开了后门。不过这种做法其实是有风险的——`Unpin`是一个trait，而且还是一个auto trait，可以为任意类型实现这个trait，或者实现其negative trait，意味着很容易把非自引用的类型给“牵涉进来”，有可能会把依赖noalias的地方爆破掉，又或者是把很多利用noalias进行优化给干没了。

在trait上开后门其实影响是不太可控的，之所以目前没有正式确定下来这条规则，是希望探索一个更合理的方法去解决这个问题。现在有在讨论的一个方案是，提供类似一个`UnsafeCell`的Wrapper，去完成这个事情[`unsafe_cell_mut`](https://hackmd.io/@CV5q1SRASEuY8WfOgd_3iQ/BkmQIn7Bs)，不过api倒是要认真考虑，别把std搞unsound了 。

（注：`UnsafeCell`目前也开了后门，即允许`&UnsafeCell`的值可变，所谓interior mutable；如果`UnsafeCellMut`确定要做的话，我们也可以称为interior aliasable）



## Unsoundness in `Pin`

相关资料

* [Unsoundness in `Pin`](https://internals.rust-lang.org/t/unsoundness-in-pin/11311)
* [Use deref target in Pin trait implementations](https://github.com/rust-lang/rust/pull/67039)
* [permit negative impls for non-auto traits](https://github.com/rust-lang/rust/pull/68004) 给std开洞，可以用negative impl
* [RFC1023](https://rust-lang.github.io/rfcs/1023-rebalancing-coherence.html) fundamental attribute

这是`Pin`刚稳定没多久的时候，就被发现了它的api是unsound的（当时官方还特别自豪，在对编译器尽可能小的改动下，设计出一套safe api表达不可移动的语义）。之前看这篇文章还不太懂unsafe和`Pin`，现在回看一下，发现还是很有意思的。



首先前置需要了解的关于`Pin`api的知识点

1. `Pin<P>`的soundness，依赖于具体的`P`的实现，然后如果`P`实现正确，其safe api则保证在`P::Target: !Unpin`的情况下不会移动`P::Target`。
2. 一旦构造了一个`Pin<P>`，如果`P::Target: !Unpin`，则所有重新拿到`&P`, `&mut P`的地方都要遵守`Pin::new_unchecked`的安全契约（这是个unsafe函数），即不可移动`P::Target`。（这是具体的`P`所必要遵守的规则）
3. 与`Pin<P>` api打交道的trait有`Clone`, `Deref`, `DerefMut`, `PartialEq`, `Drop`等trait，他们其中有些能安全地拿到`&P`或者`&mut P`。这就是2的情形，需要遵守契约。



我们开始来爆破

### `Pin<&T>` 是unsound的

首先我们需要知道，rust一般的类型不允许为外部crate的类型实现外部的crate，这一规则称之为**孤儿原则**。这带来一个好处是，一个类型一旦实现好一些trait之后，外部就不能再对它的性质进行修改，本地crate就可以信任这个实现，依赖这个类型的性质去编写一些unsafe代码。但是`&T`, `&mut T`, `Box<T>`, `Pin<P>`是特例，我们可以为他们添加之前未实现的一些trait。比如现在`&T`没有实现`DerefMut`，我们就可以为某个具体的`&Foo`实现`DerefMut`。



`Pin<P>::as_mut()`依赖`P::deref_mut`的实现，但`<&T as DerefMut>::deref_mut`的在std里还没定义，可以被外部类型不加约束地实现，但外部的实现是不能被信任的。

```rust
impl<P: DerefMut> Pin<P> {
    pub fn as_mut(&mut self) -> Pin<&mut P::Target> {
        unsafe { Pin::new_unchecked(&mut *self.pointer) }
    }
}
```

爆破的方法，就是给一个坏的`&T` `DerefMut`的实现，破坏掉`Pin`的契约：（这是无船给的例子）

```rust
struct MyType<'a>(Cell<Option<&'a mut MyType<'a>>>, PhantomPinned);

// 0. 为`&MyType`实现了`DerefMut`，这是个安全却坏的实现
impl<'a> DerefMut for &'a MyType<'a> {
    fn deref_mut(&mut self) -> &mut MyType<'a> {
        self.0.replace(None).unwrap()
    }
}


fn main() {
    let mut unpinned: MyType<'_> = MyType(Cell::new(None), PhantomPinned);
    
    // 1. 首先构造一个`Pin<&MyType>`
    let p = Box::pin(MyType(Cell::new(Some(&mut unpinned)), PhantomPinned));
    let mut p_ref: Pin<&MyType<'_>> = p.as_ref();
    
    // 2. 通过`<&MyType>::deref_mut()`构造出`Pin<&mut MyType>`。
    //    根据`<&MyType>::deref_mut()`的实现，`p_mut`实际pin住了`unpinned`
    let p_mut: Pin<&mut MyType<'_>> = p_ref.as_mut();

    // 3. 这里却可以移动被应该已被pin住的`unpinned`，违反了`Pin`的契约。
    drop(unpinned);
    
    println!("oh no!");
}
```

不过这里暂时没有ub，不过可以把`PhantomPinned`换成`Future`，在违反契约之前通过`poll`构造一个自引用，那么就可以造出一个ub了。注意这里没有加任何unsafe的代码，却违反了契约，说明接口unsound了。



那么后面的修复方案是，std通过negative impl，禁止外部为`&T`实现`DerefMut`（这其实是一个对`&T`的breaking change，并且又开洞了）：

```rust
impl<T> !DerefMut for &T {}
```



### `Pin<&mut T>`是unsound的

和上面原理类似，这次是通过坏的`<&mut MyType as Clone>::clone`来爆破`Pin<P>::clone`。把上面例子稍稍改改就好了。这里就不展开了。

修复的方案也是类似的：

```rust
impl<T> !Clone for &mut T {}
```

 

### `Pin<P>::eq`是unsound的

在修复之前，`Pin`的`PartialEq`的实现是这样的：

```rust
impl<P, Q> PartialEq<Pin<P>> for Pin<Q> 
where
    Q: PartialEq<P>
{ ... }
```

在我们实现`Q: PartialEq<P>`的时候同样能拿到`&P`和`&Q`，所以`Pin<P>::eq`同样也依赖于具体`P`和`Q`的实现是否正确。不过这次并不能通过上面的方法爆破了，因为`&T`, `&mut T`, `Box<T>`都实现了`PartialEq`（以及组合情况），外部crate无法替换他们的实现。



不过`PartialEq`对比`DerefMut`/`Clone`多了一个泛型参数，根据孤儿规则，下面的代码是可以编译的：

```rust
impl<T> PartialEq<LocalType> for RemoteType {}
impl<T> PartialEq<RemoteType> for LocalType {}
```

这意味着我们可以为std里的类型添加自己的`PartialEq`类型实现。



这次我们通过为`Rc<T>`添加新`PartialEq`实现爆破`Pin`的`PartialEq`。

* 首先我们可以通过`&Rc<T>`移动`T`

  ```rust
  fn main() {
      let rc = Rc::new(PhantomPinned);
      {    
          // 这里通过`&rc`，构造了个`Pin<Rc<PhantomPinned>>`，后面要求`PhantomPinned`不能被移动
          let pinned = unsafe { Pin::new_unchecked(rc.clone()) };
      }
      
      // 把`PhantomPinned`移动出来了，破坏了`pin`的约束
      let unpinned = rc.try_unwrap().unwrap();
  }
  ```

* 利用这点我们实现一个坏的`PartialEq`

  ```rust
  struct MyType<T>(Cell<Option<Rc<T>>>); 
  
  // 假装`MyType`是个指针
  impl<T> Deref for MyType<T> {
      type Target = MyType<T>;
      fn deref(&self) { self }
  }
  
  // 实现一个坏的`PartialEq`
  impl<T> PartialEq<MyType<T>> for Rc<T> {
      fn eq(&self, other: &MyType<T>) -> bool {
          other.set(Some(self.clone()));
          true
      }
  }
  
  fn main() {
      let my_type = Pin::new(MyType(Cell::new(None)));
      
      {
          // pin住了`PhantomPinned`
          let pinned: Pin<Rc<PhantomPinned>> = Rc::pin(PhantomPinned);
          // 把`Rc`克隆了一份到`my_type`里
          assert!(pinned_rc == pin_my_type);
      }
      
      // 把`PhantomPinned`移动了出来，违反了pin的约束
      let unpinned = my_type.0.replace(None).try_unwrap().unwrap();
  }
  ```

同理，也可以利用这个漏洞造一个UB。（具体造ub的方法，在[Unsoundness in `Pin`](https://internals.rust-lang.org/t/unsoundness-in-pin/11311)这个帖子里就有）



那么修复方法就是，`PartialEq`不再允许下游再拿到`&P`和`&Q`了，转而依赖`P::Target`的`PartialEq`实现（`&T`可移动不走`T`）：

```rust
impl<P: Deref, Q: Deref> PartialEq<Pin<P>> for Pin<Q>
where
    Q::Target: PartialEq<P::Target>
```



***

其实帖子里还有提到`CoerceUnsized`导致的unsoundness，这里就不再展开了，一来我还不太了解这个trait，二来这是个未稳定的特性，三来这个unsoundness还没有修（=。=）



总之这几个unsoundness再次告诉我们，**unsafe 不能信任一个任何一个未经验证的safe trait的实现**，因为它可能做任何事情，随时会破坏掉你依赖的契约。



## `PollFn<F>` should be `Unpin`?

参考资料：

*  [Surprising soundness trouble around `PollFn`](https://internals.rust-lang.org/t/surprising-soundness-trouble-around-pollfn/17484) 

* [poll_fn and Unpin: fix pinning](https://github.com/rust-lang/rust/pull/102737)

* [marking `Unpin` as unsafe](https://rust-lang.zulipchat.com/#narrow/stream/187312-wg-async/topic/marking.20Unpin.20as.20unsafe)

这是最近的一次发生在`Pin`相关的 api上的一次breaking change（rust 1.64，写这篇文章是1.65）。事实上这次接口并没有unsound，只是容易发生接口的误用，为此社区还发生了些争论。



在此之前，`PollFn<F>`无论`F`是否`Unpin`，`PollFn<F>`都是`Unpin`的（不过这也是1.64才稳定到std）。那么意味着我们可以把`Pin<Pointer<PollFn<F>>>`当做`Pointer<PollFn<F>>`耍，并且随意移动`PollFn<F>`。

```rust
impl<F> Unpin for PollFn<F> {}
```

这是有原因的，因为`PollFn<F>`实现`Future`需要访问`&mut F`：

```rust
impl<T, F> Future for PollFn<F>
where
    F: FnMut(&mut Context) -> Poll<T>
{
    type Output = T;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<T> {
        // 这里要`Pin<&mut Self>`能deref到`Self`并且拿到`&mut F`
        (&mut self.f)(cx)
    }
}
```



不过下面的代码就不幸运了（更不幸的是`tokio::join!`和`tokio::select!`内部就是这么实现的）：

```rust
let mut future = trouble();

let mut pinned = Box::pin(future::poll_fn(move |cx| { // 将`!Unpin`的future移动到闭包里
    unsafe { Pin::new_unchecked(&mut future) }.poll(cx)
}));

```

这里有两个会产生UB的地方

1. 可以通过`pinned`解引用移动自引用结构`future`
2. 没有实现`!Unpin`，但`future`有自引用，没有触发unpin hack，违反了noalias

究其原因，就是里面没有遵守`Pin::new_unchecked()`的调用约定——在一个随时可以被移动的闭包中试图说服编译器不会移动被捕获的`future`，但这是不对的。



修复方式也很简单，`PollFn<F>`改为只有在`F: Unpin`的时候才满足`Unpin`：

```rust
impl<F: Unpin> Unpin for PollFn<F> {} 
// 在`F: !Unpin`的时候，通过auto trait得到`PollFn<F>: !Unpin`

impl<T, F> Future for PollFn<F>
where
    F: FnMut(&mut Context) -> Poll<T>
{
    type Output = T;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<T> {
        // 这里就要确保`FnMut::call_mut`不会移动闭包了
        // 目前不允许重载`FnMut::call_mut`的话，可以保证
        unsafe { (&mut self.get_unchecked_mut().f) }(cx)
    }
}
```

修改之后，刚刚的代码将不再会产生UB：

1. 无法通过`pinned`移动`future`
2. 开启了unpin hack，关掉了noalias



***

在这次的讨论中，有几个争论的点

1. 这个breaking change，会导致破坏掉很多下游代码，而且目前在`futures`中没合到`std`的`Future`组合子都有类似的问题。

   结论：

   * 对于std来说这个`poll_fn`刚和进去几天，跑了跑crater发现基本没有break下游代码；说明这种误用很少。
   * tokio迅速跟进，先于std把`join!`和`select!`的实现改对了。如果std不改的话日后可能导致社区分裂。

2. 这完全属于`Pin::new_unchecked()`的误用，而不是`PollFn`的问题。

   结论：原则上，这是是没有错的，因为`PollFn`的接口是sound的。*但接口的作者除了保证接口sound以外，也应该减少和unsafe交互时产生UB的情况。*

3. `Unpin`只是一种lint，只是用于提Pin api高人体工学的优化。就像`UnwindSafe`一样。

   结论：

   * 这完全是一种对`Unpin`的误解，将`Unpin`设计成safe trait是一种失误。（我之前也这么误解了。。。）

   * `Unpin`其实是有安全契约的：为`T`实现`Unpin`的必要条件是，可以安全地pin-project到所有的字段到`&mut field`。现在在std里是safe  trait的原因，只是因为pin-project没有在std里提供。。（pin-project这个crate自己定义了个`UnsafeUnpin` trait）——扩展一下，如果`spawn`没有在std里提供，`Send`/`Sync`也可以不用unsafe，因为std不会产生ub。
   * 不过这个问题日后再观望。。。

   

## Disallow impl `Drop` for `Pin`

参考资料：

* [Do not allow `Drop` impl on foreign fundamental types](https://github.com/rust-lang/rust/pull/99576) bug修复pr
* [`Unimplemented` selecting `Binder(<std::pin::Pin<std::boxed::Box<B>> as std::ops::Drop>, [])`](https://github.com/rust-lang/rust/issues/99575) bug原issue
* [Prohibit specialized drops](https://github.com/rust-lang/rust/issues/8142) 禁止给Drop trait进行特化。



这个bug是在1.60的时候才被发现的，也算是很晚了，又是一个新功能影响旧功能的例子。。。

首先`Drop`trait是有特殊检查的：

1. drop check
2. 只允许`Drop`给ADT(即struct/enum/union)实现
3. impl泛型列表以及泛型约束要完全与类型定义中的完全一致

第一点可以看我之前的文章[讲讲让我熬了几天夜的Drop Check](./讲讲让我熬了几天夜的DropCheck.md)，第二第三点在[Prohibit specialized drops](https://github.com/rust-lang/rust/issues/8142) 这里实现，不过具体原因除了会导致rustc panic之外没详细讨论（后面了解到会产生unsoundness）。



那么`Pin`是ADT，并且标记了`#[fundamental]`，那么就可以在下游crate为`Pin`实现`Drop`：

```rust
struct A;

impl Drop for Pin<Box<A>> {
    fn drop(&mut self) {
        println!("Dropping!");
    }
}

fn main() {
    let _: Pin<Box<()>> = Box::pin(());
}

```

但其实违反了`Drop`trait的第三条规则，并且会导致rustc panic：

```rust
error: internal compiler error: Encountered error `Unimplemented` selecting `Binder(<std::pin::Pin<std::boxed::Box<()>> as std::ops::Drop>, [])` during codegen
  |
  = note: delayed at compiler\rustc_trait_selection\src\traits\codegen.rs:68:32

```



修复方式是`Pin`也需要遵守`Drop`的规则，禁止为`Pin`实现`Drop`。

