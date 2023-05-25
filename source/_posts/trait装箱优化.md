---
title: Box<dyn trait> 优化设计
date: 2023-05-25T11:31:09+08:00
tags:
- rust技巧
---

## box<dyn trait> 由来

`Box<dyn Trait>` 是 Rust 中的一种动态 trait 对象的表示方式。它允许将不同的类型封装在相同的类型擦除容器中，以实现多态性和运行时的动态分发。

使用 `Box<dyn Trait>` 的主要目的是在编译时不确定具体类型，而是在运行时根据需要确定类型。这对于处理具有不同类型但实现相同 trait 的对象集合非常有用。通过将这些对象包装在 `Box<dyn Trait>` 中，可以在运行时以一致的方式处理它们，无需关心具体的类型。

这种方式的好处是提供了更大的灵活性和可扩展性。您可以根据需要在运行时插入不同的对象，并对它们进行统一的处理。这对于构建插件化系统、实现接口的回调机制以及处理动态加载的模块等情况非常有用。

然而，使用 `Box<dyn Trait>` 也带来了一些性能开销。由于类型擦除和动态分发的特性，它需要在运行时进行虚函数调用，并可能导致额外的堆分配。因此，在性能敏感的场景中，静态分发通常更高效。

总而言之，`Box<dyn Trait>` 提供了一种动态多态的方式，使得在运行时能够处理不同类型的对象。它在需要灵活性和动态性的情况下非常有用，但需要权衡性能开销和设计需求。


当我们需要处理一组不同类型的对象，但这些对象都实现了相同的 trait 时，可以使用 `Box<dyn Trait>` 来统一处理它们。下面是一个简单的示例：

```rust
trait Sound {
    fn make_sound(&self);
}

struct Dog;
impl Sound for Dog {
    fn make_sound(&self) {
        println!("Woof!");
    }
}

struct Cat;
impl Sound for Cat {
    fn make_sound(&self) {
        println!("Meow!");
    }
}

fn main() {
    let animals: Vec<Box<dyn Sound>> = vec![
        Box::new(Dog),
        Box::new(Cat),
    ];

    for animal in animals {
        animal.make_sound();
    }
}
```

在上面的示例中，我们定义了一个 `Sound` trait，并为 `Dog` 和 `Cat` 实现了该 trait。然后，我们创建了一个 `Vec<Box<dyn Sound>>`，并将 `Dog` 和 `Cat` 对象的 `Box` 放入其中。最后，我们使用 `for` 循环遍历 `Vec` 中的对象，并调用它们的 `make_sound` 方法。

通过使用 `Box<dyn Sound>`，我们可以将不同类型的对象放在同一个容器中，并对它们进行统一的处理。这样，我们可以轻松地添加更多类型的对象，而无需修改遍历和处理的逻辑。这种动态多态的特性为我们提供了灵活性和可扩展性。

## Box<dyn Trait> 问题

trait 存在的一个意义就是： 隔离， 可是当我们需要原来具体的类型呢？ 在这种情况下 我们只能使用类型断言 downcast 还原类型

在上面的例子中，由于我们将不同类型的对象存储在 `Vec<Box<dyn Sound>>` 中，它们都被擦除为 `dyn Sound` 类型。如果我们需要在特定场景下恢复对象的具体类型，可以使用 `downcast` 方法进行类型转换。

下面是使用 `downcast` 还原类型的示例：

```rust
use std::any::Any;

trait Sound {
    fn make_sound(&self);
    fn as_any(&self) -> &dyn Any;
}

struct Dog;
impl Sound for Dog {
    fn make_sound(&self) {
        println!("Woof!");
    }

    fn as_any(&self) -> &dyn Any {
        self
    }
}

struct Cat;
impl Sound for Cat {
    fn make_sound(&self) {
        println!("Meow!");
    }
    fn as_any(&self) -> &dyn Any {
        self
    }
}

fn main() {
    let animals: Vec<Box<dyn Sound>> = vec![
        Box::new(Dog),
        Box::new(Cat),
    ];

    for animal in animals {
        animal.make_sound();

        if let Some(dog) = animal.as_any().downcast_ref::<Dog>() {
            println!("It's a dog!");
            // 执行特定于 Dog 类型的操作
        } else if let Some(cat) = animal.as_any().downcast_ref::<Cat>() {
            println!("It's a cat!");
            // 执行特定于 Cat 类型的操作
        }
    }
}

```

在上面的示例中，我们在 `for` 循环中使用了 `downcast_ref` 方法来尝试将 `dyn Sound` 类型转换回具体类型（`Dog` 或 `Cat`）。如果转换成功，就可以在特定的分支中执行与类型相关的操作。

请注意，`downcast_ref` 方法返回一个 `Option` 类型，因为转换可能会失败。如果对象的实际类型与尝试转换的类型不匹配，将返回 `None`。

通过使用 `downcast`，我们可以根据实际需要在特定情况下还原对象的具体类型，以执行针对该类型的特定操作。这为处理动态多态对象提供了一种灵活而安全的方式。

## 结构体泛型 “wrapper + 结构体泛型”

既能达到“多态”效果，又确保了原类型不会丢失

```rust
trait Sound {
    fn make_sound(&self);
}

struct Dog;
impl Sound for Dog {
    fn make_sound(&self) {
        println!("Woof!");
    }

}

struct Cat;
impl Sound for Cat {
    fn make_sound(&self) {
        println!("Meow!");
    }
}

struct SoundWrapper<B: Sound> {
    sound: B
}

impl<B: Sound> Sound for SoundWrapper<B> {
    fn make_sound(&self) {
        self.sound.make_sound()
    }
}

```

可以使用 SoundWrapper 来完成内部的状态的访问， 但是不可以直接使用 Vec 来完成批量存储

```rust
impl<B: Sound> Sound for SoundWrapper<B> {
    fn make_sound(&self) {
        self.sound.make_sound()
    }
}
```
来完成简化 soudWrapper 调用make_sound 的操作

## 特征集合

我注意到的一个常见模式是为特征对象创建一个枚举包装器，它可以让您完全控制传递给结构的数据。

我的意思是，创建一个枚举，实现与其每个值实现的相同特征。

为了演示，我将首先创建另一个实现Person.
```rust
struct Grandma {
    age: usize
}

impl Person for Grandma {
    fn say_hello(&self) {
        println!("G'day!")
    }
}

```
由于我现在有多个Person，因此创建一个包装器枚举来跟踪它们并允许通过 Rust 的模式匹配语法轻松访问是有意义的。

此外，由于 Rust 枚举被标记为 unions，它们将只有枚举中最大条目的内存占用（加上更多的类型信息），并且由于大小在编译时已知，因此不会有堆分配就像一个Box.

```rust
enum People {
    Grandma(Grandma),
    Me(Me)
}

impl Person for People {
    fn say_hello(&self) {
        match self {
            People::Grandma(grandma) => grandma.say_hello(),
            People::Me(me) => me.say_hello()
        }
    }
}

```

通过一些小的修改，它可以像as_any示例一样使用。

```rust
let mut zoo: PeopleZoo<People> = PeopleZoo {
  people: vec![]
};
zoo.add_person(People::Me(Me { name: "Bennett" }));

if let Some(People::Me(me)) = zoo.last_person() {
    println!("My name is {}.", me.name)
}
```
这为消费最大灵活性的人提供了PeopleZoo无需修改Person特征的人。它还具有不需要在堆上为Person 分配空间.