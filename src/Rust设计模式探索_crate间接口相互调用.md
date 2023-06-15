# Rust设计模式探索：crate间接口相互调用



在程序设计中我们常常将一个复杂的程序，划分为若干个组件，这些组件最后通过一个容器组装并管理起来，然后程序通过这个容器对外暴露功能。

在最理想的情况下，这些组件应该是相互独立的，他们不相互调用，不会反过来去访问容器的其它部分，像砖块一样垒起一个程序。这样的设计当然很好，但也很难。在一个比较复杂的工程中多多少少会遇到需要模块间相互调用的需求。

实现这个需求，有几种做法，其中方法5是我想到的最好的一个方法：



# 方法1：组件内对容器弱引用

组件中持有一个容器的引用（或弱引用），通过弱引用来访问容器的其它部分。

一般来说在rust中比较少采用这种设计，因为rust中缺乏对编写相互依赖结构的支持，写起来难受，还容易出bug。

```rust
struct ComponentA {
    container: Weak<Container>,
    // ...
}

struct Container {
    component_a: Arc<ComponentA>,
    component_b: Arc<ComponentB>,
    component_c: Arc<ComponentC>,
    // ...
}
```

# 方法2：参数注入容器的引用

程序中所有函数都传递整个容器的引用作为参数，那么一个组件就可以通过这个引用访问其它组件提供的方法了。

但这种做法也有一些问题，这样要求组件和容器需要定义在同一个crate中，如果容器定义在比组件更上层的crate中，那么组件就没法拿到容器的类型，也没法定义一个需要容器引用的方法了。

```rust
struct Container {
    component_a: ComponentA,
    component_b: ComponentB,
    component_c: ComponentC,
    // ...
}

impl ComponentA {
    fn foo(container: &Container) {
        // 可以通过container去访问另一个组件提供的方法
        container.component_b.bar(
            // 自身组件因为在同一个模块内，可以访问其字段
            container.component_a.x
        );
    }
}
```



# 方法3：接口实现分离——集中的接口层

当工程复杂起来之后，我们则一般倾向用crate而非mod去组织组件。crate的依赖结构就变成了

[![](https://mermaid.ink/img/pako:eNqFT7sOwjAM_JXKc_sDGZCgXZlgQlmsxKWRGrsyjhCq-u8EGBh7072k060QJBI4GGd5hgnVmuvg1XNT0QsbJiZtuu5QVV6Eie24k5928h5ayKQZU6zD66frwSbK5MFVGmnEMpsHz1utYjG5vDiAMy3UQlkiGg0J74oZ3Ijzo7oL8k3krykmEz3_zn0_bm8z0U8u?type=png)](https://mermaid.live/edit#pako:eNqFT7sOwjAM_JXKc_sDGZCgXZlgQlmsxKWRGrsyjhCq-u8EGBh7072k060QJBI4GGd5hgnVmuvg1XNT0QsbJiZtuu5QVV6Eie24k5928h5ayKQZU6zD66frwSbK5MFVGmnEMpsHz1utYjG5vDiAMy3UQlkiGg0J74oZ3Ijzo7oL8k3krykmEz3_zn0_bm8z0U8u)

但使用crate带来的一个问题是crate与crate之间不能相互依赖，使得crate提供的接口在crate之间无法相互调用（但其实有在链接时的魔法[linkme](https://crates.io/crates/linkme)，使得crate之间能相互调用）。

为了解决这个问题，可以引入一个Middle层，将一个程序的所有**状态**以及组件提供的**接口**都放在Middle层；而接口的**实现**和**注册**则放在Component层：

[![](https://mermaid.ink/img/pako:eNqFkEELgzAMhf-K5Kx_oIfB1Kun7TR6CTbOQptKlzKG-N_XOYaDwcwpL98j8N4MfTAECgYX7v2IUYpzq6PmIk8TWNAyxaKqDln5KTCxHHd4vcObjX8-robOGuPoB9b_YPMFoQRP0aM1Oc_8MmqQkTxpUHk1NGByokHzkq2YJJwe3IOSmKiENBkUai1eI3pQA7pbvk7IlxA2TcZKiN27s7W65Qn1EWrK?type=png)](https://mermaid.live/edit#pako:eNqFkEELgzAMhf-K5Kx_oIfB1Kun7TR6CTbOQptKlzKG-N_XOYaDwcwpL98j8N4MfTAECgYX7v2IUYpzq6PmIk8TWNAyxaKqDln5KTCxHHd4vcObjX8-robOGuPoB9b_YPMFoQRP0aM1Oc_8MmqQkTxpUHk1NGByokHzkq2YJJwe3IOSmKiENBkUai1eI3pQA7pbvk7IlxA2TcZKiN27s7W65Qn1EWrK)



rustc就是这么做的，在middle中定义了[`GlobalCtxt<'tcx>`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/context/struct.GlobalCtxt.html)包含了一个编译器的所有状态；定义了[`Providers`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/query/struct.Providers.html#)则包含了编译器对内暴露的所有接口，这些接口都会有一个包含`GlobalCtxt`引用的参数，在各个编译器的组件中实现并注册这些方法，同时各个组件也都能调用这些方法。



不过这种集中式的Middle层也有些问题：

1. Middle层将会十分庞大的codebase，且所有组件都会依赖这个Middle层。每当需要重新编译（比如修改到了middle的代码），就需要几乎编译整个工程，也没办法利用并行编译，导致编译时长爆炸。
2. 声明在Middle层中的接口形式有所限制，不能包含泛型函数。
3. 组件没法拥有一些私有的状态，必须都需要在Middle中暴露出来。
4. 组件的接口是动态提供的，没法在编译时就得知调用的接口是否已经有实现了。
5. Middle层通常是单一一个crate，当我们只需要一小部分组件的时候，难以拆分。（不过这算是个小众需求？）



# 方法4：接口实现分离——动态查询

为了解决方法3中的一些问题，我们把Component层拆分为**实现**与**接口**层，将Middle层的接口声明放到Component接口层中，将Middle层与Component相关的状态放在Component的实现层中；而Middle层只保留接口注册、接口查找（以及增加状态注册）的接口。



[![](https://mermaid.ink/img/pako:eNqN0r0KwjAQB_BXKTfrC3QQ2nRxKAg6SZajuWogHyVeEZG-u1GHVgtpM-VyP7gL_J_QeEWQQ2v8vbli4OxUySBdFo_wjlE7Ctl2u4uV7bwjx8XedmbBlCuMmJrRTqf8DS4O-zWunLiZL9M-5YqVTqTmi7RPueS_3st9eK2VMjTfc6EvfvuwAUvBolYxHM-3lcBXsiQhj1dFLfaGJUg3RIo9--PDNZBz6GkDfaeQqdJ4CWghb9Hc4muH7uz9WJPS7EP9DeAnh8MLewPXWA?type=png)](https://mermaid.live/edit#pako:eNqN0r0KwjAQB_BXKTfrC3QQ2nRxKAg6SZajuWogHyVeEZG-u1GHVgtpM-VyP7gL_J_QeEWQQ2v8vbli4OxUySBdFo_wjlE7Ctl2u4uV7bwjx8XedmbBlCuMmJrRTqf8DS4O-zWunLiZL9M-5YqVTqTmi7RPueS_3st9eK2VMjTfc6EvfvuwAUvBolYxHM-3lcBXsiQhj1dFLfaGJUg3RIo9--PDNZBz6GkDfaeQqdJ4CWghb9Hc4muH7uz9WJPS7EP9DeAnh8MLewPXWA)

大概举个例子：

Middle里提供一个类型，用于存放所有组件的状态，和组件提供的接口：

```rust
pub struct GlobalCtxt {
    /// 所有的组件都将注册在这
    components: HashMap<TypeId, Rc<dyn Any>>,
    /// 所有组件所提供的接口都将提供在这
    interfaces: HashMap<TypeId, Rc<dyn Any>>,
}

impl GlobalCtxt {
    pub fn register_component<Component: Any>(
        &mut self, 
        component: Rc<Component>
    ) -> Result<()> { /*...*/ }
    
    pub fn register_interface<Interface: Any>(
        &mut self, 
        interface: Rc<Interface>
    ) -> Result<()> { /*...*/ }
    
    pub fn get_interface<Interface: Any + ?Sized>(&self) 
        -> Option<Rc<Interface>>
    { /*...*/ }
}
```

Component接口层则提供一系列接口的定义：

```rust
// ComponentFooAPI
trait Foo {
    fn foo(&self, ctxt: &GlobalCtxt);
}
```

Component实现层则实现声明在接口层的接口，并将注册到`GlobalCtxt`中：

```rust
struct ComponentFoo {
    count: RefCell<usize>,
}

impl Foo for ComponentFoo {
    fn foo(&self, ctxt: &GlobalCtxt) {
        *self.count.borrow_mut() += 1;
        // 通过GlobalCtxt调用其它组件提供的接口
        let bar = ctxt.get_interface::<Rc<dyn Bar>>().unwrap();
        bar.bar();
    }
}

pub fn register(ctxt: &mut ClobalCtxt) -> Result<()> {
    let component = Rc::new(ComponentFoo::default());
    register.register_component(component.clone())?;
    register.register_interface(component as Rc<dyn Foo>)
}
```



这样解决了方法3中的一些问题：

1. 编译慢：首先Middle不需要巨大的codebase了，只需要一小部分管理组件的代码即可，后续也几乎不会对Middle层做改动；接口定义放在多个crate中，能充分发挥并行编译的带来的编译速度的优化。
2. 组件不可拆分：这个方案下Middle不包含所有组件的代码，且每个组件都有自己单独的两个crate，拆分起来比较简单。
3. 组件状态无法私有化：这个方式，组件的状态本来就定义在组件的实现层中，而Middle中类型则被擦除了，于是不会造成状态的泄露。



但还有一些问题没有解决，而且还引入了其他问题：

1. 接口形式受限：比如这里的例子只能以trait的形式提供满足`Any`且dyn safe的接口；
2. 接口动态提供：同样无法在编译时得知某个接口是否已经提供；
3. 接口查询引入了额外的开销：比如这里是用`HashMap` + `TypeId`来查询接口，相比于方法3，这里就引入了额外的开销。



# 方法5：接口实现分离——静态的依赖注入

有没有一种方案，能同时解决这些问题的呢：

1. 支持crate之间相互调用
2. 接口形式不受限（或者限制很小）
3. 接口调用零开销
4. 接口静态地提供
5. 组件状态不暴露
6. 组件可拆分
7. 编译速度快

最近我可能找到了可能满足这几点的方案，在看到了一个叫[`ref-cast`](https://docs.rs/ref-cast/1.0.16/ref_cast/)的库之后。

> ref-cast库提供了生成`&T -> &Wrapper<T>`转换方法的宏。



这个方案同样将组件分为了实现层和接口层，但不再需要Middle层，而所有集成的操作都放在`Container`层中：

[![](https://mermaid.ink/img/pako:eNqN0r0KgzAQB_BXkZv1BRwKGheHQqGdSpbDnFXIh6QnpYjv3tgOSgupme6OX_gHchM0ThHk0Gr3aDr0nFwq6aVNwhHOMvaWfJJlh9CZwVmyXNRm0H9MucOIrVntNuUruDjVe1y5cT--jPuYK3Y6EcsXcR9zyzshBUPeYK_Cp03LHQnckSEJeSgVtThqliDtHCiO7M5P20DOfqQUxkEhU9XjzaOBvEV9D9MB7dW5tSfVs_PHz2K892N-AfLdtxY?type=png)](https://mermaid.live/edit#pako:eNqN0r0KgzAQB_BXkZv1BRwKGheHQqGdSpbDnFXIh6QnpYjv3tgOSgupme6OX_gHchM0ThHk0Gr3aDr0nFwq6aVNwhHOMvaWfJJlh9CZwVmyXNRm0H9MucOIrVntNuUruDjVe1y5cT--jPuYK3Y6EcsXcR9zyzshBUPeYK_Cp03LHQnckSEJeSgVtThqliDtHCiO7M5P20DOfqQUxkEhU9XjzaOBvEV9D9MB7dW5tSfVs_PHz2K892N-AfLdtxY)



接下来看看这个方法究竟是怎么组织的：

1. 接口层。现在假设有`Even`和`Odd`两个组件，两套接口：

   ```rust
   // even-api
   trait Even {
       fn is_even(&mut self, n: u64) -> bool;
       /// 输出`Even::is_even`被调用的次数
       // 该方法使得`Even`不再dyn safe
       fn emit_count<T>(&self, f: impl FnOnce(usize) -> T) -> T;
   }
   
   // odd-api
   trait Odd {
       fn is_odd(&mut self, n: u64) -> bool;
   }
   ```

2. 实现层。这次的实现层中，多引入了了一个`Proxy`的结构，这个`Proxy`带了一个泛型参数，用于注入接口，以及组件自身的状态。那么所有该组件相关的功能，都可以基于`Proxy`结构来实现：

   ```rust
   // even-impl
   
   #[derive(Defaule)]
   pub struct EvenState {
       count: usize,
   }
   
   #[derive(ref_cast::RefCast)]
   #[repr(transparent)]
   pub struct EvenProxy<Ctx: ?Sized> {
       ctx: Ctx
   };
   
   // `&EvenProxy<Ctx>`可以当做`&EvenState`使用
   impl<Ctx: ?Sized> Deref for EvenProxy<Ctx> 
   where
       // 要求被注入的上下文需要满足
       Ctx: AsRef<EvenState>,
   {
       type Target = EvenState;
       fn deref(&self) -> &EvenState {
           self.ctx.as_ref()
       }
   }
   
   // `&mut EvenProxy<Ctx>`可以当做`&mut EvenState`使用
   impl<Ctx: ?Sized> Deref for EvenProxy<Ctx> 
   where
       // 要求被注入的上下文需要满足
       Ctx: AsRef<EvenState>,
       Ctx: AsMut<EvenState>
   {
       fn deref_mut(&mut self) -> &mut EvenState {
           self.ctx.as_mut()
       }
   }
   
   
   // 实现`Even`接口
   impl<Ctx: ?Sized> Even for EvenProxy<Ctx>
   where
       // 通过`Ctx`注入`Odd`接口
       Ctx: Odd,
   {
       fn is_even(&mut self, n: u64) -> bool {
          is_even_impl(self, n)
       }
       
       fn emit_count<T>(&self, f: impl FnOnce(usize) -> T) -> T {
           f(self.count)
       }
   }
   
   fn is_even_impl(
       // 用`dyn`理论上能加快编译速度，因为不需要等到泛型实例化的时候才生成代码
       proxy: &mut EvenProxy<dyn Odd>,
       n: u64,
   ) -> bool {
        // `deref_mut`到了`EvenState`
       proxy.count += 1;
   
       (n == 0) || proxy
           .ctx // 通过`ctx`调用Odd接口
           .is_odd(n - 1)
   }
   ```

   odd的实现同理。

3. 容器层。容器将会存放所有组件的状态，容器同时也实现所有组件提供的接口。上面Proxy结构的`Ctx`泛型都将实例化为`Container`：

   ```rust
   #[derive(Default)]
   struct Container {
       // 若有需要，这些字段都可以是lazy的
       even_state: EvenState,
       odd_state: OddState,
   }
   
   // 为了可以让`Proxy`可以解引用到组件的状态
   impl AsRef<EvenState> for Container { /**/ }
   impl AsMut<EvenState> for Container { /**/ }
   impl AsRef<OddState> for Container { /**/ }
   impl AsMut<OddState> for Container { /**/ }
   
   // 为`Container`实现所有组件的接口，让`Proxy`可以通过ctx调用
   impl Even for Container {
       #[inline]
       fn is_even(&mut self, n: u64) -> bool {
           // `&mut Container` -> `&mut EvenProxy<Container>`
           // 即`Ctx`被实例化为了`Container`，
           // 调用`Proxy.ctx.is_even()`时就会被`Container`转发到实际的实现中
           EvenProxy::ref_cast_mut(self).is_even(n)
       }
       
       #[inline]
       fn emit_count<T>(&self, f: FnOnce(usize) -> T) -> T {
           EvenProxy::ref_cast(self).emit_count(f)
       }
   }
   
   impl Odd for Container { /**/ }
   ```



我们来分析一下`EvenProxy::<Ctx>::is_even`的调用链：

1. `EvenProxy::<Ctx>::is_even`调用了`Ctx::is_odd`
2. `Ctx`实例化为`Container`，即相当于调用了`Container::is_odd`
3. `Container::is_odd`实际派发到了`OddProxy::<Container>::is_odd`中
4. `OddProxy::<Ctx>::is_odd`内部可能调用了`Ctx::is_even`
5. ...

现在相当于`Container`是一个中间商（之前的方法3,4的中间商则是Middle中定义的类型），通过它来派发具体的实现。



方法5其实基本解决了上面几种方案带来的问题，目前我还没想到有什么其它问题（除了`RefCast`的强转暂时没有支持其它指针类型，可以自己实现）

[*] 支持crate之间相互调用：通过`Proxy`类型

[*] 接口形式不受限：trait的定义基本没什么限制

[*] 接口调用零开销：简单的派发，可以被inline掉

[*] 接口静态地提供：可以静态地检查Ctx有没有满足某个trait。

[*] 组件状态不暴露：组件的状态类型暴露，但是内部结构可以不暴露

[*] 组件可拆分：没有Middle层，可拆分

[] 编译速度快（相对）：没有Middle层，组件接口层分开定义。不过有可能因为泛型的原因，实例化和codegen都在Container发生，导致Container层编译速度慢（但可以在内部实现中多使用dyn 的Ctx，将编译时间多分摊到各个组件中）。



demo代码可以参考我的仓库：https://github.com/TOETOE55/dep-inj。这个里面我实现了一些宏，方便生成`Proxy`相关的模板代码。

```rust
#[derive(Default, dep_inj::DepInj)]
#[target(EvenProxy)] // 生成`struct EventProxy`
pub struct EvenState {
    count: usize,
}


// 实现`Even`接口
impl<Ctx: ?Sized> Even for EvenProxy<Ctx>
where
    // 通过`Ctx`注入`Odd`接口
    Ctx: Odd,
{
    fn is_even(&mut self, n: u64) -> bool {
       is_even_impl(self, n)
    }
    
    fn emit_count<T>(&self, f: impl FnOnce(usize) -> T) -> T {
        f(self.count)
    }
}

fn is_even_impl(
    // 用`dyn`理论上能加快编译速度，因为不需要等到泛型实例化的时候才生成代码
    proxy: &mut EvenProxy<dyn Odd>,
    n: u64,
) -> bool {
     // `deref_mut`到了`EvenState`
    proxy.count += 1;

    (n == 0) || proxy
        .deps_ref_mut() 
        .is_odd(n - 1)
}
```

不过Container的一些那些代码暂时想不到该怎么生成比较好，就当做boilerplate了。。
