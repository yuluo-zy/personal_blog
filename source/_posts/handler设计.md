---
title: handler设计
date: 2023-06-01T11:30:21+08:00
tags:
- rust
- web框架
categories: 
- 自研web框架
---


## 基本类型定义


```rust
type HandlerResult = std::result::Result<(State, Response<Body>), (State, HandlerError)>;

pub type SimpleHandlerResult = std::result::Result<Response<Body>, HandlerError>;

pub type HandlerFuture = dyn Future<Output = HandlerResult> + Send;
```

这段代码是Rust编程语言中的类型定义代码片段。让我为你解释一下每个类型别名的含义：

1. `HandlerResult`：这是一个复合类型别名，它表示一个处理器函数的结果。它是`std::result::Result`类型的别名，该结果要么是一个元组 `(State, Response<Body>)`，表示成功的处理结果，其中`State`是某种状态类型，`Response<Body>`是HTTP响应类型；要么是一个元组 `(State, HandlerError)`，表示处理出错的结果，其中`HandlerError`是处理错误类型。

2. `SimpleHandlerResult`：这是一个简化版的处理器函数结果类型别名。它是`std::result::Result`类型的别名，该结果要么是一个 `Response<Body>`，表示成功的处理结果；要么是一个 `HandlerError`，表示处理出错的结果。

3. `HandlerFuture`：这是一个动态 trait 对象类型的别名。它表示由`HandlerService`返回的`Future`类型，该`Future`最终会产生`HandlerResult`类型的值作为输出。这里使用动态 trait 对象是为了在运行时灵活地处理不同类型的`Future`。

这些类型别名的目的是为了简化代码并提高可读性。通过使用类型别名，可以在代码中使用更具描述性的名称，而无需直接引用复杂的类型声明。


## 处理函数

```rust
pub trait Handler: Send {

    fn handle(self, state: State) -> Pin<Box<HandlerFuture>>;
}

impl<F, R> Handler for F
where
    F: FnOnce(State) -> R + Send,
    R: IntoHandlerFuture,
{
    fn handle(self, state: State) -> Pin<Box<HandlerFuture>> {
        self(state).into_handler_future()
    }
}

pub trait IntoHandlerFuture {

    fn into_handler_future(self) -> Pin<Box<HandlerFuture>>;
}

impl<T> IntoHandlerFuture for (State, T)
where
    T: IntoResponse,
{
    fn into_handler_future(self) -> Pin<Box<HandlerFuture>> {
        let (state, t) = self;
        let response = t.into_response(&state);
        future::ok((state, response)).boxed()
    }
}

impl IntoHandlerFuture for Pin<Box<HandlerFuture>> {
    fn into_handler_future(self) -> Pin<Box<HandlerFuture>> {
        self
    }
}

pub trait IntoResponse {
    /// Converts this value into a `hyper::Response`
    fn into_response(self, state: &State) -> Response<Body>;
}

impl IntoResponse for Response<Body> {
    fn into_response(self, _state: &State) -> Response<Body> {
        self
    }
}

impl<T, E> IntoResponse for ::std::result::Result<T, E>
where
    T: IntoResponse,
    E: IntoResponse,
{
    fn into_response(self, state: &State) -> Response<Body> {
        match self {
            Ok(res) => res.into_response(state),
            Err(e) => e.into_response(state),
        }
    }
}

impl<B> IntoResponse for (Mime, B)
where
    B: Into<Body>,
{
    fn into_response(self, state: &State) -> Response<Body> {
        (StatusCode::OK, self.0, self.1).into_response(state)
    }
}

impl<B> IntoResponse for (StatusCode, Mime, B)
where
    B: Into<Body>,
{
    fn into_response(self, state: &State) -> Response<Body> {
        response::create_response(state, self.0, self.1, self.2)
    }
}

// derive IntoResponse for Into<Body> types
macro_rules! derive_into_response {
    ($type:ty) => {
        impl IntoResponse for $type {
            fn into_response(self, state: &State) -> Response<Body> {
                (StatusCode::OK, mime::TEXT_PLAIN, self).into_response(state)
            }
        }
    };
}

// derive Into<Body> types - this is required because we
// can't impl IntoResponse for Into<Body> due to Response<T>
// and the potential it will add Into<Body> in the future
derive_into_response!(Bytes);
derive_into_response!(String);
derive_into_response!(Vec<u8>);
derive_into_response!(&'static str);
derive_into_response!(&'static [u8]);
derive_into_response!(Cow<'static, str>);
derive_into_response!(Cow<'static, [u8]>);

```

在这个代码片段中，使用 `Pin<Box<HandlerFuture>>` 来定义 `handle` 方法的返回类型。这是因为在 Rust 中，异步操作通常需要使用 `Pin` 和 `Box` 来进行安全处理。

`Pin` 是一种用于确保数据在内存中固定位置的指针类型。在异步上下文中，`Pin` 用于确保异步操作的数据不会被意外移动，从而确保其安全性。使用 `Pin` 可以防止数据被移动或释放，以满足异步操作的要求。

`Box` 则用于将值包装在堆上的盒子中，这样可以在编译时确定值的大小并在运行时动态分配内存。由于异步操作的返回类型大小在编译时是未知的，因此需要使用 `Box` 进行动态分配。

`Pin<Box<HandlerFuture>>` 表示一个被固定在内存中的指向异步操作结果的堆分配盒子。它允许异步操作返回一个 `Future` 对象，该对象在运行时具有不确定的大小，并且在内存中被固定以满足异步操作的安全要求。


