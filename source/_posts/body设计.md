---
title: body设计
date: 2023-05-24T01:16:29+08:00
tags:
- rust
- web框架
categories: 
- 自研web框架
---

## 基础包


1. http-body：
   "http-body"是一个Rust库，用于处理HTTP请求和响应体的抽象。它提供了一个通用的接口，用于读取HTTP消息的主体部分，无论它是一个完整的字节数组、一个流式的数据块还是一个异步的迭代器。它还包含了一些方便的工具函数和实用程序，用于处理HTTP请求和响应体的常见操作，例如读取和解析表单数据、处理分块传输编码等。"http-body"库是构建HTTP服务器或客户端时常用的工具之一。

2. http-body-util：
   "http-body-util"是与上述"http-body"库相关的辅助工具库。它提供了一些附加功能和实用程序，用于处理和操作HTTP请求和响应体。具体来说，它可能包含一些函数和类型，用于解析和序列化HTTP消息体、处理压缩和解压缩、处理JSON或其他格式的数据、处理文件上传等。这个库的目标是简化与HTTP消息体相关的常见任务，提供更高级的功能和抽象。

在 hyper 1.0 中， 原来版本的 hyper::Body 已经被取消， 取而代之的是一个 `http_body的 pub trait Body`, 所以我们需要封装一个 Body 来对于 requests and responses的请求体进行抽象


###  工具函数

```rust
type BoxBody = http_body_util::combinators::UnsyncBoxBody<Bytes, Error>;
```

这一行代码定义了一个类型别名`BoxBody`，它是`http_body_util`包中combinators模块的`UnsyncBoxBody`类型的特化版本。UnsyncBoxBody是一个用于包装异步（不可同步）的HTTP消息体的类型，其中Bytes是表示字节数据的类型，Error是错误类型。

在设计中， 我们的代码主要是跑在 async 中 所以我们需要使用`UnsyncBoxBody`进行包装， 来固定数据。 如果需要进行同步操作 可以尝试 `BoxBody` 类型


```rust
fn boxed<B>(body: B) -> BoxBody
    where
        B: http_body::Body<Data = Bytes> + Send + 'static,
        B::Error: Into<BoxError>,
{
    try_downcast(body).unwrap_or_else(|body| body.map_err(Error::new).boxed_unsync())
}
```

这是一个名为`boxed`的函数定义，它接受一个泛型参数`B`，并返回一个`BoxBody`类型的值。

- `B: http_body::Body<Data = Bytes> + Send + 'static`：这是一个泛型约束，要求`B`实现`http_body::Body` trait，并且其中的`Data`类型必须是`Bytes`。同时，`B`必须是可发送到其他线程（实现`Send` trait）且具有'static生命周期。

- `B::Error: Into<BoxError>`：这是另一个泛型约束，要求`B`中的错误类型实现了`Into<BoxError>` trait，以便可以将其转换为`Box<dyn Error>`类型的错误。

该函数的主要功能是将输入的`body`进行尝试向下转型（downcast）为`T`类型，如果转型成功，则返回转型后的值。如果转型失败，则对输入的`body`进行错误映射和包装，并返回一个异步的`BoxBody`类型。
> `boxed_unsync()` 是 http_body_util 包中的一个方法，用于将实现了 http_body::Body trait 的异步数据流（如 UnsyncBoxBody）封装在一个异步的 'Box<dyn http_body::Body>' 类型中。这个函数的目的是提供一种简便的方式来将异步数据流包装为具有通用 trait 的异步对象。

```rust
pub(crate) fn try_downcast<T, K>(k: K) -> Result<T, K>
    where
        T: 'static,
        K: Send + 'static,
{
    let mut k = Some(k);
    if let Some(k) = <dyn std::any::Any>::downcast_mut::<Option<T>>(&mut k) {
        Ok(k.take().unwrap())
    } else {
        Err(k.unwrap())
    }
}
```

这是一个名为`try_downcast`的函数定义，它接受两个泛型参数`T`和`K`，并返回一个`Result<T, K>`类型的结果。

该函数的目的是尝试将输入的`k`值向下转型为`T`类型。它首先通过将`k`包装在`Option`中，以便可以安全地使用`take()`方法获取内部值。然后，它使用`dyn std::any::Any`的`downcast_mut`方法尝试将`k`转型为`Option<T>`类型的可变引用。如果转型成功，它会返回转型后的值。否则，它会通过`unwrap()`获取`k`的内部值并作为错误返回。

在这个函数中，T 参数是通过类型推断来确定的，而不是在运行时确定的。编译器根据函数的调用上下文和参数类型推断出 T 的具体类型。

总结一下，这些代码是

为了提供一些工具函数和类型别名，用于处理HTTP请求和响应体。`boxed`函数将输入的HTTP消息体封装在一个异步的`BoxBody`类型中，并提供了一些类型约束以确保正确性。`try_downcast`函数用于尝试将值向下转型，并返回转型结果或错误。