---
title: rust学习笔记
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["rust"]
tags: ["rust"]
---

# rust学习笔记
#rust
## mut
只有声明了mut，该变量才能被修改。如果要在引用中修改，前提也必须是该变量是mut类型才可以
```rust
let guess = String::new();
io::stdin().read_line(&mut guess).expect("read stdin err");
```
会报错，因为guess不是可变变量，无法在引用时给它引用
```
error[E0596]: cannot borrow `guess` as mutable, as it is not declared as mutable
```

必须定义为可变变量才行
```rust
    let mut guess = String::new();
    io::stdin().read_line(&mut guess).expect("read stdin err");
    println!("guess is {guess}")
```

## Cargo.lock
第一次构建项目时，Cargo 计算出所有符合要求的依赖版本并写入 Cargo.lock 文件。当将来构建项目时，Cargo 会发现 Cargo.lock 已存在并使用其中指定的版本，而不是再次计算所有的版本。这使得你拥有了一个自动化的可重现的构建。换句话说，项目会持续使用 0.8.3 直到你显式升级。

当你 **确实** 需要升级 crate 时，Cargo 提供了这样一个命令，update，它会忽略 *Cargo.lock* 文件，并计算出所有符合 *Cargo.toml* 声明的最新版本。Cargo 接下来会把这些版本写入 *Cargo.lock* 文件。不过，Cargo 默认只会寻找大于 0.8.3 而小于 0.9.0 的版本。

## 变量与可变性
### 变量
Rust 编译器保证，如果声明一个值不会变，它就真的不会变，所以你不必自己跟踪它。这意味着你的代码更易于推导。
### 常量
1. 不允许对常量使用 mut。常量不光默认不能变，它总是不能变。
2. 声明常量使用 const 关键字而不是 let，并且 *必须* 注明值的类型。
3. 常量只能被设置为常量表达式，而不可以是其他任何只能在运行时计算出的值。
### 隐藏
我们可以定义一个与之前变量同名的新变量。可以改变之前变量的类型。但是必须使用let关键字
### 数据类型
当多种类型均有可能时，例如使用 parse 将 String 转换为数字时，必须增加类型注解，像这样：
```rust
let guess: u32 = “42”.parse().expect(“Not a number!”);
```
如果不像上面这样添加类型注解 : u32，Rust 会显示如下错误，这说明编译器需要我们提供更多信息，来了解我们想要的类型：
```sh
$ cargo build
   Compiling no_type_annotations v0.1.0 (file:///projects/no_type_annotations)
error[E0282]: type annotations needed
 --> src/main.rs:2:9
  |
2 |     let guess = "42".parse().expect("Not a number!");
  |         ^^^^^ consider giving `guess` a type

For more information about this error, try `rustc --explain E0282`.
error: could not compile `no_type_annotations` due to previous error

```

### 表达式和语句
**语句**（*Statements*）是执行一些操作但不返回值的指令。**表达式**（*Expressions*）计算并产生一个值

- 语句
```rust
fn main() {
    let x = (let y = 6);
}
```
这样编译时会报错，因为let y = 6是一个语句，没有返回值。

- 表达式
```rust
fn main() {
    let y = {
        let x = 3;
        x + 1
    };

    println!("The value of y is: {y}");
}
```

这个表达式：
{
    let x = 3;
    x + 1
}
是一个代码块，它的值是 4。这个值作为 let 语句的一部分被绑定到 y 上。注意 x+1 这一行在结尾没有分号，与你见过的大部分代码行不同。表达式的结尾没有分号。如果在表达式的结尾加上分号，它就变成了语句，而语句不会返回值。在接下来探索具有返回值的函数和表达式时要谨记这一点。

### 具有返回值的函数
因为 if 是一个表达式，我们可以在 let 语句的右侧使用它
```rust
fn main() {
    let condition = true;
    let number = if condition { 5 } else { 6 };

    println!("The value of number is: {number}");
}
```

记住，代码块的值是其最后一个表达式的值，而数字本身就是一个表达式。在这个例子中，整个 if 表达式的值取决于哪个代码块被执行。这意味着 if 的每个分支的可能的返回值都必须是相同类型

### 从循环返回值
loop 的一个用例是重试可能会失败的操作，比如检查线程是否完成了任务。然而你可能会需要将操作的结果传递给其它的代码。如果将返回值加入你用来停止循环的 break 表达式，它会被停止的循环返回：
```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;

        if counter == 10 {
            break counter * 2;
        }
    };

    println!(“The result is {result}”);
}
```
在循环之前，我们声明了一个名为 counter 的变量并初始化为 0。接着声明了一个名为 result 来存放循环的返回值。在循环的每一次迭代中，我们将 counter 变量加 1，接着检查计数是否等于 10。当相等时，使用 break 关键字返回值 counter * 2。循环之后，我们通过分号结束赋值给 result 的语句。最后打印出 result 的值，也就是 20。

## 所有权
所有权规则
1. Rust 中的每一个值都有一个 **所有者**（*owner*）。
2. 值在任一时刻有且只有一个所有者。
3. 当所有者（变量）离开作用域，这个值将被丢弃。


### 移动
在Rust 中，多个变量可以采取不同的方式与同一数据进行交互。让我们看看示例 4-2 中一个使用整型的例子。
```rust
    let x = 5;
    let y = x;
```
**示例 4-2：将变量**x**的整数值赋给**y
我们大致可以猜到这在干什么：“将 5 绑定到 x；接着生成一个值 x 的拷贝并绑定到 y”。现在有了两个变量，x 和 y，都等于 5。这也正是事实上发生了的，因为整数是有已知固定大小的简单值，所以这两个 5 被放入了栈中。
现在看看这个 String 版本：
```rust
    let s1 = String::from(“hello”);
    let s2 = s1;
```
这看起来与上面的代码非常类似，所以我们可能会假设他们的运行方式也是类似的：也就是说，第二行可能会生成一个 s1 的拷贝并绑定到 s2 上。不过，事实上并不完全是这样。

当我们将 s1 赋值给 s2，String 的数据被复制了，这意味着我们从栈上拷贝了它的指针、长度和容量。我们并没有复制指针指向的堆上数据。


两个数据指针指向了同一位置。这就有了一个问题：当 s2 和 s1 离开作用域，他们都会尝试释放相同的内存。这是一个叫做 **二次释放**（*double free*）的错误，也是之前提到过的内存安全性 bug 之一。两次释放（相同）内存会导致内存污染，它可能会导致潜在的安全漏洞。
为了确保内存安全，在 let s2 = s1 之后，Rust 认为 s1 不再有效，因此 Rust 不需要在 s1 离开作用域后清理任何东西。看看在 s2 被创建之后尝试使用 s1 会发生什么；这段代码不能运行：
```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;

    println!("{}, world!", s1);
}
```

如果你在其他语言中听说过术语 **浅拷贝**（*shallow copy*）和 **深拷贝**（*deep copy*），那么拷贝指针、长度和容量而不拷贝数据可能听起来像浅拷贝。不过因为 Rust 同时使第一个变量无效了，这个操作被称为 **移动**（*move*），而不是浅拷贝。

> go语言遇到的如果一个map赋给了另一个变量，修改另一个变量会将map的值修改掉，在rust就不会出现了


### 克隆
如果我们 **确实** 需要深度复制 String 中堆上的数据，而不仅仅是栈上的数据，可以使用一个叫做 clone 的通用函数。

这是一个实际使用 clone 方法的例子：
```rust
    let s1 = String::from(“hello”);
    let s2 = s1.clone();

    println!(“s1 = {}, s2 = {}”, s1, s2);
```
这段代码能正常运行，这里堆上的数据 **确实** 被复制了。
当出现 clone 调用时，你知道一些特定的代码被执行而且这些代码可能相当消耗资源。

- 只在栈上的数据：拷贝
这里还有一个没有提到的小窍门。这些代码使用了整型并且是有效的，他们是示例 4-2 中的一部分：
```rust
    let x = 5;
    let y = x;
    println!(“x = {}, y = {}”, x, y);
```
但这段代码似乎与我们刚刚学到的内容相矛盾：没有调用 clone，不过 x 依然有效且没有被移动到 y 中。
原因是像整型这样的在编译时已知大小的类型被整个存储在栈上，所以拷贝其实际的值是快速的。这意味着没有理由在创建变量 y 后使 x 无效。换句话说，这里没有深浅拷贝的区别，所以这里调用 clone 并不会与通常的浅拷贝有什么不同，我们可以不用管它。

> Rust 有一个叫做 Copy trait 的特殊注解，可以用在类似整型这样的存储在栈上的类型上（ [第十章](https://kaisery.github.io/trpl-zh-cn/ch10-00-generics.html) 将会详细讲解 trait）。如果一个类型实现了 Copy trait，那么一个旧的变量在将其赋值给其他变量后仍然可用。


哪些类型实现了 Copy trait 呢？你可以查看给定类型的文档来确认，不过作为一个通用的规则，任何一组简单标量值的组合都可以实现 Copy，任何不需要分配内存或某种形式资源的类型都可以实现 Copy 。如下是一些 Copy 的类型：
* 所有整数类型，比如 u32。
* 布尔类型，bool，它的值是 true 和 false。
* 所有浮点数类型，比如 f64。
* 字符类型，char。
* 元组，当且仅当其包含的类型也都实现 Copy 的时候。比如，(i32, i32) 实现了 Copy，但 (i32, String) 就没有。

### 所有权与函数
```rust
fn main() {
    let s = String::from("hello");  // s 进入作用域

    takes_ownership(s);             // s 的值移动到函数里 ...
                                    // ... 所以到这里不再有效

    let x = 5;                      // x 进入作用域

    makes_copy(x);                  // x 应该移动函数里，
                                    // 但 i32 是 Copy 的，
                                    // 所以在后面可继续使用 x

} // 这里, x 先移出了作用域，然后是 s。但因为 s 的值已被移走，
  // 没有特殊之处

fn takes_ownership(some_string: String) { // some_string 进入作用域
    println!("{}", some_string);
} // 这里，some_string 移出作用域并调用 `drop` 方法。
  // 占用的内存被释放

fn makes_copy(some_integer: i32) { // some_integer 进入作用域
    println!("{}", some_integer);
} // 这里，some_integer 移出作用域。没有特殊之处

```

### 引用与借用
**引用**（*reference*）像一个指针，因为它是一个地址，我们可以由此访问储存于该地址的属于其他变量的数据。 与指针不同，引用确保指向某个特定类型的有效值。

**引用**（*reference*）像一个指针，因为它是一个地址，我们可以由此访问储存于该地址的属于其他变量的数据。 与指针不同，引用确保指向某个特定类型的有效值。
下面是如何定义并使用一个（新的）calculate_length 函数，它以一个对象的引用作为参数而不是获取值的所有权：
文件名: src/main.rs
```rust
fn main() {
    let s1 = String::from(“hello”);

    let len = calculate_length(&s1);

    println!(“The length of ‘{}’ is {}.”, s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```
首先，注意变量声明和函数返回值中的所有元组代码都消失了。其次，注意我们传递 &s1 给 calculate_length，同时在函数定义中，我们获取 &String 而不是 String。这些 & 符号就是 **引用**，它们允许你使用值但不获取其所有权。

```rust
fn calculate_length(s: &String) -> usize { // s is a reference to a String
    s.len()
} // 这里，s 离开了作用域。但因为它并不拥有引用值的所有权，
  // 所以什么也不会发生
```

正如变量默认是不可变的，引用也一样。（默认）不允许修改引用的值。

  [可变引用](https://kaisery.github.io/trpl-zh-cn/ch04-02-references-and-borrowing.html#%E5%8F%AF%E5%8F%98%E5%BC%95%E7%94%A8) 
我们通过一个小调整就能修复示例 4-6 代码中的错误，允许我们修改一个借用的值，这就是 **可变引用**（*mutable reference*）：

> 可变引用的前提是，其引用的值必须是可变类型
文件名: src/main.rs
```rust
fn main() {
    let mut s = String::from(“hello”);

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(“, world”);
}
```
首先，我们必须将 s 改为 mut。然后在调用 change 函数的地方创建一个可变引用 &mut s，并更新函数签名以接受一个可变引用 some_string: &mut String。这就非常清楚地表明，change 函数将改变它所借用的值。
可变引用有一个很大的限制：如果你有一个对该变量的可变引用，你就不能再创建对该变量的引用。这些尝试创建两个 s 的可变引用的代码会失败：

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s;
    let r2 = &mut s;

    println!("{}, {}", r1, r2);
}

```

这个报错说这段代码是无效的，因为我们不能在同一时间多次将 s 作为可变变量借用。第一个可变的借入在 r1 中，并且必须持续到在 println！ 中使用它，但是在那个可变引用的创建和它的使用之间，我们又尝试在 r2 中创建另一个可变引用，该引用借用与 r1 相同的数据。
这一限制以一种非常小心谨慎的方式允许可变性，防止同一时间对同一数据存在多个可变引用。新 Rustacean 们经常难以适应这一点，因为大部分语言中变量任何时候都是可变的。这个限制的好处是 Rust 可以在编译时就避免数据竞争。**数据竞争**（*data race*）类似于竞态条件，它可由这三个行为造成：
* 两个或更多指针同时访问同一数据。
* 至少有一个指针被用来写入数据。
* 没有同步数据访问的机制。
数据竞争会导致未定义行为，难以在运行时追踪，并且难以诊断和修复；Rust 避免了这种情况的发生，因为它甚至不会编译存在数据竞争的代码！

## crate和module
crate是rust的基本构建单位，cargo new一个项目时可以通过参数指定是一个library crate还是一个binary crate；

lib类型
```rust
cargo new --lib restaurant
```
这种类型会生成：src/lib.rs 文件

binary类型
```rust
cargo new --bin restaurant
```
这种类型会生成：src/main.rs文件

### 模块与文件分离
```rust
pub mod front_of_house {
    pub mod hosting {
        pub fn add_to_wait_list() {}
        pub fn seat_at_table() {}
    }
    pub mod serving {
        fn take_order() {}
        fn serve_order() {}
        fn take_payment() {}
    }
}

mod back_of_house;

pub fn eat_at_restaurant() {
    front_of_house::hosting::add_to_wait_list();
    crate::front_of_house::hosting::add_to_wait_list();
    back_of_house::hosting::cooking();
}
```

`mod back_of_house;`告诉rust从另一个和back_of_house同名的文件中加载该模块的内容，即同级目录下的back_of_house.rs；

也可以是一个新的目录，里面包含mod.rs，如：
```
.
├── back_of_house
│   └── mod.rs
└── lib.rs
```

所以，hosting::cooking的内容需要去mod.rs中寻找
back_of_house/mod.rs
```rust
pub mod hosting {
    pub fn cooking() {}
}
```

如何引入其他文件中的函数
[Rust:  mod文件、main文件调用_songroom的博客-CSDN博客](https://blog.csdn.net/wowotuo/article/details/86442150)
https://tonydeng.github.io/2019/10/28/rust-mod/

## 博客精选
[飞书Rust实习面试 | KuangjuX(狂且)](https://blog.kuangjux.top/2021/10/22/%E9%A3%9E%E4%B9%A6Rust%E5%AE%9E%E4%B9%A0%E9%9D%A2%E8%AF%95/)