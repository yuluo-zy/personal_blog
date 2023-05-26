---
title: status 上下文设计
date: 2023-05-26T10:25:13+08:00
tags:
- rust
- web框架
categories: 
- 自研web框架
---

## 核心结构定义

```rust
#[derive(Default)]
struct IdHasher(u64);

impl Hasher for IdHasher {
    #[inline]
    fn finish(&self) -> u64 {
        self.0
    }

    fn write(&mut self, _: &[u8]) {
        unreachable!("TypeId calls write_u64");
    }

    #[inline]
    fn write_u64(&mut self, id: u64) {
        self.0 = id;
    }
}

pub struct State {
    data: HashMap<TypeId, Box<dyn Any + Send>, BuildHasherDefault<IdHasher>>,
}

```

上面的代码定义了一个名为 `IdHasher` 的结构体和一个名为 `State` 的结构体。

`IdHasher` 结构体实现了 `Hasher` trait，用于自定义哈希算法。它具有一个字段 `0`，默认情况下为 0。`finish` 方法返回该字段的值作为哈希结果。`write` 方法是一个未实现的方法，它会产生一个不可达的错误，因为我们不希望在 `TypeId` 调用 `write_u64` 之外的地方调用它。`write_u64` 方法将给定的 `id` 写入到 `self.0` 字段中。

`State` 结构体具有一个字段 `data`，它是一个 `HashMap`，其中键是 `TypeId`，值是 `Box<dyn Any + Send>`。`TypeId` 是用于表示类型的 Rust 标准库类型之一。`Box<dyn Any + Send>` 是一个可存储任意类型的盒子，同时保证了类型是 `Send` 的，即可跨线程传递。

`HashMap` 的哈希算法是通过 `BuildHasherDefault<IdHasher>` 指定的，默认情况下使用 `IdHasher` 作为哈希算法。

综上所述，这段代码定义了一个 `State` 结构体，其中包含一个以 `TypeId` 为键、`Box<dyn Any + Send>` 为值的哈希表。它使用了自定义的哈希算法 `IdHasher`，以及 Rust 的类型系统和 trait 来实现类型安全和灵活性。

在 Rust 中，HashMap 使用哈希算法来确定键值对在内部存储结构中的位置，以便快速查找和访问。默认情况下，HashMap 使用的哈希算法是基于 `std::hash::Hasher trait` 实现的。

但是，在某些情况下，我们可能需要自定义哈希算法以满足特定的需求。例如，在这个例子中，State 结构体的 data 字段是一个 HashMap，它的键是 TypeId 类型，值是任意类型的 `Box<dyn Any + Send>`。

为了确保在 HashMap 中正确使用 TypeId 作为键，并且每个 TypeId 对应的哈希结果是唯一的，我们需要实现一个自定义的哈希算法。这样可以确保 TypeId 的哈希值在不同的实例之间不会冲突。

通过实现自定义的哈希算法 IdHasher，我们可以控制 TypeId 的哈希结果，以适应我们的特定需求。在这个例子中，简单地将 TypeId 直接作为哈希结果，确保了每个 TypeId 对应的哈希值是唯一的，并且不会与其他类型的哈希值冲突。

## 核心数据 trait 实现

```rust
pub trait StateData: Any + Send {}
```

对于我们 上下文 内容的存储， 我们现在简单实现为 box<dyn trait>, 这种实现借用了 rust 中动态分发的特性， 但是存在了一定的寻址损耗。

## 生命周期

创建过程如下： 

这段代码于从 HTTP 请求 `Request<Body>` 创建一个 `State` 对象。

首先，它创建一个空的 `State` 对象，使用 `Self::new()` 来创建。接着，通过 `put_client_addr` 函数将 `client_addr` 存储到 `state` 中。

然后，通过 `req.into_parts()` 将请求拆解为请求的各个部分，即 `request::Parts` 和 `body`。`request::Parts` 结构体包含了请求的方法、URI、协议版本、头部信息等。这些部分分别赋值给相应的变量，如 `method`、`uri`、`version`、`headers`、`extensions`。

接下来，将 `uri.path()` 创建为 `RequestPathSegments` 对象，并将其放入 `state` 中。同样地，将 `method`、`uri`、`version`、`headers`、`body` 依次放入 `state` 中。

如果在 `extensions` 中存在类型为 `OnUpgrade` 的值，那么将其移除并放入 `state` 中。

最后，使用 `set_request_id` 函数设置请求 ID，并在日志中输出请求 ID 和当前线程 ID。

完成上述步骤后，函数返回创建好的 `state` 对象。

整体来说，这段代码是根据 HTTP 请求的各个部分，将相关信息存储到 `State` 对象中，以便后续处理和使用。

```rust
 pub fn from_request(req: Request<Body>, client_addr: SocketAddr) -> Self {
        let mut state = Self::new();

        put_client_addr(&mut state, client_addr);

        let (
            request::Parts {
                method,
                uri,
                version,
                headers,
                mut extensions,
                ..
            },
            body,
        ) = req.into_parts();

        // 添加静态常量池

        // 将 请求体 解包然后 进行复制
        state.put(RequestPathSegments::new(uri.path()));
        state.put(method);
        state.put(uri);
        state.put(version);
        state.put(headers);
        state.put(body);

        if let Some(on_upgrade) = extensions.remove::<OnUpgrade>() {
            state.put(on_upgrade);
        }

        {
            let request_id = set_request_id(&mut state);
            // 设置请求ID
            debug!(
                "[DEBUG][{}][Thread][{:?}]",
                request_id,
                std::thread::current().id(),
            );
        };

        state
    }

```

## 剩余方法概览


```rust
pub fn put<T>(&mut self, t: T)
where
    T: StateData,
{
    let type_id = TypeId::of::<T>();
    trace!(" inserting record to state for type_id `{:?}`", type_id);
    // 上下文的数据 都是存放到堆内存中， 相同类型大 数据会出现数值覆盖情况
    self.data.insert(type_id, Box::new(t));
}

    pub fn has<T>(&self) -> bool
    where
        T: StateData,
    {
        let type_id = TypeId::of::<T>();
        self.data.get(&type_id).is_some()
    }

       pub fn try_borrow<T>(&self) -> Option<&T>
    where
        T: StateData,
    {
        let type_id = TypeId::of::<T>();
        trace!(" borrowing state data for type_id `{:?}`", type_id);

        // 如果内部是 对应泛型实例， 则使用对应 的 引用返回数据， 返回借用， 是存在着
        self.data.get(&type_id).and_then(|b| b.downcast_ref::<T>())
    }

        pub fn borrow<T>(&self) -> &T
    where
        T: StateData,
    {
        // 直接返回对应的数据 而不是返回OPen的类型
        self.try_borrow()
            .expect("required type is not present in State container")
    }

        pub fn try_take<T>(&mut self) -> Option<T>
    where
        T: StateData,
    {
        let type_id = TypeId::of::<T>();
        trace!(
            " taking ownership from state data for type_id `{:?}`",
            type_id
        );
        self.data
            .remove(&type_id)
            .and_then(|b| b.downcast::<T>().ok())
            .map(|b| *b) //  // 转换成 option<T>
    }

```

这些函数是 `State` 结构体的方法，用于在 `State` 存储中操作和访问数据。

- `put`: 将一个值存储到 `State` 中。如果相同类型的值已经存在，则会覆盖之前的值。
- `has`: 判断 `State` 存储中是否存在指定类型的值。
- `try_borrow`: 尝试以不可变借用的方式获取存储中指定类型的值的引用。如果值不存在，则返回 `None`。
- `borrow`: 借用 `State` 存储中指定类型的值的引用。如果值不存在，则会产生 panic。
- `try_borrow_mut`: 尝试以可变借用的方式获取存储中指定类型的值的可变引用。如果值不存在，则返回 `None`。
- `borrow_mut`: 借用 `State` 存储中指定类型的值的可变引用。如果值不存在，则会产生 panic。
- `try_take`: 尝试将存储中指定类型的值取出并返回所有权。如果值不存在，则返回 `None`。
- `take`: 将存储中指定类型的值取出并返回所有权。如果值不存在，则会产生 panic。

这些方法允许在 `State` 存储中进行数据的存储、访问和所有权的转移。通过 `put` 方法可以将值存储到 `State` 中，然后可以使用 `has` 方法检查值是否存在，使用 `borrow` 或 `borrow_mut` 方法进行借用或可变借用访问。如果需要获取所有权，可以使用 `try_take` 或 `take` 方法。这些方法提供了对 `State` 存储中数据的灵活操作方式，方便在处理请求时存储和获取必要的数据。

以下是 `State` 结构体中核心技术点的详细解释：

1. **泛型和类型擦除**：`State` 使用泛型来实现对不同类型数据的存储和访问。在 `put` 方法中，使用了泛型参数 `T` 来接收传入的值，并使用 `Box` 将值进行类型擦除，然后存储到 `data` 字典中。在 `borrow`、`borrow_mut` 和其他方法中，通过类型擦除后的 `Box` 对象，使用 `downcast_ref` 和 `downcast_mut` 方法将其转换回原始类型的引用。这种类型擦除的技术允许 `State` 存储和操作不同类型的数据，而不需要提前指定具体的类型。

2. **Type ID 和数据索引**：为了实现对不同类型数据的索引和查找，`State` 使用了 `TypeId` 和 `data` 字典。在 `put` 方法中，通过 `TypeId::of::<T>()` 获取到类型 `T` 的唯一标识符 `type_id`，然后使用 `type_id` 作为键将类型擦除后的值存储到 `data` 字典中。在其他方法中，通过 `type_id` 从 `data` 字典中获取值，并进行类型转换。这种机制允许在 `State` 中存储多个不同类型的数据，并根据类型进行索引和访问。

3. **借用和所有权管理**：`State` 提供了借用和所有权的管理机制，通过 `borrow`、`borrow_mut`、`try_borrow`、`try_borrow_mut`、`take` 和 `try_take` 方法进行数据的借用和所有权的转移。这些方法通过引用和 `Option` 类型的返回值，提供了对存储数据的灵活访问方式。借用规则确保在特定作用域中只有一个可变引用或多个不可变引用，并在编译时检查所有权转移的有效性。

4. **错误处理**：部分方法如 `borrow` 和 `try_take` 在数据不存在时会产生 panic，而另一些方法如 `try_borrow` 和 `try_borrow_mut` 则返回 `Option` 类型的值以处理可能的失败情况。这样的设计允许根据具体的需求选择合适的错误处理机制，从而提高代码的健壮性和可靠性。

综上所述，`State` 的核心技术点包括泛型和类型擦除、Type ID 和数据索引、借用和所有权管理以及错误处理。这些技术点共同实现了对不同类型数据的存储、访问和管理，使得 `State` 在 Web 应用程序开发中具有高度的灵活性和可扩展性。