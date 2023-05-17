---
title: 文件存储缓存设计
date: 2023-05-14 19:57:24
categories: 
- 数据库设计
tags:
- page_file
- cache
---

## 文件存储设计

### 概论

这段代码实现了一个名为`FileReaderCache`的结构体，用于缓存`FileReader`对象，并在需要时从缓存中获取或初始化一个新的`FileReader`对象。

`FileReader`是一个用于读取文件的结构体，而`FileReaderCache`则是在`page_store`模块中使用的一个缓存对象。

`FileReaderCache`包含一个`LRUCache`类型的缓存对象`cache`，它是一个LRU（最近最少使用）缓存，用于保存`FileReader`对象。`_marker`是一个PhantomData类型的占位符，用于表示该结构体中有一个类型参数`E`，但在运行时并不需要实际传递该参数。

`new`方法创建并返回一个新的`FileReaderCache`对象，并初始化了缓存的大小限制。`get_with`方法用于从缓存中获取`FileReader`对象，如果缓存中已经有了，则直接返回；否则，它会使用一个异步的Future来初始化一个新的`FileReader`对象，并将其插入缓存中。`invalidate`方法用于从缓存中删除指定的`FileReader`对象，`stats`方法用于获取缓存的统计信息。

总之，`FileReaderCache`是一个通用的缓存对象，用于管理`FileReader`对象的生命周期，从而提高文件读取操作的性能。


### FileReader 设计

这段代码定义了一个名为 `FileReader` 的结构体

`FileReader` 结构体包含以下字段：

- `use_direct`：一个布尔值，指示是否使用直接 IO。
- `align_size`：一个整数，表示数据对齐的字节数。
- `file_size`：一个整数，表示文件的大小。
- `read_bytes`：一个计数器，用于统计已读取的字节数。

`FileReader` 实现了一个名为 `from` 的函数，该函数接收 `reader`、`use_direct`、`align_size` 和 `file_size` 四个参数，并返回一个 `FileReader` 的实例。

`FileReader` 还实现了一个名为 `read_exact_at` 的异步函数，该函数接收一个可变的字节数组 `buf` 和一个表示文件偏移量的 `req_offset` 参数，从文件中准确地读取 `buf.len()` 个字节到 `buf` 中。如果 `buf` 是空的，则函数直接返回。

如果 `use_direct` 为 `false`，则 `read_exact_at` 直接使用 `reader` 从文件中读取数据。如果 `use_direct` 为 `true`，则函数会执行以下操作：

1. 根据 `align_size` 对 `req_offset` 进行对齐，计算出对齐后的偏移量`align_offset`。这个偏移量是向下取正之后的位置
2. 计算 `offset_ahead` 这个偏移差是多少
2. 计算需要读取的对齐后的字节数 `align_buf_size`。需要读取的对齐缓冲区的长度，即对齐结束偏移量减去对齐开始偏移量的差值
3. 创建一个大小为 `align_buf_size` 的 `AlignBuffer`，并从中获取可变的字节数组 `read_buf`。
4. 调用 `inner_read_exact_at` 函数从 `reader` 中读取数据到 `read_buf` 中。
5. 将 `read_buf` 中的数据复制到 `buf` 中，偏移量为 `offset_ahead` 到 `offset_ahead + buf.len()`。！
6. 更新 `read_bytes` 的值。


最后，`FileReader` 还实现了一个名为 `read_block` 的异步函数，该函数接收一个名为 `block_handle` 的 `BlockHandle` 参数，表示创建指定长度的buf，然后根据偏移量来读取文件。

```rust
pub async fn read_block(&mut self, block_handle: BlockHandle) -> Result<Vec<u8>> {
        let mut buf = vec![0u8; block_handle.length as usize];
        self.read_exact_at(&mut buf, block_handle.offset).await?;
        Ok(buf)
    }
```

> 在 Rust 中，静态派发（Static Dispatch）和动态派发（Dynamic Dispatch）是两种不同的方法用于调用函数或方法。
静态派发是指在编译时确定调用的具体函数或方法。编译器根据编写的代码、类型信息和 trait 实现等静态信息来确定要调用的函数或方法。静态派发具有较低的运行时开销，因为编译器可以直接生成针对特定类型的优化代码，无需在运行时进行类型检查和分发。
动态派发是指在运行时根据对象的实际类型来确定要调用的函数或方法。编译器无法事先确定要调用的具体函数或方法，因此需要在运行时通过查找虚函数表（vtable）或类似的机制来确定要调用的函数或方法。动态派发通常与 trait 对象相关，因为 trait 对象是在运行时确定具体类型的。
静态派发适用于以下情况：
    - 函数或方法的调用目标在编译时是已知的，不会发生变化。
    - 函数或方法的调用目标是具体的类型，而不是 trait 对象。使用泛型来完成
动态派发适用于以下情况：
    - 函数或方法的调用目标在运行时可能是不确定的，例如使用 trait 对象来处理不同的类型。
    - 需要在运行时动态地选择调用不同的函数或方法。
在 Rust 中，默认情况下，函数和方法的调用使用静态派发。但是，通过使用 trait 对象，可以实现动态派发。使用 `dyn` 关键字标记的 trait 对象实现了动态派发，因为在运行时才确定调用的具体函数或方法。
总结：
    - 静态派发是在编译时确定调用的具体函数或方法，具有较低的运行时开销。
    - 动态派发是在运行时根据对象的实际类型确定调用的函数或方法，适用于需要在运行时动态选择调用不同函数或方法的情况。

### 原子计数 结构体

这段代码定义了一个名为 `Counter` 的结构体，用于实现一个计数器。下面逐个解释代码的含义：

```rust
#[derive(Debug)]
pub(crate) struct Counter(AtomicU64);
```
这段代码定义了一个包含单个字段的结构体 `Counter`，字段的类型是 `AtomicU64`。`AtomicU64` 是 Rust 提供的原子类型，用于实现原子级别的操作。`#[derive(Debug)]` 是一个派生属性，它会自动为结构体实现 `Debug` trait，以便在调试时打印结构体的信息。

```rust
impl Counter {
    pub(crate) const fn new(value: u64) -> Self {
        Self(AtomicU64::new(value))
    }
```
这个代码块实现了 `Counter` 结构体的一个关联函数 `new`。`new` 函数用于创建一个新的 `Counter` 实例，并初始化内部的 `AtomicU64` 字段。通过调用 `AtomicU64::new(value)` 创建 `AtomicU64` 实例，并将其作为参数传递给 `Self` 结构体的构造函数，最终返回一个 `Counter` 实例。

```rust
    pub(crate) fn get(&self) -> u64 {
        self.0.load(Ordering::Relaxed)
    }
```
这段代码定义了一个公共方法 `get`，用于获取当前计数器的值。它调用 `AtomicU64` 的 `load` 方法来获取内部的原子值，并指定 `Ordering::Relaxed` 作为原子顺序参数。这意味着对计数器的读取操作是无序的，没有任何同步保证或顺序限制。

```rust
    pub(crate) fn inc(&self) -> u64 {
        self.add(1)
    }
```
这个代码块实现了一个公共方法 `inc`，用于将计数器的值增加1。它调用 `add` 方法，传递参数 `1` 来实现增加操作。

```rust
    pub(crate) fn add(&self, n: u64) -> u64 {
        self.0.fetch_add(n, Ordering::Relaxed)
    }
```
这段代码定义了一个公共方法 `add`，用于将计数器的值增加指定的数量 `n`。它调用 `AtomicU64` 的 `fetch_add` 方法来进行原子的增加操作，并使用 `Ordering::Relaxed` 作为原子顺序参数。

```rust
impl Default for Counter {
    fn default() -> Self {
        Self::new(0)
    }
}
```
这段代码实现了 `Default` trait 的默认实现，用于创建一个默认的 `Counter` 实例。默认实现调用了之前定义的 `new` 方法，并传递初始值 `0`，以便创建一个初始计数器值为 `0` 的实例。

总结来说，这段代码定义了一个计数器结构体 `Counter`，使用 `AtomicU64` 实现了原子级别的计数操作。它提供了创建计数器实例、获取计数器值、增加等。


### 辅助函数

因为写入的时候需要对齐位置来提高ssd的读取

#### floor_to_block_lo_pos

 `floor_to_block_lo_pos` 函数用于将给定的位置向下取整到最接近的块边界位置。下面是函数的详细解释：

```rust
fn floor_to_block_lo_pos(pos: usize, align: usize) -> usize {
    pos - (pos & (align - 1))
}
```

参数解释：
    - `pos`：要进行取整的位置。
    - `align`：块的大小，即要对齐到的边界值。

计算过程：
    1. `(align - 1)`：计算块大小减 1 的结果。例如，如果块大小为 4096，则 `(align - 1)` 的值为 4095。
    2. `pos & (align - 1)`：将 `pos` 与 `(align - 1)` 进行按位与操作，即将 `pos` 与块大小减 1 进行按位与操作。这一步的目的是获取 `pos` 与块大小的低位部分，即与块对齐之后保留的部分。
    3. `pos - (pos & (align - 1))`：从 `pos` 中减去步骤 2 中计算得到的低位部分。这样就可以将 `pos` 向下取整到最接近的块边界位置。

具体示例：
假设有一个块大小为 4096 字节（align = 4096），我们想要将一个给定的位置向下取整到最接近的块边界位置。

假设给定的位置为 5000，我们可以按照以下步骤计算：

    1. `(align - 1)` 的值为 4095。
    2. `pos & (align - 1)` 的结果为 `5000 & 4095`，即 904。
    3. `pos - (pos & (align - 1))` 的结果为 `5000 - 864`，即 4096。

因此，5000 在向下取整到最接近的块边界位置时，结果为 4096。

#### ceil_to_block_hi_pos

'ceil_to_block_hi_pos(pos: usize, align: usize) -> usize' ：
这个函数用于将给定的位置 pos 向上取整到最接近的块边界位置。它通过将 pos 加上 align - 1，然后除以 align 并乘以 align 来实现。这样做可以将 pos 向上舍入到下一个块的起始位置。

```rust
#[inline]
pub(crate) fn ceil_to_block_hi_pos(pos: usize, align: usize) -> usize {
    ((pos + align - 1) / align) * align
}
```

### 缓冲区设计

这段代码定义了一个名为 `AlignBuffer` 的结构体。它用于创建和管理对齐的缓冲区。

结构体中的字段如下：

- `data`：一个非空的指针，指向分配的缓冲区内存块的起始位置。
- `layout`：用于描述内存布局的结构体 `Layout`，指定了缓冲区的大小和对齐要求。
- `size`：缓冲区的实际大小。

`AlignBuffer` 结构体有以下方法：

- `new(n: usize, align: usize)`：创建一个新的 `AlignBuffer` 实例。它接受两个参数，`n` 是所需的缓冲区大小，`align` 是对齐要求。它会根据对齐要求调整缓冲区的大小，并分配对应大小的内存块。如果分配失败，则会产生 panic。
- `len(&self) -> usize`：返回缓冲区的大小。
- `as_bytes(&self) -> &[u8]`：将缓冲区转换为不可变的字节数组切片。
- `as_bytes_mut(&mut self) -> &mut [u8]`：将缓冲区转换为可变的字节数组切片。

此外，结构体还实现了 `Drop` trait，用于在 `AlignBuffer` 实例被释放时释放分配的内存。

`unsafe impl Send for AlignBuffer` 和 `unsafe impl Sync for AlignBuffer` 分别实现了 `Send` 和 `Sync` trait，表示 `AlignBuffer` 类型是可安全地在多线程环境中进行发送和共享的。这是基于以下假设：所有对内部缓冲区的访问都不会出现别名重叠的情况。注意，这些 `unsafe` 实现需要开发者保证别名重叠不会发生，否则会导致不安全的行为。

```rust
pub(crate) struct AlignBuffer {
    data: std::ptr::NonNull<u8>,
    layout: Layout,
    size: usize,
}

impl AlignBuffer {
    pub(crate) fn new(n: usize, align: usize) -> Self {
        assert!(n > 0);
        let size = ceil_to_block_hi_pos(n, align);
        let layout = Layout::from_size_align(size, align).expect("Invalid layout");
        let data = unsafe {
            // Safety: it is guaranteed that layout size > 0.
            std::ptr::NonNull::new(std::alloc::alloc(layout)).expect("The memory is exhausted")
        };
        Self { data, layout, size }
    }

    #[inline]
    fn len(&self) -> usize {
        self.size
    }

    fn as_bytes(&self) -> &[u8] {
        // as_ptr 获取原始指针
        unsafe { std::slice::from_raw_parts(self.data.as_ptr(), self.size) }
    }

    pub(crate) fn as_bytes_mut(&mut self) -> &mut [u8] {
        unsafe { std::slice::from_raw_parts_mut(self.data.as_ptr(), self.size) }
    }
}

impl Drop for AlignBuffer {
    fn drop(&mut self) {
        unsafe {
            std::alloc::dealloc(self.data.as_ptr(), self.layout);
        }
    }
}

/// # Safety
///
/// [`AlignBuffer`] is [`Send`] since all accesses to the inner buf are
/// guaranteed that the aliases do not overlap.
unsafe impl Send for AlignBuffer {}

/// # Safety
///
/// [`AlignBuffer`] is [`Send`] since all accesses to the inner buf are
/// guaranteed that the aliases do not overlap.
unsafe impl Sync for AlignBuffer {}

```