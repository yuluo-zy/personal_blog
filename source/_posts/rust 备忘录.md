---
title: rust 备忘录
date: 2023-05-26T11:57:14+08:00
tags:
- rust
---

### 函数式

这里是关于`FnOnce`、`Fn`和`FnMut`之间的主要区别：

1. **调用次数和所有权**: 
   - `FnOnce`: 只能被调用一次，因为它获取了闭包或函数对象的所有权并在调用后消耗自身。这意味着它不能再被再次调用或移交所有权。适用于需要获取所有权并只执行一次的情况。
   - `Fn`: 可以被多次调用，并且可以被多个地方借用。它不能获取所有权，因此在调用后可以再次使用。适用于不需要修改闭包或函数对象所捕获环境的情况。
   - `FnMut`: 可以被多次调用，并且可以修改闭包或函数对象所捕获的环境。它可以被多个地方借用，并且每次调用都可以对环境进行修改。适用于需要在调用过程中修改闭包或函数对象所捕获环境的情况。

2. **捕获环境的可变性**:
   - `FnOnce` 和 `Fn`: 它们不能修改捕获的环境，因为它们是不可变的引用。如果闭包或函数对象需要对捕获的环境进行修改，则需要使用 `FnMut`。
   - `FnMut`: 允许对捕获的环境进行修改，因为它是可变的引用。如果闭包或函数对象需要修改其捕获环境的状态，则应使用 `FnMut`。

3. **函数对象的特点**:
   - `FnOnce`：适用于一次性的、不可变的闭包或函数对象。它们可以拥有闭包的所有权，并在调用后自动销毁。
   - `Fn`：适用于可以多次调用的、不需要修改闭包或函数对象所捕获环境的闭包或函数对象。
   - `FnMut`：适用于可以多次调用且需要修改闭包或函数对象所捕获环境的闭包或函数对象。

通过选择正确的 trait，可以确保闭包或函数对象在需要的情况下具有适当的调用方式和环境访问权限。这样可以使代码更加清晰和可靠，并确保代码的行为符合预期。


## rust 俚语


### 为函数选择参数类型

当为函数选择参数类型时，使用带有强制隐式转换的目标类型会增加代码的复杂性。在这种情况下，函数将接受更多的输入参数类型。

使用可切片类型或胖指针类型没有限制。事实上，你应该总是使用借用类型（Borrowed type），而不是拥有数据类型的借用（Borrowing the owned type）。例如，使用`&str`而不是`&String`，`&[T]`而不是`&Vec<T>`，或者`&T`而不是`&Box<T>`。

当拥有数据结构（owned type）的实例已经提供了访问数据的间接层时，使用借用类型可以避免增加额外的间接层。举个例子，`String`类型有一层间接层，所以`&String`将有两个间接层。我们可以使用`&str`来避免这种情况，并且无论何时调用函数，将`&String`强制转换为`&str`。

例如，在下面的示例中，我们将说明使用`&String`和`&str`作为函数参数的区别。这种思路也适用于比较`&Vec<T>`和`&[T]`，`&T`和`&Box<T>`。

考虑一个判断单词是否包含连续三个元音字母的例子。我们不需要获取字符串的所有权，因此我们将使用引用。

代码如下：

```rust
fn three_vowels(word: &String) -> bool {
    let mut vowel_count = 0;
    for c in word.chars() {
        match c {
            'a' | 'e' | 'i' | 'o' | 'u' => {
                vowel_count += 1;
                if vowel_count >= 3 {
                    return true;
                }
            }
            _ => vowel_count = 0,
        }
    }
    false
}

fn main() {
    let ferris = "Ferris".to_string();
    let curious = "Curious".to_string();
    println!("{}: {}", ferris, three_vowels(&ferris));
    println!("{}: {}", curious, three_vowels(&curious));

    // 下面两行将失败：
    // println!("Ferris: {}", three_vowels("Ferris"));
    // println!("Curious: {}", three_vowels("Curious"));
}
```

在这个例子中，因为我们传递的参数是`&String`类型，所以代码可以正常运行。最后两行注释掉的代码将失败，因为`&str`类型不能隐式转换为`&String`类型。通过修改参数类型，我们可以轻松解决这个问题。

例如，将函数定义改为：

```rust
fn three_vowels(word: &str) -> bool {
```

那么两个版本都可以编译通过并打印相同的输出：

```
Ferris: false
Curious: true
```

### 用format!连接字符串

说明
对一个可变的String类型对象使用push或者push_str方法，或者用+操作符可以构建字符串。然而，使用format!常常会更方便，尤其是结合字面量和非字面量的时候。

例子

```rust
fn say_hello(name: &str) -> String {
    // 我们可以手动构建字符串
    // let mut result = "Hello ".to_owned();
    // result.push_str(name);
    // result.push('!');
    // result

    // 但是用format! 更好
    format!("Hello {}!", name)
}
```

- 优点 使用format! 连接字符串通常更加简洁和易于阅读。

- 缺点 它通常不是最有效的连接字符串的方法。对一个可变的String类型对象进行一连串的push操作通常是最有效率的（尤其这个字符串已经预先分配了足够的空间）


### 将集合视为智能指针
说明
使用集合的Deref特性使其像智能指针一样，提供数据的借用或者所有权。

例子

```rust

use std::ops::Deref;

struct Vec<T> {
    data: T,
    //..
}

impl<T> Deref for Vec<T> {
    type Target = [T];

    fn deref(&self) -> &[T] {
        //..
    }
}

```

一个Vec<T>是一些 T类型的所有权的集合，一个&[T]切片借用了一部分T。为Vec类型实现Deref特性使其可以隐式的 从 &Vec<T>转为&[T] ，并且也包括自动解引用的关系搜索。Vec类型大多数方法也对切片适用。

See also String and &str.

### 用mem::{take(_), replace(_)}在修改枚举变体时保持值的所有权

说明
假设我们有一个至少有两种变体的枚举&mut MyEnum，一种是A { name: String, x: u8 }， 另一种是B { name: String }。现在我们想要当x=0时，将A变为B，同时变量除变体类型变化外其他不变。

我们可以不用克隆name变体即可实现上述操作。

例子

```rust
use std::mem;

enum MyEnum {
    A { name: String, x: u8 },
    B { name: String }
}

fn a_to_b(e: &mut MyEnum) {

    // we mutably borrow `e` here. This precludes us from changing it directly
    // as in `*e = ...`, because the borrow checker won't allow it. Therefore
    // the assignment to `e` must be outside the `if let` clause.
    *e = if let MyEnum::A { ref mut name, x: 0 } = *e {

        // this takes out our `name` and put in an empty String instead
        // (note that empty strings don't allocate).
        // Then, construct the new enum variant (which will
        // be assigned to `*e`, because it is the result of the `if let` expression).
        MyEnum::B { name: mem::take(name) }

    // In all other cases, we return immediately, thus skipping the assignment
    } else { return }
}
这种方法对多种枚举变体也适用:



use std::mem;

enum MultiVariateEnum {
    A { name: String },
    B { name: String },
    C,
    D
}

fn swizzle(e: &mut MultiVariateEnum) {
    use MultiVariateEnum::*;
    *e = match *e {
        // Ownership rules do not allow taking `name` by value, but we cannot
        // take the value out of a mutable reference, unless we replace it:
        A { ref mut name } => B { name: mem::take(name) },
        B { ref mut name } => A { name: mem::take(name) },
        C => D,
        D => C
    }
}
```

出发点
当使用枚举的时候，我们可能想要改变枚举变体类型为其他类型。为了通过借用检查器检查，我们将分为两个阶段。在第一阶段，我们查看现有的值然后决定下一步怎么做。第二阶段我们可以修改值。

借用检查器不允许我们拿走name字段的值（因为那总得有有个东西放在那啊）。我们当然可以用.clone()克隆一个name的值，然后把这个克隆的值赋给MyEnum::B， 不过这样就是一个反模式的实例（为了满足借用检查器就用克隆，增大了开销）。综上，我们可以通过仅仅一个可变借用来改变值，避免多余的空间申请。

mem::take支持我们交换值，用默认值替换，并且返回原值。对于String类型，默认值是一个空字符串，无需申请空间。因此，我们获取原来的name(作为一个拥有值的变量)，我们可以把它包装成另一个枚举。

注：mem:replace非常相似，不过其允许我们指定要替换的值。可以用它实现mem::take的功能：mem::replace(name,String::new())。

然而，如果我们要使用Option的默认值替换掉枚举变体的值，那么用take()方法还是更习惯和简便的。

优点
看好啦，没有内存申请！同时你在这么做的时候会感觉自己像Indiana Jones。（译者注：没看过夺宝奇兵，没get到梗）

缺点
这会变得有点啰嗦。如果错误地重复这个操作将会让你厌恶借用检查器。编译器将无法对替换操作优化，结果是让你觉得相比其他不安全的语言来说性能更低。

此外，take操作需要类型实现Default特性。然而，如果这个类型没有实现Default特性，你还是可以用 mem::replace。


### 栈上动态分发

当需要动态分发多个值时，可以使用栈上的延迟条件初始化来实现。这种方法允许我们根据条件来声明和初始化不同类型的变量，并将它们绑定到一个共同的接口。

下面是一个示例：

```rust
use std::io;
use std::fs;

// Declare the variables with extended lifetimes.
let (mut stdin_read, mut file_read);

// We need to ascribe the type to get dynamic dispatch.
let readable: &mut dyn io::Read = if arg == "-" {
    stdin_read = io::stdin();
    &mut stdin_read
} else {
    file_read = fs::File::open(arg)?;
    &mut file_read
};

// Read from `readable` here.
```

这段代码中，我们首先声明了`stdin_read`和`file_read`两个变量，它们具有延长的生命周期，需要活得比`readable`更久。然后，我们使用延迟条件初始化的方式来根据条件选择不同的对象，并将它们的可变引用绑定到`readable`变量上。通过这种方式，我们可以实现动态分发，即根据条件选择不同的对象进行操作。

这种方法的优点是我们不需要在堆上分配任何空间，也不需要初始化不使用的对象，同时可以同时支持不同类型的对象（如`File`和`Stdin`）。

然而，这种方法相比使用`Box`实现的版本需要更多的代码来处理动态分发。使用`Box`实现动态分发时，代码会更简洁，但需要在堆上分配内存。

除了上述示例中的延迟条件初始化，Rust还提供了其他的动态分发方式，如使用`Box<dyn Trait>`来存储实现了某个Trait的对象。这种方式可以在运行时选择不同的实现，实现灵活的动态分发。

希望以上信息对您有所帮助！如果您有任何进一步的问题，请随时提问。


### 关于 Option的迭代器
说明
Option可以被视为一个包含一个0个或者1个元素的容器。特别是它实现了IntoIterator特性，这样我们就可以用来写泛型代码。

示例
因为Option实现了IntoIterator特性，它就可以用来当.extend()的参数:


```rust
let turing = Some("Turing");
let mut logicians = vec!["Curry", "Kleene", "Markov"];

logicians.extend(turing);

// equivalent to
if let Some(turing_inner) = turing {
    logicians.push(turing_inner);
}
如果你需要将一个Option添加到已有的迭代器后面，你可以用 .chain():



let turing = Some("Turing");
let logicians = vec!["Curry", "Kleene", "Markov"];

for logician in logicians.iter().chain(turing.iter()) {
    println!("{} is a logician", logician);
}

```

注意如果这个Option总是非空的，那么用std::iter::once更加合适。

此外，因为Option实现了IntoIterator特性，它就可以用for循环来迭代。这等价于用if let Some(..)，大多数情况下倾向于用后者。



