# State machine Pattern 

程序界流传着一句名言
> 算法 + 数据结构 = 程序
>
> --*Niklaus*

我望文生义一波，擅自将这句话解读为：**程序可以分为 控制 和 数据 两部分**。所以接下来这篇文章会对这句话中的“算法”替换成“代码”、“控制流”、“过程”、“函数”等，带有“控制”行为的概念。

“算法”相比于“数据结构“来说更加抽象。后者可以有”实体“，有数据和结构，也对其进行修改；但“算法”就不太行了（或者很难），除非你用的是类似lisp那样的语言，“代码即数据”，可以直接将代码当做列表的数据进行操作。

所以最近一直在思考如何描述“过程”或者“控制流”，使得他们可以像数据结构一样变得更加具体一些，变得更容易管理一些。为了解决这些问题：

1. 我们的项目里当然也存在着大量的数据结构 和 各种各样的过程和函数，前者我们将其放入一个套一个的所谓“容器”里进行管理，但是后者，一些正在“运行”的东西，我们其实并没有一个很好的手段去管理他们，尤其是在并发的情况下。
2. 有时候单个函数动辄上千行，成百上千个条件；后面的语句可能会依赖前面语句的一些状态，在不断迭代的过程中很容易就会破坏这些约束从而造成bug。我们是否有手段将这些代码行为描述起来，从而可以更好地验证后续修改没有bug呢？



要思考这些问题，我们就要思考 数据 和 代码 之间究竟有什么不同，以及之间有什么关系。



## Church/Scott encoding或是CPS

如果你对functional programing比较了解，那就会发现，在函数式编程中习惯用很多等价的形式去表示一个数据结构。其中比较简单而闻名的就是 Church encoding 和 Scott encoding了。



### Church number

church encoding用函数来定义自然数：

| 数字 | 函数定义                   | lambda表达式        |
| ---- | -------------------------- | ------------------- |
| `0`  | `zero(f, x) = x`           | `λf.λx.x`           |
| `1`  | `one(f, x) = f(x)`         | `λf.λx.f x`         |
| `2`  | `two(f, x) = f(f(x))`      | `λf.λx.f (f x)`     |
| `3`  | `three(f, x) = f(f(f(x)))` | `λf.λx.f (f (f x))` |
| ...  | ...                        | ...                 |

那么数字运算也可以用函数来定义：

| 运算 | 函数定义                                      | lambda表达式              |
| ---- | --------------------------------------------- | ------------------------- |
| 加   | `plus(m, n) = (f, x) => m(f, n(f, x))`        | `λm.λn.λf.λx.m f (n f x)` |
| 后继 | `succ(n) = (f, x) => f(n(f(x)))`              | `λn.λf.λx.f (n f x)`      |
| 乘   | `mult(m, n) = (f, x) => m((y) => n(f, y), x)` | `λm.λn.λf.λx.m (n f) x`   |
| 乘方 | `exp(m, n) = (f, x) => n(m, f)(x)`            | `λm.λn.n m`               |
| ...  | ...                                           | ...                       |

看起来很抽象？那我们来直接拿一个例子来运算一下：

```
1 + 2
= plus(one, two)
= (f, x) => one(f, two(f, x))   // 展开`plus`定义
= (f, x) => f(two(f, x))		// 展开`one`定义
= (f, x) => f(f(f(x)))			// 展开`two`定义
= three
= 3
```



church encoding除了定义了自然数以外，还定义了bool, list, pair甚至还有一些命题，有兴趣可以直接移步到[Church encoding](https://en.wikipedia.org/wiki/Church_encoding)



### Scott encoding

[scott encoding](https://en.wikipedia.org/wiki/Mogensen%E2%80%93Scott_encoding)就是在church encoding上给出了更加通用的用函数编码数据结构的方式。

比如定义一个只有三个构造子的类型：

```rust
enum Data<A, B, C> {
    C1(),
    C2(A),
    C3(B, C)
}
```

那么，`C1`, `C2`, `C3`就可以用函数来编码

| 构造子 | 函数                                  |
| ------ | ------------------------------------- |
| `C1`   | `C1() = (f1, f2, f3) => f1()`         |
| `C2`   | `C2(a) = (f1, f2, f3) => f2(a)`       |
| `C3`   | `C3(b, c) = (f1, f2, f3) => f3(b, c)` |

那么下面的模式匹配，相当于`value(f1, f2, f3)`。

```rust
match value {
    C1() => f1(),
    C2(a) => f2(a),
    C3(b, c) = f3(b, c)
}
```

这其实不难理解。代入计算就可以知道

* 当`value`是`C1`时，那么`value(f1, f2, f3) = f1()`；
* 当`value`是`C2(a)`，那么`value(f1, f2, f3) = f2(a)`；
* 当`value`是`C3(b, c)`，那么`value(f1, f2, f3) = f3(b, c)`；

当然如果大家有闲情逸致的话可以用scott encoding的方法去编码一下自然数（然后就会发现你得到了Church number）。



### Continuation-passing Style

上面的对数据的编码方式，都是用函数去表示数据结构。其实我们有一个更加深刻的东西来揭示这种表示的原理，也就是所谓的CPS变换。CPS变换大概就是说：

>  对于任意`x: A`，都有一个等价的表示`f: for<R> fn(fn(A) -> R) -> R`，其中`f = (k) => k(x)`，`x = f((y) => y)`.

这里`x`是一个数据，`f`是一个函数，他们可以相互表示。

BTW，CPS的具体应用我这里就不展开了，我只是想表述一件事： **本质上，我们可以说 函数 和 数据 其实都是等价的，可以相互表示的。** 函数通过描述如何消费来定义数据，数据结构通过描述如何构造来定义数据。

但在日常编程的时候，一般会认为 函数 相对于 具体的数据结构来说 会更难使用 且 心智负担会更重一些。那么我们又是否可以利用这个这一点，用数据结构来表示代码呢？



## 迭代、遍历 和 解释器模式？

我们来回过头来思考一下，我们是通过什么来决定代码的控制流的。



### Walk through on your data structure

1. 当我们写出一个迭代的控制流的时候，通常离不开一个列表（一个线性的数据结构）。我们会遍历这个列表然后运算得到一些东西。反过来说，如果我们手里只有一个列表的时候，我们除了迭代也写不出其它有意义的控制流了。

   ```rust
   let mut arr = [1,2,3,4,5];
   let iter = arr
       .iter()
       .map(|x| x * 2)
       .filter(|x| x < 7);
   
   // 迭代，其实是遍历了arr这个数组
   for x in iter {
       println!("{x}");
   }
       
   ```

2. 同样的，当我们手里有一颗树的时候，我们可能用这棵树来查找，或者来做其它事情。但由这棵树导出的控制流，无非都是用来遍历这颗树的。

   ```rust
   /// 普普通通二叉树
   enum BinTree<T> {
       Leaf(T),
       Node(Box<Self>, Box<Self>)
   }
   
   /// 普普通通用于遍历二叉树的Visitor
   trait BinTreeVisitor<T> {
       fn visit_leaf(&mut self, t: &T) {}
       fn visit_node(&mut self, left: &BinTree<T>, right: &BinTree<T>) {
           match left {
               Leaf(t) => self.visit_leaf(t),
               Node(l, r) => self.visit_node(l, r),
           }
           
           match right {
               Leaf(t) => self.visit_leaf(t),
               Node(l, r) => self.visit_node(l, r),
           }
       }
   }
   ```

   

**我们的控制流取决于我们处理的数据的结构，数据的结构决定程序的控制流**。



### 解释器模式——DSL

将刚刚这句话贯彻得最彻底的莫过于设计模式中的“解释器模式”了。为了表示某种特定功能的程序，需要设计特定的数据结构，遍历即解释；然后程序的行为（控制流）取决输入的数据的结构长什么样。

DSL就是解释器模式最常见的应用，比如我们用的diesel就是DSL（强类型的DSL）：

```rust
// 构造了一个dsl结构
// `update_dsl: UpdateStatement<Filter<user, Like<email, String>>, _, <Eq<banned, bool>>::Changeset>`
let update_dsl = update(users.filter(email.like("%@spammer.com")))
    .set(banned.eq(true)); 
// 解释执行
update_dsl.execute(&mut connection); 
```

或者我们用语法树简单实现一个四则运算的计算器，本质上也是解释器模式：

```rust
enum Expression {
    /// 0,1,2
    Lit(i64) 
    /// a + b
    Plus(Box<Self>, Box<Self>),
    /// a - b
    Minus(Box<Self>, Box<Self>),
    /// a * b
    Multi(Box<Self>, Box<Self>),
    /// a / b
    Divi(Box<Self>, Box<Self>),
}

impl Expression {
    fn eval(self) -> Result<i64, DiviByZero> { 
        match self {
            /* eval 的控制流决定于 Expression 这棵树长什么样 */ 
        }
    }
}
```

也有比解释器模式稍微弱一点的模式——访问者模式其实也是一种data-driven的模式。



上一节说的是可以用函数/过程来表示数据结构，那么这一节就是一个“对偶”的表述，**我们可以用数据结构决定函数/过程的控制流**。那么在Rust里，我们是否也能用某种数据结构来解决文章开头的两个问题呢——管理“运行中”的实体；抽象复杂的控制流。



那么到这里其实我忍不住再曲解一下文章开头的那句话：

1. 解读1：程序 分为 控制 和 数据 两部分；
2. 解读2：**算法 和 数据 是 程序 的一体两面。**

是不是冯氏架构其实也是想表达这个事情呢？我冥冥中有这样的感觉。。



## Rust中的状态机

说实话，我其实是为了想介绍Rust中的状态机，才牵强地写出了前面的内容。不妨把某个状态机看作一种特化的数据结构时，其实也符合上面给的结论，状态机可以在某种程度上表示控制流——**将控制流编码到状态机的节点和边上**。

Rust的API中很多时候都能看到状态机的身影，状态机几乎无处不在。



### `async`/`.await`状态机

Rust中自带状态机的“语法糖”，`async`/`.await`，用于编写异步代码。

```rust
let fut = async {
    fut_a().await;
    
    for _ in 0..10 {
        fut_b().await;
    }
};
```

这个async block大概会编译成下面的代码，为了方便理解，简化的规则：

* 每个`.await`会编译成一个enum的Variant，会存储当前的局部变量以及`.await`的操作数
* `.await`之间的代码则会插入到match的对应分支中

```rust
struct Fut {
    _marker: PhantomPinned,
    state: FutState,
}

enum FutState {
    Start,
    S1(fut_a),
    S2(Range<i32>, fut_b)
    Done(Option<()>), // result
}

impl Future for Fut {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'a>) -> Poll<Self::Output> {
        let this = unsafe { Pin::into_inner_unchecked(self) };
        loop {
            match &mut this.state {
                /// Start --> S1
                Start => {
                    this.state = S1(fut_a());
                }
                
                /// S1 --> S2 or
                /// S1 --> Done
                S1(fut_a) => {
                    let fut_a = unsafe { Pin::new_unchecked(fut_a) };
                    ready!(fut_a.poll(cx));
                    let mut range = 0..10;
                    if range.next().is_some() {
                        this.state = S2(range, fut_b());
                    } else {
                        this.state = Done(Some(()));
                    }
                }
                
                /// S2 --> S2 or
                /// S2 --> Done
                S2(mut range, fut_b) => {
                    let fut_b = unsafe { Pin::new_unchecked(fut_b) };
                    ready!(fut_b.poll(cx));
                    if range.next().is_some() {
                        this.state = S2(range, fut_b());
                    } else {
                        this.state = Done(Some(()));
                    }
                }
                
                Done(result) => {
                    return result.take().expect("Fut::poll after is done");
                }
            }
        }
    }
}
```

就可以看做下面这个状态机

* 将enum的variant看作是状态机的一个**节点**，
* 将`.await`之间的代码看作状态机的**边**
* 另外在`S1`和`S2`状态中`fut_a`和`fut_b`自己内部也可能有自己的一个状态机

异步控制流则相当于“按需”地将这个状态机从`Start`状态推进到`Done`状态：

[![](https://mermaid.ink/img/pako:eNqdkc1qwzAQhF9F6BCk4pjYRx166hv4lqqEjbV2DdbK2BIkhLx7ZTmmPxQfchBoZ0ajD_bGa2eQKz558PjWQTuC1aR9VShWFaIJ_gQyCWUUSjECtSgOeV4cZMZm-5zs-by_fLD9_jW-WxoeA1MpB_ng-l7UF5l302lAMh21Qv6Mlv9GRwRzFZJpTWy3Yywh5IQXL5I_OYu_emaOZ4rI0VpUPoCWmvMG-lbyz4cb4OUK_kTNgs0zbnG00Jm4zVtMMs39J1rUXMWrwQZC7zXXdI9RCN5VV6q58mPAjIfBfO9_FQego3NxbKCf8P4FNHqyiQ?type=png)](https://mermaid.live/edit#pako:eNqdkc1qwzAQhF9F6BCk4pjYRx166hv4lqqEjbV2DdbK2BIkhLx7ZTmmPxQfchBoZ0ajD_bGa2eQKz558PjWQTuC1aR9VShWFaIJ_gQyCWUUSjECtSgOeV4cZMZm-5zs-by_fLD9_jW-WxoeA1MpB_ng-l7UF5l302lAMh21Qv6Mlv9GRwRzFZJpTWy3Yywh5IQXL5I_OYu_emaOZ4rI0VpUPoCWmvMG-lbyz4cb4OUK_kTNgs0zbnG00Jm4zVtMMs39J1rUXMWrwQZC7zXXdI9RCN5VV6q58mPAjIfBfO9_FQego3NxbKCf8P4FNHqyiQ)



### 尾调用优化和状态机

当然，我们还可以用状态机来描述其它控制流，比如说用来做尾递归的优化。做法其实和`async`/`.await`类似，也需要将控制流映射到一个状态机上。

一个比较经典的尾调用例子，如果没有尾调用优化的话，这个`is_odd(n)`的空间复杂度是`S(n) = O(n)`的，因为每个函数调用都会增加`O(1)`的栈内存：

```rust
fn is_odd(n: u64) -> bool {
    match n {
        0 => false,
        1 => true,
        /// 尾调用了`is_even`
        n => is_even(n - 1)
    }
}

fn is_odd(n: u64) -> bool {
    match n {
        0 => true,
        1 => false,
        /// 尾调用了`is_odd`
        n => is_odd(n - 1)
    }
}
```

如果描述成状态机的话，大概是下图的样子（比如调用`is_odd(n)`）：

* 整个调用链上的每一个尾调用作状态机的状态
* 最终结果作为终态
* 第一个调用作为初态
* 尾调用前的语句作为边

[![](https://mermaid.ink/img/pako:eNptj08LwjAMxb9KyVG2g9einvQwEAS9aT2ENdPC2o7-EWTsu9sxy4aYU_J775Gkh9pKAg4-YKC9wodDLQxLJZWjOihr2PE8kdvqzspyxyp_knJClT-8yOQ-4a9hpJwZVrL10jjHf8ScTCtGZbP9k1toUIAmp1HJdHo_OgWEJ2kSwFMrqcHYBgHCDMmKMdjL29TAg4tUQOzk_CzwBlufaIfmam2ehw84mFRc?type=png)](https://mermaid.live/edit#pako:eNptj08LwjAMxb9KyVG2g9einvQwEAS9aT2ENdPC2o7-EWTsu9sxy4aYU_J775Gkh9pKAg4-YKC9wodDLQxLJZWjOihr2PE8kdvqzspyxyp_knJClT-8yOQ-4a9hpJwZVrL10jjHf8ScTCtGZbP9k1toUIAmp1HJdHo_OgWEJ2kSwFMrqcHYBgHCDMmKMdjL29TAg4tUQOzk_CzwBlufaIfmam2ehw84mFRc)



同样按照`async`/`.await`的映射规则，可以得到下面的代码：

* 将状态机的状态映射为enum的Variant
* 将状态机的边映射为Variant之间的转换

```rust
enum OddEvenState {
    IsOdd(u64),
    IsEven(u64),
    Return(bool)
}

impl OddEvenState {
    fn trampoline(mut self) -> bool {
        loop { 
            match self {
                /// IsOdd -> [*]
                /// IsOdd -> IsEven
                IsOdd(n) => {
                    match n {
                        0 => self = Return(false),
                        1 => self = Return(true),
                        n => self = IsEven(n - 1),
                    }
                }
                /// IsEven -> [*]
                /// IsEven -> IsOdd
                IsEven(n) => {
                    match n {
                        0 => self = Return(true),
                        1 => self = Return(false),
                        n => self = IsOdd(n - 1),
                    }
                }

                Return(res) => {
                    break res;
                }
            }
        }
    }
}

fn is_odd(n: u64) -> bool {
    IsOdd(n).trampoline()
}


fn is_even(n: u64) -> bool {
    IsEven(n).trampoline()
}
```

这段优化后的代码空间复杂度则降为了`S(n) = O(1)`，因为每个状态的变更都不会增加内存使用。



### 类型上的状态机

刚刚说的这两种都是将状态编码到enum Variant上，这种做法在其它语言也是可以做的，比如C其实很多时候也大量应用这种方式。但对于Rust来说还有一种得天独厚的编码方式——**将状态编码到类型上**——这得益于rust的affine type system，除了Copy类型的值，其它值最多只能使用一遍，这可以保证状态转化的唯一性。

比如说有个状态`A`到状态`B`的转换，在rust中就可以写成：

```rust
struct A;
struct B;
fn trans(a: A) -> B { ... }
```

状态`A`一旦转换成`B`之后，原来的`A`被消耗了，无法再被使用；我们也无法得到多个`B`。我们无需考虑`A`在被消耗后的情况。

而如果考虑状态`A`到状态`B`再到`A`的转换，还可以考虑在`B`中持有`&mut A`，表示对`A`的独占，在`B`结束后，状态又能回到`A`：

```rust
struct A;
struct B<'a>(&'a mut A);

fn trans<'a>(&'a mut A) -> B<'a> {...}
```





#### `TcpListner`和`TcpStream`

rust中TCP Socket的API，我觉得就很好地体现了这种状态机的应用。首先，TCP Socket的状态机可以描述为：

* `TcpListner`表示欢迎套接字
* `TcpStream`表示连接套接字
* 如果A进程要监听连接的话则通过`bind`先创建一个`TcpListner`
  * `TcpListner`监听到建立连接B进程的请求，则会通过`accept`创建一个`TcpStream`
* 对于B进程来说则可以通过`connect`创建一个与A通信的`TcpStream`
* 两种套接字都可以关闭

[![](https://mermaid.ink/img/pako:eNp1Ub1OAzEMfpXIE6BW7BmYGNnKBGEIia-NuDinnE8IVZVAYkAgISYGeAkmJOB1uPY1SBtajhY8RLG_H32yx2CCRZBQs2bcd3oYtVckUh3vnIh-f08cmurA1YyEUYpTR3YDHnBE7aUwgQgNZ7wj27TRxmD1P3FpmGlbJnWM25lOgVFENxyxCEVXnOF5Zdnn2137cN_ePLcf79PHl9nT9cp5dvs6vbxKkzqYM_zOgWQX5qtUmbueKb129zy6DvFX_LSXtIky1H8araPQA4_Ra2fTDcbzmQIeoUcFMn0tFropWYGiSaLqhsPgggxIjg32oKnsz9VAFrqs07TSdBTCsp98AT4oqBc?type=png)](https://mermaid.live/edit#pako:eNp1Ub1OAzEMfpXIE6BW7BmYGNnKBGEIia-NuDinnE8IVZVAYkAgISYGeAkmJOB1uPY1SBtajhY8RLG_H32yx2CCRZBQs2bcd3oYtVckUh3vnIh-f08cmurA1YyEUYpTR3YDHnBE7aUwgQgNZ7wj27TRxmD1P3FpmGlbJnWM25lOgVFENxyxCEVXnOF5Zdnn2137cN_ePLcf79PHl9nT9cp5dvs6vbxKkzqYM_zOgWQX5qtUmbueKb129zy6DvFX_LSXtIky1H8araPQA4_Ra2fTDcbzmQIeoUcFMn0tFropWYGiSaLqhsPgggxIjg32oKnsz9VAFrqs07TSdBTCsp98AT4oqBc)

那么在rust std中就可以设计出对应的接口：

```rust

impl TcpListener {
    /// [*] --> `TcpListener`: 创建欢迎套接字
    fn bind(addr: impl ToSocketAddrs) -> Result<TcpListner> { ... }
    
    /// `TcpListener` --> `TcpListener`: `accept`自己状态不改变
    /// `TcpListener` --> `TcpStream`: `accept`创建一个新的`TcpStream`
    fn accept(&self) -> Result<(TcpStream, SocketAddr)> { ... }
}

/// `TcpListener` --> [*]: 关闭欢迎套接字
impl Drop for TcpListner { ... }


impl TcpStream {
    /// [*] --> `TcpStream`: 建立连接，创建连接套接字
    fn connect(addr: impl ToSocketAddrs) -> Result<TcpStream> { ... }
}

/// `TcpStream` --> [*]: 关闭连接套接字
impl Drop for TcpStream { ... }
/// 可以在连接套接字上读数据
impl Read for TcpStream { ... }
/// 可以在连接套接字上写数据
impl Write for TcpStream { ... }
```

这里其实就是用`TcpListner`和`TcpStream`两种类型表示了两种不同的socket状态，用他们的方法表示表示状态机的边。





#### Lock，即访问控制

其实我们常用的锁，本身也可以看做一个状态机：

* `Mutex`表示unlocked的状态
* `MutexGuard`表示locked的状态

[![](https://mermaid.ink/img/pako:eNpNTzsOwjAMvUrkCVC5QAYmJBa6wAZhsBIXIpqkSh0Bqnp30mZoPVh-P8lvAB0MgYSekelo8RnRKS_yGBtJsw1enC-FKfu-e4j9_iDqxPSVwtOn8DNelFPCaKRog36v9Jldx00M3Sb5ybaFChxFh9bkh4YppYBf5EiBzKehBlPLCpQfsxUTh-vPa5AcE1WQOrNUANlg22e2Q38LYcFkLIdYl9Jz9_EPT21VHQ?type=png)](https://mermaid.live/edit#pako:eNpNTzsOwjAMvUrkCVC5QAYmJBa6wAZhsBIXIpqkSh0Bqnp30mZoPVh-P8lvAB0MgYSekelo8RnRKS_yGBtJsw1enC-FKfu-e4j9_iDqxPSVwtOn8DNelFPCaKRog36v9Jldx00M3Sb5ybaFChxFh9bkh4YppYBf5EiBzKehBlPLCpQfsxUTh-vPa5AcE1WQOrNUANlg22e2Q38LYcFkLIdYl9Jz9_EPT21VHQ)



对应的rust接口是：

```rust
impl<T> Mutex<T> {
    /// 创建一个unlocked的数据
    fn new(t: T) -> Mutex<T> { ... }
    
    /// 将锁的状态转换为locked的状态
    fn lock<'a>(&'a self) -> MutexGuard<'a, T> { ... }
}

/// MutexGuard<'_, T>释放后变为了unlocked的状态
impl<T> Drop for MutexGuard<'_, T> { ... }
```

不过跟这节开头讲到的用`&mut`表示独占的写法有点不同，`Mutex::lock`接受的是一个`&self`，但这里也同样做到了独占的效果，大家可以思考一下为什么。





## State machine Pattern

上面一节中列了rust一些状态机的存在形式，主要分为enum-based的状态机（即以enum variant作为状态机的状态），和type-based的状态机（即以type为状态机的状态）。但状态机在rust中实在是太常见了，在谷歌上一搜，就能搜到很多相关的文章，大家也都可以去看看：

* [Pretty State Machine Patterns in Rust](https://hoverbear.org/blog/rust-state-machine-pattern/)
* [State in Rust](https://refactoring.guru/design-patterns/state/rust/example)
* [State Machines: Introduction — 2020-03-30](https://blog.yoshuawuyts.com/state-machines/)
* [State Machines II: an enum-based design — 2022-08-23](https://blog.yoshuawuyts.com/state-machines-2/)
* [State Machines III: Type States — 2023-01-02](https://blog.yoshuawuyts.com/state-machines-3/)
* [Ringbahn II: the central state machine](https://without.boats/blog/ringbahn-ii/)

本来想把能想到的一些状态机的写法总结下来，但是精力有限，这里仅在**异步的语境**下，再展开一下enum-based的state machine。



### 规则1：基础规则

rust中async的”基础“自然是`Future`trait，这代表一个“异步的运算”，需要通过调用`poll`不断地轮询`Future`，直到返回`Ready`才能得到计算的值：

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

这也可以视作一个状态机，具有`Pending`和`Ready`两种状态（`Pending`里一般有子状态），`poll`则作为状态转移的边：

[![](https://mermaid.ink/img/pako:eNplTzEKwzAM_IrRnHzAQ6eOLZR0K15EpKQGWw6OPISQv9etKR2qQZzuDk63w5iIwcKqqHz2OGeMTkwd8plH9UnMZWhM2zcW8jKbvj99sTVLCuFfHhhpayJ0EDlH9FTD9rfVgT45sgNbIfGEJagDJ0e1YtF032QEq7lwB2Wh33tgJwxrZReUR0q_m8lrytdW6NPreAGuq0mn?type=png)](https://mermaid.live/edit#pako:eNplTzEKwzAM_IrRnHzAQ6eOLZR0K15EpKQGWw6OPISQv9etKR2qQZzuDk63w5iIwcKqqHz2OGeMTkwd8plH9UnMZWhM2zcW8jKbvj99sTVLCuFfHhhpayJ0EDlH9FTD9rfVgT45sgNbIfGEJagDJ0e1YtF032QEq7lwB2Wh33tgJwxrZReUR0q_m8lrytdW6NPreAGuq0mn)



对于`Future`来说，`Ready`都是状态机的终态，一般不允许再推进其状态。我们可以放松这样的限制，将这两个状态当做两个普通的状态放到一个更大的状态机中。



比如说现在有一个状态机：

* 有两个状态`Ping`和`Pong`，两个状态中都会有一个Future需要推进
* 状态`Ping`中的Future就绪后，就会进入状态`Pong`中
* 状态`Pong`中的Future就绪后，也会进入状态`Ping`中

对应的状态图如下：

[![](https://mermaid.ink/img/pako:eNptkDEKwzAMRa9iNCcX8NCpYwul3YqhiFhJDbZlHHkoIXevmwyhbTSIr_80SH-Cji2BhlFQ6OhwyBhMVLWsy9SJ46hO19VZ-8XFQbXtYRFaJfb-kar8pbxHeYfyP3VfFBoIlAM6Ww-dPrsG5EmBDOgqLfVYvBgwca6rWIRvr9iBllyogZLs9hroHv1Y3YTxzrzNZJ1wPq9hLJnMb2CvXJE?type=png)](https://mermaid.live/edit#pako:eNptkDEKwzAMRa9iNCcX8NCpYwul3YqhiFhJDbZlHHkoIXevmwyhbTSIr_80SH-Cji2BhlFQ6OhwyBhMVLWsy9SJ46hO19VZ-8XFQbXtYRFaJfb-kar8pbxHeYfyP3VfFBoIlAM6Ww-dPrsG5EmBDOgqLfVYvBgwca6rWIRvr9iBllyogZLs9hroHv1Y3YTxzrzNZJ1wPq9hLJnMb2CvXJE)



可以写成以下代码：

```rust
struct PingPong<FutPing, FutPong> {
    _marker: PhantomPinned,
    state: PingPongState<FutPing, FutPong>,
}

enum PingPongState<FutPing, FutPong> {
    Ping(FutPing),
    Pong(FutPong)
}


impl<FutPing, FutPong> PingPong<FutPing, FutPong>
where
    FutPing: Future,
    FutPong: Future,
{
    /// Ping --> Pong
    fn poll_ping(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>,
        /// 构造一个`FutPong`
        mut pong: impl FnMut(FutPing::Output) -> FutPong
    ) -> Poll<()> {
        let this = unsafe { Pin::into_inner_unchecked(self) };
        
        loop {
            match this.state {
                Ping(fut_ping) => {
                    let fut_ping = unsafe { Pin::new_unchecked(fut_ping) };
                    let output = ready!(fut_ping.poll(cx));
                    // Ping --> Pong
                    this.state = Pong(pong(output));
                    // `poll_ping`这条边结束
                    return Ready(());
                }
                /// 没有从`Pong`出发的`poll_ping`的边
                /// 所以是`unreachable!()`
                Pong(_) => {
                    unreachable!();
                }
            }
        }
    }
    
    /// Pong --> Ping 与 `Poll_ping`类似
    fn poll_pong(...) {...}
    
    /// 对外提供的，用于推进这个状态机的接口
    pub fn poll(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>,
        
        pong: impl FnMut(FutPing::Output) -> FutPong,
        ping: impl FnMut(FutPing::Output) -> FutPing,
    ) -> Poll<()>
    {
        let this = unsafe { Pin::into_inner_unchecked(self.as_mut()) };
        match this {
            Ping(_) => self.poll_ping(cx, pong),
            Pong(_) => self.poll_pong(cx, ping),
        }
    }
}
```

当然，现在我们有`async`/`.await`，可以将`PingPong`大概写成这样：

```rust
async {
    let mut ping_output = ...;
    let mut pong_output = ...;
    loop {
        ping_output = ping(pong_output).await;
        pong_output = pong(ping_output).await;
    }
}
```



到这里，我们可以得到一个初步的状态机映射**规则1**：

* 用enum variant表示状态机的状态

* 用`poll_xxx`方法表示这样的边：

  [![](https://mermaid.ink/img/pako:eNplj8EKwjAMhl-l5Ly9QA_CxKNe9CaFEdZMC20zuhQmY-_is_hkVneYYMgh-b8fkn-Gji2BhlFQ6ODwljCo17O0iaqUdYk6cRzV8bzqv7RRdb1TjVYDe99O0_QP9xuECgKlgM6Wg_PHZEDuFMiALqOlHrMXAyYuxYpZ-PKIHWhJmSrIg91eBN2jH4s6YLwybztZJ5xOa6hvtuUNjuZOvA?type=png)](https://mermaid.live/edit#pako:eNplj8EKwjAMhl-l5Ly9QA_CxKNe9CaFEdZMC20zuhQmY-_is_hkVneYYMgh-b8fkn-Gji2BhlFQ6ODwljCo17O0iaqUdYk6cRzV8bzqv7RRdb1TjVYDe99O0_QP9xuECgKlgM6Wg_PHZEDuFMiALqOlHrMXAyYuxYpZ-PKIHWhJmSrIg91eBN2jH4s6YLwybztZJ5xOa6hvtuUNjuZOvA)
  
  
  



rust基于规则1的抽象有：

* `Future`
* `Stream`，异步的迭代
* `AsyncRead`，异步读bytes
* `AsyncWrite`，异步写bytes



这里给个小小习题，根据规则1写出对应下面状态机的代码：

[![](https://mermaid.ink/img/pako:eNp1UEsKwjAQvUqYdXuBLIRWl7rRnQQkNFMN5FPSBJqW3sWzeDKjAVsFh-Ex894MzLwJGisQKPSee9xJfnVck8c9JTMkhZAOGy-tIfvjh8xYkbLckIqSzip1GYZhvZfFehH_b8YY1_T2l85Yf4njOEIBGp3mUqTzp9cQA39DjQxoKgW2PCjPgJk5jfLg7SmaBqh3AQsInVgeBtpy1Se24-Zs7dKjkN66Q7bo7dT8BOFoYZk?type=png)](https://mermaid.live/edit#pako:eNp1UEsKwjAQvUqYdXuBLIRWl7rRnQQkNFMN5FPSBJqW3sWzeDKjAVsFh-Ex894MzLwJGisQKPSee9xJfnVck8c9JTMkhZAOGy-tIfvjh8xYkbLckIqSzip1GYZhvZfFehH_b8YY1_T2l85Yf4njOEIBGp3mUqTzp9cQA39DjQxoKgW2PCjPgJk5jfLg7SmaBqh3AQsInVgeBtpy1Se24-Zs7dKjkN66Q7bo7dT8BOFoYZk)



### 规则2：立刻转移



接下来在规则1中添加一个更简单的规则。

`poll_xxx`的边一般只用来描述那些不会立刻发生改变的状态，即异步地改变状态，但有些状态的转移却是可以立刻发生的，比如重置计时器。

一般的计时器都会提供一个`reset`的方法，用来重置计时，这个操作是能立刻完成的，大概的状态机可以描述如下：

[![](https://mermaid.ink/img/pako:eNp1kDEOwzAIRa9iMScX8NApY7u0nSovKCapJdtEDh6iKHevW1fNkjIgPjwQsELPlkDDLCjUORwTBhNVMesS9eI4qvO1ZqrvyOPi4qja9vQTWk3s_QFwd4E4y9_6PiDRTFKJb9MRAA0ESgGdLUuvb9yAPCmQAV1CSwNmLwZM3AqKWfi2xB60pEwN5MnuZ4Ie0M8lO2F8MO-arBNOl_qYz3-2F4kaYX0?type=png)](https://mermaid.live/edit#pako:eNp1kDEOwzAIRa9iMScX8NApY7u0nSovKCapJdtEDh6iKHevW1fNkjIgPjwQsELPlkDDLCjUORwTBhNVMesS9eI4qvO1ZqrvyOPi4qja9vQTWk3s_QFwd4E4y9_6PiDRTFKJb9MRAA0ESgGdLUuvb9yAPCmQAV1CSwNmLwZM3AqKWfi2xB60pEwN5MnuZ4Ie0M8lO2F8MO-arBNOl_qYz3-2F4kaYX0)

​    

函数签名一般会写成这样：

```rust
impl Delay {
    pub fn reset(self: Pin<&mut Self>, dur: Duration);
}
```



我们将这种能立刻完成的状态转移，添加到规则中去，得到**规则2：**

* enum variant映射为状态机的状态

* `fn poll_xxx(self: Pin<&mut Self>, cx: &mut Context<'_>, arg: Arg) -> Poll<Res>`方法映射为这样的边，用来处理一些异步的操作：

  [![](https://mermaid.ink/img/pako:eNplj8EKwjAMhl-l5Ly9QA_CxKNe9CaFEdZMC20zuhQmY-_is_hkVneYYMgh-b8fkn-Gji2BhlFQ6ODwljCo17O0iaqUdYk6cRzV8bzqv7RRdb1TjVYDe99O0_QP9xuECgKlgM6Wg_PHZEDuFMiALqOlHrMXAyYuxYpZ-PKIHWhJmSrIg91eBN2jH4s6YLwybztZJ5xOa6hvtuUNjuZOvA?type=png)](https://mermaid.live/edit#pako:eNplj8EKwjAMhl-l5Ly9QA_CxKNe9CaFEdZMC20zuhQmY-_is_hkVneYYMgh-b8fkn-Gji2BhlFQ6ODwljCo17O0iaqUdYk6cRzV8bzqv7RRdb1TjVYDe99O0_QP9xuECgKlgM6Wg_PHZEDuFMiALqOlHrMXAyYuxYpZ-PKIHWhJmSrIg91eBN2jH4s6YLwybztZJ5xOa6hvtuUNjuZOvA)
  
* `fn set(self: Pin<&mut Self>, arg: Arg)`映射成这样的边，则表示一些立刻发生的操作（读或写）：

  [![](https://mermaid.ink/img/pako:eNpNjjEOwjAMRa8SeW4vkAEJxAgLbCiLFbsQqUmqxBlQ1btwFk6GoUOxPNj_f329GXwmBgtVUPgY8F4wmvdL1yWjQ6Gwl5CTOV1W_d_dm77fmYM1lQU6iFwiBtK6-es7kAdHdmD1JB6wjeLApUWj2CRfn8mDldK4gzbRBgB2wLGqOmG65bz9TEFyOa_IP_LlA72LQwo?type=png)](https://mermaid.live/edit#pako:eNpNjjEOwjAMRa8SeW4vkAEJxAgLbCiLFbsQqUmqxBlQ1btwFk6GoUOxPNj_f329GXwmBgtVUPgY8F4wmvdL1yWjQ6Gwl5CTOV1W_d_dm77fmYM1lQU6iFwiBtK6-es7kAdHdmD1JB6wjeLApUWj2CRfn8mDldK4gzbRBgB2wLGqOmG65bz9TEFyOa_IP_LlA72LQwo)





### 规则3：取消

在规则1中，我撒了个小谎，说`Future`只有两种状态，但是事实上`Future`至少有三种状态——取消状态（或者叫终止）：

[![](https://mermaid.ink/img/pako:eNplULsKwzAM_BWjsSQ_4KFTxxZKurXuICIlNfgRHHsIIf9eN6akJRrE6e4E0s3QemKQMEaMfNLYB7TKiVykA7dReyfOTWFKv7Ij7XpR18cvlmLwxuzlhpGmX3FveRyeUlDwQ5HWjX8BKrAcLGrKZ84fm4L4YssKZIbEHSYTFSi3ZCum6G-Ta0HGkLiCNND2GMgOzZjZAd3d-21m0tGHS4liTWR5A4poWdE?type=png)](https://mermaid.live/edit#pako:eNplULsKwzAM_BWjsSQ_4KFTxxZKurXuICIlNfgRHHsIIf9eN6akJRrE6e4E0s3QemKQMEaMfNLYB7TKiVykA7dReyfOTWFKv7Ij7XpR18cvlmLwxuzlhpGmX3FveRyeUlDwQ5HWjX8BKrAcLGrKZ84fm4L4YssKZIbEHSYTFSi3ZCum6G-Ta0HGkLiCNND2GMgOzZjZAd3d-21m0tGHS4liTWR5A4poWdE)



因为在Rust中所有的类型都是可以**随时随地** **安全地**析构的，这是一条“公理”，通常可以表示为：

```rust
fn drop<T>(_: T) {}
```

对于`Future`这个抽象来说，则利用了Rust这个特性来表示取消一个异步运算——在不再关心异步计算的结果时，我们可以直接析构对应的`Future`，由析构函数自动回收相关资源。

而对于状态机来说，每个状态都需要考虑析构此时，思考对应状态是否允许析构；状态机结构上，至少都会有一个终态，且其它所有状态都会有一条到这个终态的边；我们也对这个终态其赋予“取消“的含义。



结合上面所说的几点，我们修订一下规则2，得到**规则3**：

* enum variant映射为状态机的状态

* `fn poll_xxx(self: Pin<&mut Self>, cx: &mut Context<'_>, arg: Arg) -> Poll<Res>`方法映射为这样的边，用来处理一些异步的操作：

  [![](https://mermaid.ink/img/pako:eNplj8EKwjAMhl-l5Ly9QA_CxKNe9CaFEdZMC20zuhQmY-_is_hkVneYYMgh-b8fkn-Gji2BhlFQ6ODwljCo17O0iaqUdYk6cRzV8bzqv7RRdb1TjVYDe99O0_QP9xuECgKlgM6Wg_PHZEDuFMiALqOlHrMXAyYuxYpZ-PKIHWhJmSrIg91eBN2jH4s6YLwybztZJ5xOa6hvtuUNjuZOvA?type=png)](https://mermaid.live/edit#pako:eNplj8EKwjAMhl-l5Ly9QA_CxKNe9CaFEdZMC20zuhQmY-_is_hkVneYYMgh-b8fkn-Gji2BhlFQ6ODwljCo17O0iaqUdYk6cRzV8bzqv7RRdb1TjVYDe99O0_QP9xuECgKlgM6Wg_PHZEDuFMiALqOlHrMXAyYuxYpZ-PKIHWhJmSrIg91eBN2jH4s6YLwybztZJ5xOa6hvtuUNjuZOvA)
  
* `fn set(self: Pin<&mut Self>, arg: Arg)`方法映射成这样的边，则表示一些立刻发生的操作（读或写）：

  [![](https://mermaid.ink/img/pako:eNpNjjEOwjAMRa8SeW4vkAEJxAgLbCiLFbsQqUmqxBlQ1btwFk6GoUOxPNj_f329GXwmBgtVUPgY8F4wmvdL1yWjQ6Gwl5CTOV1W_d_dm77fmYM1lQU6iFwiBtK6-es7kAdHdmD1JB6wjeLApUWj2CRfn8mDldK4gzbRBgB2wLGqOmG65bz9TEFyOa_IP_LlA72LQwo?type=png)](https://mermaid.live/edit#pako:eNpNjjEOwjAMRa8SeW4vkAEJxAgLbCiLFbsQqUmqxBlQ1btwFk6GoUOxPNj_f329GXwmBgtVUPgY8F4wmvdL1yWjQ6Gwl5CTOV1W_d_dm77fmYM1lQU6iFwiBtK6-es7kAdHdmD1JB6wjeLApUWj2CRfn8mDldK4gzbRBgB2wLGqOmG65bz9TEFyOa_IP_LlA72LQwo)
  
* 状态机有终态，将析构函数映射为 状态机所有状态到终态的边，表示“取消”推进这个状态机：

  [![](https://mermaid.ink/img/pako:eNpNjz0OwjAMha8SeUTtBTIgITHCAhuEwapdiNTEVeoMVdW7cBZORgCJ1nqDfz49-U3QCDFYGBSV9x7vCYN5Pf9y0ZQin7hRL9EcTuvrmtmZut6a6-ZmDSXpoYLAKaCn4j59EAf64MAObGmJW8ydOnBxLihmlfMYG7CaMleQe1r-AdtiN5Rtj_EissxMXiUdfwm-QeY3HJpKGw?type=png)](https://mermaid.live/edit#pako:eNpNjz0OwjAMha8SeUTtBTIgITHCAhuEwapdiNTEVeoMVdW7cBZORgCJ1nqDfz49-U3QCDFYGBSV9x7vCYN5Pf9y0ZQin7hRL9EcTuvrmtmZut6a6-ZmDSXpoYLAKaCn4j59EAf64MAObGmJW8ydOnBxLihmlfMYG7CaMleQe1r-AdtiN5Rtj_EissxMXiUdfwm-QeY3HJpKGw)
  
  
  
  


在rust不支持linear type之前（niko称之为must move type，用且必须只能用一次的的类型，这样的类型不允许自动析构），其实都需要考虑取消这件事。

考虑一个场景，假设一个状态机有一段缓冲区，因为因为是异步操作，把这个缓冲区的引用丢给了其它线程去读写，这时候状态机被取消了，缓冲区被释放，那么其他线程就得到了一个悬垂引用，产生UB。这就是一个典型的cancel safe问题，没有考虑对应状态下可能被析构的可能性。





### 规则4：并发

到规则3为止，我们还只能在单一状态机上做状态转移，不太好去描述一些并发的场景。就比如说前面说描述的TCP套接字：

* 欢迎套接字在监听到新连接后，会创建一个 连接套接字
* 欢迎套接字和连接套接字相互独立，因此能分别接收和发送数据

对应的接口是这样：

```rust
impl TcpListener {
    /// `TcpListener` --> `TcpListener`: `accept`自己状态不改变
    /// `TcpListener` --> `TcpStream`: `accept`创建一个新的`TcpStream`
    fn accept(&self) -> Result<(TcpStream, SocketAddr)> { ... }
}
```

因为`TcpListener`和`TcpStream`是两个独立的socket，于是就可以在不同的线程下，并发地去收发数据。那么我们的状态机模式，也可以参考这个接口添加新规则，需要并发的时候，提供方法返回一个独立的状态机：



**规则4：**

* enum variant映射为状态机的状态

* `fn poll_xxx(self: Pin<&mut Self>, cx: &mut Context<'_>, arg: Arg) -> Poll<Res>`方法映射为这样的边，用来处理一些异步的操作：

  [![](https://mermaid.ink/img/pako:eNplj8EKwjAMhl-l5Ly9QA_CxKNe9CaFEdZMC20zuhQmY-_is_hkVneYYMgh-b8fkn-Gji2BhlFQ6ODwljCo17O0iaqUdYk6cRzV8bzqv7RRdb1TjVYDe99O0_QP9xuECgKlgM6Wg_PHZEDuFMiALqOlHrMXAyYuxYpZ-PKIHWhJmSrIg91eBN2jH4s6YLwybztZJ5xOa6hvtuUNjuZOvA?type=png)](https://mermaid.live/edit#pako:eNplj8EKwjAMhl-l5Ly9QA_CxKNe9CaFEdZMC20zuhQmY-_is_hkVneYYMgh-b8fkn-Gji2BhlFQ6ODwljCo17O0iaqUdYk6cRzV8bzqv7RRdb1TjVYDe99O0_QP9xuECgKlgM6Wg_PHZEDuFMiALqOlHrMXAyYuxYpZ-PKIHWhJmSrIg91eBN2jH4s6YLwybztZJ5xOa6hvtuUNjuZOvA)
  
* `fn set(self: Pin<&mut Self>, arg: Arg)`方法映射成这样的边，则表示一些立刻发生的操作（读或写）：

  [![](https://mermaid.ink/img/pako:eNpNjjEOwjAMRa8SeW4vkAEJxAgLbCiLFbsQqUmqxBlQ1btwFk6GoUOxPNj_f329GXwmBgtVUPgY8F4wmvdL1yWjQ6Gwl5CTOV1W_d_dm77fmYM1lQU6iFwiBtK6-es7kAdHdmD1JB6wjeLApUWj2CRfn8mDldK4gzbRBgB2wLGqOmG65bz9TEFyOa_IP_LlA72LQwo?type=png)](https://mermaid.live/edit#pako:eNpNjjEOwjAMRa8SeW4vkAEJxAgLbCiLFbsQqUmqxBlQ1btwFk6GoUOxPNj_f329GXwmBgtVUPgY8F4wmvdL1yWjQ6Gwl5CTOV1W_d_dm77fmYM1lQU6iFwiBtK6-es7kAdHdmD1JB6wjeLApUWj2CRfn8mDldK4gzbRBgB2wLGqOmG65bz9TEFyOa_IP_LlA72LQwo)
  
* 状态机有终态，将析构函数映射为 状态机所有状态到终态的边，表示“取消”推进这个状态机：

  [![](https://mermaid.ink/img/pako:eNpNjz0OwjAMha8SeUTtBTIgITHCAhuEwapdiNTEVeoMVdW7cBZORgCJ1nqDfz49-U3QCDFYGBSV9x7vCYN5Pf9y0ZQin7hRL9EcTuvrmtmZut6a6-ZmDSXpoYLAKaCn4j59EAf64MAObGmJW8ydOnBxLihmlfMYG7CaMleQe1r-AdtiN5Rtj_EissxMXiUdfwm-QeY3HJpKGw?type=png)](https://mermaid.live/edit#pako:eNpNjz0OwjAMha8SeUTtBTIgITHCAhuEwapdiNTEVeoMVdW7cBZORgCJ1nqDfz49-U3QCDFYGBSV9x7vCYN5Pf9y0ZQin7hRL9EcTuvrmtmZut6a6-ZmDSXpoYLAKaCn4j59EAf64MAObGmJW8ydOnBxLihmlfMYG7CaMleQe1r-AdtiN5Rtj_EissxMXiUdfwm-QeY3HJpKGw)
  
* `fn spawn(self: Pin<&mut Self>, arg: Arg) -> NewStateMachine` 方法返回一个新的与`Self`独立的状态机，两个状态机可以并发地推进：

  [![](https://mermaid.ink/img/pako:eNpdUbFOwzAQ_RXrJkDtD2RASsVIGegGZrDsC7WU2JHjqEJVJJiKGBhZYEcCBiQkhkT8DU3zGTg2UqLecD6_e3fPvlsD1wIhgsIyiyeSXRuWUUWcCWmQW6kVOT0PSPCeSc5wteiDOeNLqZCsQ7K3y6MrMp0e92cAq3F58LFn7DWJSJGzlTrgBh14OGbO_nPjHkq7d6SYWKITEg_6nki298_bpm6fPncP3-3tXftS78sNFd3mravfA_G3eYy7n4_da9NtvmaBg0p4NZhAhiZjUriJ-R9TsEvMkELkQoEJK1NLgarKUVlp9eJGcYisKXECZS6GGUOUsLRwaM7UhdbDHYW02szDVvxyqj8ndJLO?type=png)](https://mermaid.live/edit#pako:eNpdUbFOwzAQ_RXrJkDtD2RASsVIGegGZrDsC7WU2JHjqEJVJJiKGBhZYEcCBiQkhkT8DU3zGTg2UqLecD6_e3fPvlsD1wIhgsIyiyeSXRuWUUWcCWmQW6kVOT0PSPCeSc5wteiDOeNLqZCsQ7K3y6MrMp0e92cAq3F58LFn7DWJSJGzlTrgBh14OGbO_nPjHkq7d6SYWKITEg_6nki298_bpm6fPncP3-3tXftS78sNFd3mravfA_G3eYy7n4_da9NtvmaBg0p4NZhAhiZjUriJ-R9TsEvMkELkQoEJK1NLgarKUVlp9eJGcYisKXECZS6GGUOUsLRwaM7UhdbDHYW02szDVvxyqj8ndJLO)
  
  
  



`tower::Service`就是一个典型的基于规则4的抽象：

* 把`Service`试作状态机，除了终态以外，至少有两个状态，`NotReady`和`Ready`，
* 通过`poll_ready`将`NotReady`推进到`Ready`，才能调用`call`返回一个新的状态机`Future`，然后状态又回到`NotReady`
* 这时`Service`可以运行，同时也可以通过推进`Future`并发地处理上个请求

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future
    where
        <Self::Future as Future>::Output == Result<Self::Response, Self::Error>;

    fn poll_ready(
        &mut self, 
        cx: &mut Context<'_>
    ) -> Poll<Result<(), Self::Error>>;
    
    /// 返回一个独立的新的Future，可以并发地处理请求
    fn call(&mut self, req: Request) -> Self::Future;
}
```

可以将其表示为以下的状态机：

[![](https://mermaid.ink/img/pako:eNp1kbFOwzAQhl_F8gSofQEPTIgJGMoGQciKr60lx47c81BVkWAqYmBkgR0JGJCQGBLxNjTNY-DYgCsEHs535-_ut30LmhsBlNEZcoQ9ySeWF5kmfglpIUdpNDkYxUy0RwZHwMWcDIe7PwEjpVHq3Pb-H9g_TLQJ23foLDCSc6W2cg8ibP9mkmJPbfYJT_jqQRYx16_TnbNQ6feYrDartPFFVk6mSMw46qTSXoKsru5WTd3evqyv39qLy_a-jhoJ65aPXf0Ujz-am9Cke39ePzTd8vX7whEHLYIkHdACbMGl8J8fLptRnEIBGWXeFTDmTmFGM115lDs0x3OdU4bWwYC6UqRxUTbmauazJdcnxqQYhERjD-OAw5yrT3v-rJg?type=png)](https://mermaid.live/edit#pako:eNp1kbFOwzAQhl_F8gSofQEPTIgJGMoGQciKr60lx47c81BVkWAqYmBkgR0JGJCQGBLxNjTNY-DYgCsEHs535-_ut30LmhsBlNEZcoQ9ySeWF5kmfglpIUdpNDkYxUy0RwZHwMWcDIe7PwEjpVHq3Pb-H9g_TLQJ23foLDCSc6W2cg8ibP9mkmJPbfYJT_jqQRYx16_TnbNQ6feYrDartPFFVk6mSMw46qTSXoKsru5WTd3evqyv39qLy_a-jhoJ65aPXf0Ujz-am9Cke39ePzTd8vX7whEHLYIkHdACbMGl8J8fLptRnEIBGWXeFTDmTmFGM115lDs0x3OdU4bWwYC6UqRxUTbmauazJdcnxqQYhERjD-OAw5yrT3v-rJg)





### 小结：

上面总结的一些**异步语境下的状态机模式**完全可以根据实际需求继续添加新的规则，这里就不展开了。但就目前为止我觉得已经是将平时开发的常见要素给囊括了，而且也介绍了基于这些模式的相关抽象，大家可以尝试使用这些抽象来设计自己的代码。





## 总结

前面的内容是从**用状态机表示控制流**的角度切入，来介绍rust中的状态机，以及展开了下我心目中的状态机模式。而且，我确信状态机就是解决开篇提到的问题的钥匙——将状态机视为一种具象化的、first class的控制流，利用trait等机制对其施以约束，我们就可以将程序中的大大小小控制流给管理起来。

而且，状态机还有其独特的优势：每个状态转移的前置条件，后置条件都可以清晰地定义出来，这样会更加方便地进行测试与验证。（当然这是在状态比较少的情况）



BTW，有些场景，其实并不适合状态机的发挥：

1. 太简单的场景，不需要状态机。这时候我们更倾向于与直接用语句这种更**线性**的表达。
2. 太复杂的场景，很难使用状态机。在同一个抽象层面上如果存在太多状态和条件，是一件十分难以设计和维护的事情。但这可能并不是状态机自身的问题，而是代码的设计出了问题，没有**控制**好复杂度（注意这里表达，没有说降低复杂度）
3. 变动的比较多的场景，很难使用状态机。改动总是会引起条件变动，从而破坏状态机的一些假设，我们可能需要重新调整状态机的结构，这是一个成本比较大的工作。



最后，我们使用rust几个原则来简单评判一下状态机模式吧：

* 可靠：是的，因为状态机能比较好的验证代码逻辑的正确性
* 高性能：不确定，需要看场景。
* 支持：是的，rust的机制能帮助我们去写出一个好的状态机
* 高产效：可能不，因为状态机的设计成本比较高，而且因为状态机形状各异不一定能到处兼容。
* 透明：不涉及。
* 通用：可能不，比如上面所说的几个场景可能不适合使用状态机模式。





***

一个程序是一个精密的机器，

* 状态机就像里面的齿轮，
* 程序的event loop就像机器的链条，
* 数据则是纸带

我们通过拉动链条驱动机器的内部的齿轮，对纸带上面的数据进行读写。。。



