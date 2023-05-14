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

1. 根据 `align_size` 对 `req_offset` 进行对齐，计算出对齐后的偏移量 `align_offset`。
2. 计算需要读取的对齐后的字节数 `align_buf_size`。
3. 创建一个大小为 `align_buf_size` 的 `AlignBuffer`，并从中获取可变的字节数组 `read_buf`。
4. 调用 `inner_read_exact_at` 函数从 `reader` 中读取数据到 `read_buf` 中。
5. 将 `read_buf` 中的数据复制到 `buf` 中，偏移量为 `offset_ahead` 到 `offset_ahead + buf.len()`。
6. 更新 `read_bytes` 的值。

`FileReader` 还实现了一个名为 `inner_read_exact_at` 的异步函数，该函数接收 `r`、`buf` 和 `pos` 三个参数，其中 `r` 是实现了 `PositionalReader` trait 的实例，`buf` 是一个可变的字节数组，`pos` 表示读取数据的偏移量。该函数使用循环从 `r` 中读取数据到 `buf` 中，直到 `buf` 中的数据被完全填充。如果读取数据时发生了 `UnexpectedEof` 错误，则函数返回该错误。如果读取数据时发生了 `Interrupted` 错误，则函数会忽略该错误并继续读取数据。如果读取数据时发生了其他类型的错误，则函数返回该错误。

最后，`FileReader` 还实现了一个名为 `read_block` 的异步函数，该函数接收一个名为 `block_handle` 的 `BlockHandle` 参数，表示要读

