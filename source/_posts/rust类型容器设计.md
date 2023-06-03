---
title: rust类型容器设计
date: 2023-06-03T22:23:06+08:00
tags:
- rust
- web框架
categories: 
- 自研web框架
---

## 介绍

`BorrowBag` 是一个类型安全的异构集合，具有零成本的添加和借用功能。

`BorrowBag` 允许存储任何值，并返回一个 `Handle`，以便稍后借用该值。由于 `BorrowBag` 是只能添加元素的，所以 `Handle` 值在 `BorrowBag` 的生命周期内始终有效。

核心点在于 可以存储具体的数据类型而不丢失类型信息， 并且在移动集合之后还能借用它们。


## 使用说明

```rust
#[derive(PartialEq, Debug)]
struct X;

#[derive(PartialEq, Debug)]
struct Y;

#[derive(PartialEq, Debug)]
struct Z;

let bag = BorrowBag::new();
let (bag, x_handle) = bag.add(X);
let (bag, y_handle) = bag.add(Y);
let (bag, z_handle) = bag.add(Z);

let x: &X = bag.borrow(x_handle);
assert_eq!(x, &X);
let y: &Y = bag.borrow(y_handle);
assert_eq!(y, &Y);
let z: &Z = bag.borrow(z_handle);
assert_eq!(z, &Z);

// Can borrow multiple times using the same handle
let x: &X = bag.borrow(x_handle);
assert_eq!(x, &X);

```

在同一个 集合中， 添加多个 结构体， 并 可以使用句柄来完成借用和访问。


## 句柄的设计


```rust
use std::marker::PhantomData;
```
这行代码从 `std::marker` 模块中导入了 `PhantomData` 类型。`PhantomData` 是一个标记类型，用于指示特定结构体中使用了一个泛型类型参数，即使它在结构体的字段中并没有实际使用。

```rust
pub struct Skip;
```
这行代码定义了一个公共结构体 `Skip`。这个结构体用作导航类型，在 `BorrowBag` 上下文中描述一个被跳过的元素。

```rust
pub struct Take;
```
这行代码定义了一个公共结构体 `Take`。这个结构体用作导航类型，在 `BorrowBag` 上下文中描述目标元素。

```rust
pub struct Handle<T, N> {
    phantom: PhantomData<(T, N)>,
}
```
这行代码定义了一个名为 `Handle` 的公共结构体，它有两个泛型参数 `T` 和 `N`。`Handle` 结构体是可以与 `BorrowBag` 一起使用的值，用于借用已添加的元素。`phantom` 字段的类型为 `PhantomData<(T, N)>`，这是为了在类型系统中保留 `T` 和 `N` 的信息。

### PhantomData 使用

`PhantomData` 是一个用于标记类型参数的零大小类型。它在泛型代码中用于指示存在某个类型参数，即使在运行时并不实际使用该参数的值。

`PhantomData` 的作用之一是在类型系统中保留类型信息，以便进行静态检查和类型推断。例如，在 `Handle<T, N>` 结构体中，使用 `PhantomData<(T, N)>` 字段的目的是为了在类型系统中表明 `T` 和 `N` 的存在，即使实际上在结构体中并没有使用这些值。这样可以在编译时进行类型检查，确保在使用 `Handle` 时传递正确的类型参数。

下面是 `PhantomData` 的使用示例：

```rust
use std::marker::PhantomData;

struct Container<T> {
    value: T,
    phantom: PhantomData<T>,
}

fn main() {
    let container: Container<i32> = Container {
        value: 42,
        phantom: PhantomData,
    };

    // 使用 container.value 进行操作
}
```

在这个示例中，`Container` 结构体包含一个值 `value` 和一个 `PhantomData<T>` 字段，用于标记类型参数 `T` 的存在。即使 `PhantomData` 字段不包含任何实际的数据，但它确保在 `Container` 实例中保留了类型信息。这对于类型检查和类型推断是非常有用的。

请注意，使用 `PhantomData` 需要小心，确保正确地使用和理解它的目的。它通常用于高级的泛型编程和类型系统抽象中，而在一般情况下可能并不需要使用它。

### handle 派生 clone 和 copy

```rust
pub(crate) fn new_handle<T, N>() -> Handle<T, N> {
    Handle {
        phantom: PhantomData,
    }
}
```
这个函数定义了一个 `new_handle` 函数，用于创建任意给定类型的 `Handle`。函数返回一个具有类型 `Handle<T, N>` 的新实例，其中 `phantom` 字段的值被设为 `PhantomData`，以保留类型参数 `T` 和 `N` 的信息。

```rust
impl<T, N> Clone for Handle<T, N> {
    fn clone(&self) -> Handle<T, N> {
        new_handle()
    }
}
```
这个 `impl` 块实现了 `Clone` trait，允许对 `Handle<T, N>` 类型的值进行克隆。`clone` 方法创建并返回一个与原始 `Handle` 值相同的新实例。

```rust
impl<T, N> Copy for Handle<T, N> {}
```
这个 `impl` 块实现了 `Copy` trait，允许对 `Handle<T, N>` 类型的值进行复制。这里没有提供具体的实现，因为 `PhantomData` 字段不需要实际复制。由于 `Handle` 中没有其他字段，可以自动派生 `Copy` trait。

## Lookup 查找实现


下面是对相关代码的详细解释：

```rust
use super::handle::{Skip, Take};
```
这行代码导入了 `handle` 模块中的 `Skip` 和 `Take` 结构体。这些结构体是用作导航类型，用于描述在 `BorrowBag` 上下文中是跳过元素还是获取目标元素。

```rust
pub trait Lookup<T, N> {
    fn get_from(&self) -> &T;
}
```
这个 trait 定义了一个名为 `Lookup` 的公共 trait。它有两个泛型参数 `T` 和 `N`，用于指定要查找的值的类型和导航类型。该 trait 包含一个方法 `get_from`，它返回一个对类型 `T` 的引用。这个 trait 可以用来约束一个 `Handle` 参数，以确保它可以与相应的 `BorrowBag` 一起使用。

```rust
impl<T, U, V, N> Lookup<T, (Skip, N)> for (U, V)
where
    V: Lookup<T, N>,
{
    fn get_from(&self) -> &T {
        self.1.get_from()
    }
}
```
这个 `impl` 块实现了 `Lookup<T, (Skip, N)>` trait，用于将 `(U, V)` 类型的值转换为 `T` 类型的引用。条件是 `V` 类型实现了 `Lookup<T, N>` trait。当导航类型为 `Skip` 时，我们通过调用 `self.1.get_from()` 来获取目标元素的引用。这个实现允许在元组中递归地查找目标元素。

```rust
impl<T, V> Lookup<T, Take> for (T, V) {
    fn get_from(&self) -> &T {
        &self.0
    }
}
```
这个 `impl` 块实现了 `Lookup<T, Take>` trait，用于将 `(T, V)` 类型的值转换为 `T` 类型的引用。当导航类型为 `Take` 时，我们直接返回元组的第一个元素 `&self.0`，也就是目标元素的引用。

这些实现允许通过 `Lookup` trait 来从 `BorrowBag` 中获取存储的值。根据导航类型 `N` 的不同，使用不同的实现来选择适当的方式来获取目标元素的引用。

## 添加操作

下面是对相关代码的详细解释：

```rust
use super::handle::{new_handle, Handle, Skip, Take};
use super::lookup::Lookup;
```
这部分代码导入了 `handle` 模块中的 `new_handle` 函数和 `Handle`、`Skip`、`Take` 结构体，以及 `lookup` 模块中的 `Lookup` trait。

```rust
pub trait Append<T> {
    type Output: PrefixedWith<Self> + Lookup<T, Self::Navigator>;
    type Navigator;

    fn append(self, t: T) -> (Self::Output, Handle<T, Self::Navigator>);
}
```
这是一个名为 `Append` 的 trait，用于描述将元素 `T` 添加到 `BorrowBag` 中后的结果。它有两个关联类型：`Output` 和 `Navigator`。`Output` 指定了添加元素后的新的 `BorrowBag` 类型，而 `Navigator` 则指定了如何借用添加的元素。该 trait 还定义了一个 `append` 方法，用于将元素添加到集合中并返回新的集合和一个可以借用添加的元素的 `Handle`。

```rust
impl<T, U, V> Append<T> for (U, V)
where
    V: Append<T>,
{
    type Output = (U, <V as Append<T>>::Output);
    type Navigator = (Skip, <V as Append<T>>::Navigator);

    fn append(self, t: T) -> (Self::Output, Handle<T, Self::Navigator>) {
        let (u, v) = self;
        ((u, v.append(t).0), new_handle())
    }
}
```
这个 `impl` 块为元组 `(U, V)` 实现了 `Append<T>` trait，其中 `V` 类型也实现了 `Append<T>` trait。这意味着当前元组是一个多个元素组成的序列，通过将元素 `T` 添加到 `V` 中来添加元素。`Output` 类型是 `(U, <V as Append<T>>::Output)`，即当前元组的头部 `U` 和将 `T` 添加到 `V` 后得到的新集合的输出类型。`Navigator` 类型是 `(Skip, <V as Append<T>>::Navigator)`，即当前元组的导航类型为 `Skip`，并在 `V` 中使用 `V` 的导航类型。在 `append` 方法中，我们首先将当前元组解构为头部 `u` 和尾部 `v`，然后调用 `v.append(t)` 将元素 `T` 添加到尾部，并获取新集合和相应的 `Handle`。最后，我们通过构建一个新的元组 `(u, v.append(t).0)` 和调用 `new_handle()` 来返回新集合和一个新的 `Handle`。

```rust
impl<T> Append<T> for () {
    type Output = (T, ());
    type Navigator = Take;

    fn append(self, t: T) -> (Self::Output, Handle<T, Self::Navigator>) {
        ((t, ()), new_handle())
    }
}
```
这个 `impl` 块为 `()`（空元组）实现了 `Append<T>` trait，表示在添加元

素时达到了集合的末尾。`Output` 类型是 `(T, ())`，即新集合的类型为元素 `T` 和空元组。`Navigator` 类型是 `Take`，表示在导航时直接使用当前元素。在 `append` 方法中，我们创建一个新的元组 `((t, ()), new_handle())`，其中元素 `T` 是添加的元素，而空元组表示集合的末尾。最后，我们返回新集合和一个新的 `Handle`。

## 元素位置固定性证明


```rust
pub trait PrefixedWith<T>
where
    T: ?Sized,
{
}

impl<U, V0, V1> PrefixedWith<(U, V0)> for (U, V1) where V1: PrefixedWith<V0> {}

// 类似 （U, （）） for ( U, (U, ()))
impl<U> PrefixedWith<()> for (U, ()) {}
```
这是一个名为 `PrefixedWith` 的 trait，用于提供证明已有列表元素不会移动的功能，从而保证现有的 `Handle` 值继续有效。该 trait 没有定义方法，而是作为一个标记 trait。它有一个泛型参数 `T`，表示要前缀的类型。为了实现 `PrefixedWith<T>` trait，需要为当前元组类型和 `T` 之间的每一层关系都实现 `PrefixedWith` trait。这些实现确保了在添加新元素时，现有的元素保持不变，并且没有移动。

`PrefixedWith` trait 的目的是提供一个机制来证明添加新元素时现有元素不会移动。这是通过 trait 的实现关系来实现的。

它只是作为一个标记 trait 存在。它的目的是为了建立类型之间的关联，并提供一个证明机制，用于表明一个类型是由另一个类型作为前缀和尾部组成的。

这个实现语句是为了实现 `PrefixedWith` trait，它指定了当一个元组类型 `(U, V1)` 的尾部类型 `V1` 实现了 `PrefixedWith<V0>` trait 时，`(U, V1)` 类型就可以被认为是由 `(U, V0)` 类型的元组前缀和类型为 `V0` 的尾部组成的。

这个实现语句指定了当一个元组类型 (U, V1) 的尾部类型 V1 实现了 PrefixedWith<V0> trait 时，(U, V1) 类型就可以被认为是由 (U, V0) 类型的元组前缀和类型为 V0 的尾部组成的。

这个实现使用了 Rust 中的泛型类型参数 U、V0 和 V1，它们表示元组的第一个元素、第二个元素的前一个元素和第二个元素的类型。这个实现通过嵌套的 trait 约束 V1: PrefixedWith<V0> 来建立 V1 是 V0 的前缀的关联。

具体来说，当 `V1` 实现了 `PrefixedWith<V0>` trait 时，我们可以断言 `(U, V1)` 类型的元组由 `(U, V0)` 类型的元组前缀和类型为 `u` 的尾部组成。

这个实现的作用是递归地构建了一个元组类型链，通过将 `(U, V0)` 类型的元组作为前缀，不断添加尾部类型 `U`，直到达到最后一个尾部类型为 `()` 的元组。 ，空元组 () 是任何类型的前缀。

举个例子，假设有以下的类型链：

```rust
type MyTuple = (u8, (u16, (u32, ())));
```

根据 `PrefixedWith` trait 的实现，我们可以将 `MyTuple` 类型解读为：

```rust
(MyTuple) -> (u8, MyTuple0)
(MyTuple0) -> (u16, MyTuple1)
(MyTuple1) -> (u32, MyTuple2)
(MyTuple2) -> ()
```

这里，`MyTuple` 类型是由 `(u8, MyTuple0)` 组成的，其中 `MyTuple0` 是 `(u16, MyTuple1)`，依此类推，直到 `MyTuple2` 是 `(u32, ())`。

这样，通过使用 `PrefixedWith` trait 的实现关系，我们可以在编译时建立元组类型之间的连接，并确保在添加新元素时不会破坏现有元素的结构。

通过这种递归的方式，我们可以在类型系统中构建出一系列的嵌套元组，其中每个元组都通过前缀和尾部的方式连接起来。这样，我们就可以确保在添加新元素时，现有元素的内存位置不会发生变化，从而保证现有的 `Handle` 值继续有效。

这种原理可以被称为结构共享（structural sharing），它利用了 Rust 的类型系统和 trait 实现来提供静态的类型保证。通过将类型的结构信息编码到 trait 中，并使用 trait 的实现关系来建立类型之间的关联，我们可以在编译时进行验证和推理，以保证添加新元素时的不变性。

## 完整实现

```rust
use std::marker::PhantomData;


#[derive(PartialEq, Debug)]
struct X;

#[derive(PartialEq, Debug)]
struct Y;

#[derive(PartialEq, Debug)]
struct Z;
fn main() {
    let bag = BorrowBag::new();
    let (bag, x_handle) = bag.add(X);
    let (bag, y_handle) = bag.add(Y);
    let (bag, z_handle) = bag.add(Z);
    let (bag, z2_handle) = bag.add(Z);

    let z: &Z = bag.borrow(z_handle);
    assert_eq!(z, &Z);
    let x: &X = bag.borrow(x_handle);
    assert_eq!(x, &X);
    let y: &Y = bag.borrow(y_handle);
    assert_eq!(y, &Y);
    let z: &Z = bag.borrow(z2_handle);
    assert_eq!(z, &Z);

    // 使用 container.value1 和 container.value2 进行操作
}

struct Skip;

struct Take;

struct Handle<T,N> {
    phantom: PhantomData<(T,N)>
}

fn new_handle<T,N>() -> Handle<T,N> {
    Handle {
        phantom: PhantomData
    }
}


impl<T, N> Clone for Handle<T, N> {
    fn clone(&self) -> Self {
        new_handle()
    }
}

impl<T,N> Copy for Handle<T, N> {}

trait LookUp<T,N> {
    fn get_from(&self) -> &T;
}

impl<T, N> LookUp<T,Take> for (T, N) {
    fn get_from(&self) -> &T {
        &self.0
    }
}

impl<T, U, V: LookUp<T, N>, N> LookUp<T, (Skip, N)> for (U, V)
{
    fn get_from(&self) -> &T {
        &self.1.get_from()
    }
}

trait PrefixedWith<T> where T: ?Sized {}

impl<U> PrefixedWith<()> for (U, ()) {}

impl<U, V0, V1: PrefixedWith<V0>> PrefixedWith<(U, V0)> for (U, V1)  {}

trait Append<T> {
    type Output: PrefixedWith<Self> + LookUp<T, Self::Navigator>;
    type Navigator;

    fn append(self, t: T) -> (Self::Output, Handle<T, Self::Navigator>);
}

impl<T> Append<T> for () {
    type Output = (T, ());
    type Navigator = Take;

    fn append(self, t: T) -> (Self::Output, Handle<T, Self::Navigator>) {
        ((t, ()), new_handle())
    }
}

impl<T, U, V: Append<T>> Append<T> for (U, V) {
    type Output = (U, V::Output);
    type Navigator = (Skip, V::Navigator);

    fn append(self, t: T) -> (Self::Output, Handle<T, Self::Navigator>) {
        let (u, v) = self;
        ((u, v.append(t).0), new_handle())
    }
}

#[derive(Default)]
struct BorrowBag<V> {
    v: V
}

impl BorrowBag<()> {
    pub fn new() -> Self {
        Self {
            v: (),
        }
    }
}

impl<V> BorrowBag<V> {
    pub fn add<T>(self, t: T) ->( BorrowBag<V::Output>, Handle<T, V::Navigator>)
    where V: Append<T> {
       let (v, handle) = Append::append(self.v, t);

        (BorrowBag{v}, handle)
    }

    pub fn borrow<T, N>(&self, _handler: Handle<T, N>) -> &T
    where V: LookUp<T, N>{
        LookUp::<T, N>::get_from(&self.v)
    }
}



```

总体来说 就是用来 handle 来保存便利顺序， 通过类型的模式匹配来 完成查询操作

