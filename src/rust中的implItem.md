# Rust中的impl Item

> * [rust reference-implementations](https://doc.rust-lang.org/reference/items/implementations.html?highlight=coherence#implementations) impl item的语法描述
> * [RFC48-traits](https://rust-lang.github.io/rfcs/0048-traits.html) trait的最初的定义
> * [RFC447-no-unused-impl-parameters](https://rust-lang.github.io/rfcs/0447-no-unused-impl-parameters.html) 不允许impl中有未限定的泛型
> * [RFC1023-rebalancing-coherence](https://github.com/rust-lang/rfcs/blob/master/text/1023-rebalancing-coherence.md) trait impl的一致性原则的调整
> * [RFC2541-re-rebalancing-coherence](https://rust-lang.github.io/rfcs/2451-re-rebalancing-coherence.html) trait impl的一致性原则的再调整
> * [RFC0132-UFCS](https://github.com/rust-lang/rfcs/blob/master/text/0132-ufcs.md) Universal Function Call Syntax



之前写了几篇话题比较大的文章，现在又回来关注一下具体某个特性上。

估计用Rust写过库的人都可能遇到过这样的报错

1. > unconstrained type parameter

   ```rust
   trait Foo {}
   
   impl<A, F: Fn(A)> Foo for F {}
   
   // error[E0207]: the type parameter `A` is not constrained by the impl trait, self type, or predicates
   //  --> src/lib.rs:3:6
   //   |
   // 3 | impl<A, F: Fn(A)> Foo for F {}
   //   |      ^ unconstrained type parameter
   // 
   // For more information about this error, try `rustc --explain E0207`.
   ```

2. >  only traits defined in the current crate can be implemented for types defined outside of the crate）

   ```rust
   use std::num::NonZeroU64;
   
   impl TryFrom<NonZeroU64> for i64 {
       type Error = NonZeroU64;
       fn try_from(x: NonZeroU64) -> Result<i64, Self::Error> {
           Err(x)
       }
   }
   
   // error[E0117]: only traits defined in the current crate can be implemented for primitive types
   //  --> src/lib.rs:3:1
   //   |
   // 3 | impl TryFrom<NonZeroU64> for i64 {
   //   | ^^^^^-------------------^^^^^---
   //   | |    |                       |
   //   | |    |                       `i64` is not defined in the current crate
   //   | |    `NonZeroU64` is not defined in the current crate
   //   | impl doesn't use only types from inside the current crate
   //   |
   //   = note: define and implement a trait or new type instead
   ```
   
   
   

本文打算借着讲一下impl的一些规则顺带讲一下这块设计的原因，以及遇到这些问题的解决方法。



# impl 块的分类以及语法

Inherent Implementations

Trait Implementations

Universal Function Call Syntax

# Constrainted Parameter?

（原因）

（解决）



# impl 的一致性原则

rust有边界(函数边界、crate边界)



（原因）

（解决）