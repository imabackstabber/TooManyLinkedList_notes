## Learning Rust with Entirely Too Many Linked Lists-Ch4

### 4. A Persistent Stack

这一节中，我们会了解到Rc（ref counting），基于此我们可以实现单线程下的共享数据结构

#### 4.1 Layout

作者企图通过直接改掉Box Version的链表定义来定义Rc Version的链表

```rust
// Original
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

// Rc version
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Rc<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

事实上这样确实可行，我们接下来就会和Layout频繁打交道，用来构建我们的代码。



#### 4.2 Basic

因为在这里的定义中push和pop不再具有什么意义了（想想Rc的情况），所以改成prepend和tail

所幸这两个方法都很简单，先这么写：

```rust
pub fn prepend(&self, elem: T) -> List<T> {
    List { head: Some(Rc::new(Node {
        elem: elem,
        next: self.head.clone(),
    }))}
}

pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().map(|node| node.next.clone()) }
}
```

随后编译器yells that这个tail方法有问题：

```rust
cargo build

error[E0308]: mismatched types
  --> src/third.rs:27:22
   |
27 |         List { head: self.head.as_ref().map(|node| node.next.clone()) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `std::rc::Rc`, found enum `std::option::Option`
   |
   = note: expected type `std::option::Option<std::rc::Rc<_>>`
              found type `std::option::Option<std::option::Option<std::rc::Rc<_>>>`
```

我们再看一眼map的函数签名

```rust
pub fn map<U, F>(self, f: F) -> Option<U>
where
    F: FnOnce(T) -> U
```

原来是要求返回U类型而不是`Option<U>`

然而作者表示Option存在一个`and_then`方法是支持我们送进去一个返回`Option`的回调函数的：

```rust
pub fn and_then<U, F>(self, f: F) -> Option<U>
where
    F: FnOnce(T) -> Option<U>
```

随后只需要把代码改成`and_then`就完事！

另外作者还加了一个`head`函数，用于peek出head含有的元素：

```rust
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
}
```

其实就是`tail`的翻版，其实还比`tail`更简单了。



#### 4.3 Drop

这里比较吊诡的是因为Rc是ref count的，所以没办法直接粗暴的释放了:(

让我们看看原来怎么实现Drop的：

```rust
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

问题就出在：

```rust
cur_link = boxed_node.next.take();
```

因为我们不能就在原地留下一个None（我们现在是共享的诶）

但如果是最后一个Ref，我们就可以这样做，这需要我们查看ref count

Rc的try_wrap可以帮我们实现这个功能

先来看看这个函数的签名:

```rust
pub fn try_unwrap<T>(rc: Rc<T>) -> Result<T, Rc<T>>
```

下面是一个使用实例：

```rust
use rc::{self, Rc};

let x = Rc::new(3);
assert_eq!(rc::try_unwrap(x), Ok(3));

let x = Rc::new(4);
let _y = x.clone();
assert_eq!(rc::try_unwrap(x), Err(Rc::new(4)));
```

于是在`Drop`中，我们尝试用OK去匹配它：

```rust
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take(); // 可以放心take，留下None
            } else {
                break;
            }
        }
    }
}
```

ok，这次编译器没有抱怨。



#### 4.4 Arc

总而言之，Arc是考虑到线程安全情况的Rc，但我们目前为止还远够不到多线程Rust的水平。

作者自己心里也很清楚，所以就全篇口胡了（

下面是我自己领悟的全文大意：

> 对于线程安全来说，**可变性**是魔鬼（考虑写者问题）。既然如此的话，我们直接把可变性杀掉，换成不可变的东西不就线程安全了（真是大聪明）。
>
> 然而对于我们基于Rc实现的list来说却不是这样的，因为多个线程试图同时inc ref count的情况是存在的（导致ref count错误），所以我们基于Rc的list，即便是天生具有不可变性，也不是线程安全的。
>
> 所以我们需要Arc。Arc其实就是Rc，但是支持了**原子地修改**引用计数。
>
> 对于我们构建的list来说（注意这里的限定条件），我们只需要把Rc改成Arc，我们实现的list就是线程安全的了。
>
> 下面是一个重要的问题：我们如何知道一个类型是否线程安全？我们可能搞砸吗？
>
> 不能，因为rust中的**线程安全类型是第一公民类型**。`Send`和`Sync`这两个类型就是用于描述线程安全与否的。
>
> `Send`是指线程中能够安全移动所有权的，`Sync`是指线程中能够安全`share`的。
>
> 如果`T`是`Sync`的，那么`&T`是`Send`的（确实没问题）
>
> 这里“安全”是说不会造成数据冒险（但不是说其他冒险也能避免）
>
> `Send`和`Sync`实际上是marker trait，不包含任何实现，只是作为指示出现。
>
> 这样API在实现的时候就能要求你是`Send`或者`Sync`的，在静态编译时就能检测出线程中的操作是否是安全的。
>
> 另外，他们俩都是可以可以自动派生的trait，考虑下copy trait就能理解这个道理。
>
> 几乎所有的类型都是`Send`and`Sync`的。他们成为`Send`的原因是因为他们完全拥有了自己数据的所有权(totoally own their data)（说实话这一句话不理解）。他们成为`Sync`的原因是因为想要在多线程条件下访问他们的唯一方法是通过shared reference，这将会使得数据本身**不可变**（immutable）
>
> 但仍然有一些东西不满足上述条件——那些拥有**内部可变性**（internal mutability）的数据。
>
> 目前为止我们接触的都是**继承可变性**（aka外部可变性），也就是说数据的可变性取决于它的container。比如我们不能改变不可变结构体中的数据域（field）。
>
> 但对于那些拥有内部可变性的数据而言并不是这样的，因为尽管外部的container没变，你仍然可以修改里面的数据。
>
> 比如cell类型的数据就具有内部可变性，它们只能在单线程下工作。对于多线程，你可能需要lock类型。也有atomics，这是一组原语，工作方式和lock差求不多。
>
> 对于Arc来说，你没法指望把随便什么数据类型丢进去然后就让它变成线程安全的（真是个大聪明）。你需要确保内部数据也是线程安全的。

#### 