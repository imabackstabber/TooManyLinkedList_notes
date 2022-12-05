## Learning Rust with Entirely Too Many Linked Lists-Ch3

### 3. An Ok Stack

#### 3.0 Layout

本节中用到的数据结构设计如下

```rust
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}
```

在之后的小节中我们会频繁回过头来查看数据结构的设计（这样我们才能写出代码嘛）



#### 3.1 Option

介绍了`option`中`take`方法的使用，which is analogous to mem::replace

介绍了option::map方法的使用

> Maps an `Option<T>` to `Option<U>` by applying a function to a contained value.
>
> **consuming the original**（所有权转移）

这样的话我们可以rewrite它的pop方法：

```rust
pub fn pop(&mut self) -> Option<i32> {
  match self.head.take() {
    None => None,
    Some(node) => {
      self.head = node.next;
      Some(node.elem)
    }
  }
}

// 改写之后
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```



#### 3.2 Generic

由单一类型到泛型的转换是非常无痛的（对于本数据结构而言），没有什么额外要说的



#### 3.3 Peek

Peek作为一个只应返回shared ref（即仅仅满足观察途径，当然，你也可以选择实现一个能获取mut元素的peek_mut方法）的方法，要求我们的函数返回类型是`Option<&T>`

这里作者引入了`option::as_ref`方法，签名如下:

```rust
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

> 这个函数的作用非常重要，后面的教程中我们会经常见到它，它帮助我们把`&option<T>`转变为`option<&T>`。 

这样就能很简单的返回`Option<&T>`（也可以是mut的，使用`as_mut`方法）

```rust
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

除此之外，还介绍了闭包函数的写法。

> 这里的逻辑非常离谱，参数的声明方法不再和平常一样而是遵循一种类似于**模式匹配**规则的手法。由于规则实在过于抽象，建议看代码增强直观的理解。

首先是正常人版本（也就是惯例上，我们认为闭包参数应该这样写）

```rust
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

这样将会导致报错：

```bash
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

这是因为rust处理的方法其实是：

> It turns out that writing the argument of the closure that way **doesn't specify** that `value` is a mutable reference. 
>
> Instead, it creates a **pattern that will be matched** against the argument to the closure; `|&mut value|` means "the argument is a mutable reference, but just copy the value it points to into `value`, please." 

所以应该这样写：

```rust
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

行吧



#### 3.4 IntoIter

在接下来的三小节里面我们将会观察迭代器的不同形式：

- IntoIter - `T`
- IterMut - `&mut T`
- Iter - `&T`

最简单的实现方法是`IntoIter`，因为它转交所有权，而依托于之前对LinkedList实现的pop方法就能很方便地实现此功能。

```rust
// Tuple structs are an alternative form of struct,
// useful for trivial wrappers around other types.
pub struct IntoIter<T>(List<T>); 

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self) // 这说明了into_iter()方法将会take away ownership
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // access fields of a tuple struct numerically
        self.0.pop()
    }
}
```



#### 3.5 Iter

这一节的知识点是三小节里面最充实的

因为我们不再是转交所有权，所以我们需要自己去fiddle指针，同时reserve value。

所以在一开始的layout上我们就不能采取和`IntoIter`一样的写法，而需要更复杂的结构保存指针。

##### 3.5.1 Lifetime

```rust
// Layout
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

// Method
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| { // 注意到这里map其实带走了ownership，这样写却不报错
          										// 这是因为&（ref类型）实际上实现了Copy trait
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

上述代码将会使得编译器抱怨如下：

```bash
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:72:18
   |
72 |     next: Option<&Node<T>>,
   |                  ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:82:17
   |
82 |     type Item = &T;
   |                 ^ expected lifetime parameter
```

由于这里试图返回一个&类型，就牵扯到了Lifetime问题。

作者在这里尝试使用`rustc --explain E0106`查看Lifetime问题相关的解决办法，最后决定全部加上`'a`试一下（我本人

于是代码被改写成了这样(就是原版代码里面哪里有&就加上`'a`）：

```rust
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> List<T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &'a node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&'a mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &'a node);
            &'a node.elem
        })
    }
}
```

然而！肯定不会这么简单，编译器抱怨如下：

```bash
> cargo build

error: expected `:`, found `node`
  --> src/second.rs:77:47
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ---- while parsing this struct        ^^^^ expected `:`

error: expected `:`, found `node`
  --> src/second.rs:85:50
   |
85 |             self.next = node.next.map(|node| &'a node);
   |                                                  ^^^^ expected `:`

error[E0063]: missing field `next` in initializer of `second::Iter<'_, _>`
  --> src/second.rs:77:9
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ^^^^ missing `next`
```

> Oh god we broke rust

于是作者又娓娓道来了什么是生命周期以及我们为什么需要它。

下面是我自己理解的行文大意：

> 生命周期这个概念在拥有gc的语言中并不必须，因为gc会帮我们清理掉这些东西（自动地）。然而在Rust这门语言中大多数数据都是手动管理的，所以我们需要某种途径来（正确地）管理他们。C-like语言向我们展示了拥有一个野指针可能的后果：
>
> 1. 野指针指向数据占有的内存的外部
> 2. 野指针指向了已经被改写的数据（读/写脏数据会导致糟糕的后果）
>
> 而生命周期就能在99%的时候解决这些问题。这就是**生命周期**存在的意义。
>
> 那么什么是生命周期呢？
>
> 很直白地来说，生命周期就是程序代码中的一个区块，在这个区块中那些&类型数据（reference）被认为是有效的。不同的代码区域会有不同的生命周期限制。rust程序中整个生命周期系统就像是一个**约束条件求解问题**一样，如果你能找到一种满足各项约束的解（每个变量/抑或是所谓的reference）的生命周期能够有一组解使得程序代码合理，那么rust程序就能成功编译。
>
> 很好的一个消息是在函数体内部我们并不需要这个概念，因为我们能清楚地明白这些函数体内局部变量的作用域。但是对于API-level（我想就是函数传出去的参数的生命周期）的信息而言，编译器并不知道，所以我们在传出一些reference的时候，就需要带上生命周期，不然编译器并不能确定这是否会是一个野指针。

我们平时写函数也并没有关心一些生命周期但能成功编译，这是因为`lifetime elision`的语法糖导致。

```rust
// Only one reference in input, so the output must be derived from that input
fn foo(&A) -> &B; // sugar for:
fn foo<'a>(&'a A) -> &'a B;

// Many inputs, assume they're all independent
fn foo(&A, &B, &C); // sugar for:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// Methods, assume all output lifetimes are derived from `self`
fn foo(&self, &B, &C) -> &D; // sugar for:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

注意到这里传入和传出的参数类型实际上都是`&`类型的。

对于`fn foo<'a>(&'a A) -> &'a B`这一句话来说，其实就是在说:

> A生存的时间务必要比传出去的&B更长

这样就预防了野指针的出现。

于是作者给出了自己的改版：

```rust
// Iter is generic over *some* lifetime, it doesn't care
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// No lifetime here, List doesn't have any associated lifetimes
impl<T> List<T> {
    // We declare a fresh lifetime here for the *exact* borrow that
    // creates the iter. Now &self needs to be valid as long as the
    // Iter is around.
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
      	// 人话：self存在的时间必须要长过Iter<'a, T>
        Iter { next: self.head.map(|node| &node) }
    }
}

// We *do* have a lifetime here, because Iter has one that we need to define
impl<'a, T> Iterator for Iter<'a, T> {
    // Need it here too, this is a type declaration
    type Item = &'a T;

    // None of this needs to change, handled by the above.
    // Self continues to be incredibly hype and amazing
    fn next(&mut self) -> Option<Self::Item> {
      	// 这里没有加'a，应该是因为impl里面自动带了'a的声明了
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

OK，让我们看看编译器这次怎么发电：

```bash
cargo build

error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

好吧，是因为Box的原因，看来需要多deref一下子：

```rust
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &*node) } // * 用来解引用
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &*node); // * 用来解引用
            &node.elem
        })
    }
}
```

看起来没问题了，但是根据剧本，编译器现在必须要继续发电：

```bash
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:77:43
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                                           ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                      ^^^^^^^^^ cannot move out of borrowed content

error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:85:46
   |
85 |             self.next = node.next.map(|node| &*node);
   |                                              ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &*node);
   |                         ^^^^^^^^^ cannot move out of borrowed content
```

这里报错的原因是我们没有使用`as_ref`（在3.3中提及），所以这里`option::map`会带走所有权，然后我们传的又是引用，所以传出去的引用又彻底变成了野指针😱

所以加上`as_ref`（在报错的地方），用来抚慰`map`

```rust
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { 
          next: self.head.as_ref().map(|node| &*node)  // &option<T> -> option<&T>
      }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &*node); // &option<T> -> option<&T>
            &node.elem
        })
    }
}
```

这是本小节编译器最后一次发电：

```bash
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.as_ref().map(|node| &*node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.as_ref().map(|node| &*node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

看起来这里又有一层Box，再deref一次就好了：

```rust
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.elem
        })
    }
}
```

可以看出这里`as_deref()`实际上取代了这里的`map(|node| &**node)`，`option::as_deref()`的签名是：

> Converts from `Option<T>` (or `&Option<T>`) to `Option<&T::Target>`

实际上这里只是做了隐式的deref（插入了一长串*）,以及顺个大便把`&Option<T>`变成了`Option<&T>`

最终的这个版本就能通过编译，所以no bull shit again！

另外还可以通过语法糖有效地减少生命周期出现的次数👇

```rust
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_deref() }
    }
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_deref() }
    }
}
```



#### 3.6 IterMut

这一节其实没什么新的东西，把上一节的代码全部改成mut就可以了，但事实上是过不了的，因为前面的`next`方法利用了Copy trait，事实上应该显式地调用`Option::take`方法来抚慰`map`，作者在这一节也花了很大的力气去说明什么是Copy trait。

以下是一些take away message：

1. 所有numeric primitives（i32，u64，bool，f32，etc）都是实现了Copy的。Copy trait可以派生，所以`Option<&>`也是有Copy trait的（&是i32，而且是shared）
2. 在rust中，&mut自然不能是有Copy trait的，这样会导致有多个写者，这也自然不符合rust对于&mut的定义

所以最后的代码应该是这个样子：

```rust
// Layout
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

// Methods
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
    	IterMut { next: self.head.as_deref_mut() }
		}
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.take().map(|node| {
            self.next = node.next.as_deref_mut();
            &mut node.elem
        })
    }
}

```

