## Learning Rust with Entirely Too Many Linked Lists-Ch6

### 6. An Ok Unsafe Singly-Linked Queue

#### 6.1 Layout

首先，为了确保`push`和`pop`的时间复杂度是$O(1)$，需要额外设置头指针和尾指针。这就导致了Queue的设计和Stack的不同之处。

作者首先给出了一版不能用的代码，注意到下面的代码违反了**所有权规则**：

```rust
use std::mem;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // swap the old tail to point to the new tail
        let old_tail = mem::replace(&mut self.tail, Some(new_tail));

        match old_tail {
            Some(mut old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
            }
        }
    }
}
```

> 作者为了不劝退我们还特别说明了这里的代码也不是一蹴而就的，经历了几次调试，她真的我哭死

会报错如下：

```bash
> cargo build

error[E0382]: use of moved value: `new_tail`
  --> src/fifth.rs:38:38
   |
26 |         let new_tail = Box::new(Node {
   |             -------- move occurs because `new_tail` has type `std::boxed::Box<fifth::Node<T>>`, which does not implement the `Copy` trait
...
33 |         let old_tail = mem::replace(&mut self.tail, Some(new_tail));
   |                                                          -------- value moved here
...
38 |                 old_tail.next = Some(new_tail);
   |                                      ^^^^^^^^ value used here after move
```

编译器向我们指出这里存在所有权问题。因为Box自己没有Copy trait，所以我们不能将一个Box赋给两个值。这是两个写指针，想一想就很吓人。而且上述代码还会导致Box被drop的话同一块内存区域被double-free。

总而言之，我们不能用所有权这个东西了。那就用reference：

```rust
pub struct List<T> {
    head: Link<T>,
    tail: Option<&mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

但是你知道不会这么简单的吧（一般都是等编译器报错了后知后觉）：

```bash
> cargo build

error[E0106]: missing lifetime specifier
 --> src/fifth.rs:3:18
  |
3 |     tail: Option<&mut Node<T>>, // NEW!
  |                  ^ expected lifetime parameter
```

那么加上`'a`再试一下，不一定一下改对，但是就这么一步一步改：

```rust
pub struct List<'a, T> {
    head: Link<T>,
    tail: Option<&'a mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<'a, T> List<'a, T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_deref_mut()
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_deref_mut()
            }
        };

        self.tail = new_tail;
    }
}
```

但是编译器觉得不行：

```bash
cargo build

error[E0495]: cannot infer an appropriate lifetime for autoref due to conflicting requirements
  --> src/fifth.rs:35:27
   |
35 |                 self.head.as_deref_mut()
   |                           ^^^^^^^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the method body at 18:5...
  --> src/fifth.rs:18:5
   |
18 | /     pub fn push(&mut self, elem: T) {
19 | |         let new_tail = Box::new(Node {
20 | |             elem: elem,
21 | |             // When you push onto the tail, your next is always None
...  |
39 | |         self.tail = new_tail;
40 | |     }
   | |_____^
note: ...so that reference does not outlive borrowed content
  --> src/fifth.rs:35:17
   |
35 |                 self.head.as_deref_mut()
   |                 ^^^^^^^^^
note: but, the lifetime must be valid for the lifetime 'a as defined on the impl at 13:6...
  --> src/fifth.rs:13:6
   |
13 | impl<'a, T> List<'a, T> {
   |      ^^
   = note: ...so that the expression is assignable:
           expected std::option::Option<&'a mut fifth::Node<T>>
              found std::option::Option<&mut fifth::Node<T>>
```

有用的话就这里一句：

```bash
the lifetime must be valid for the lifetime 'a as defined on the impl
```

作者的分析是，我们是在从`self`借用，但我们完全没声明这里`self`的生命周期，而我们需要确保`self.head`的生命周期至少和`'a`一样长（这是因为我们把`self.head.as_deref_mut()`作为了`new_tail`，而`new_tail`将会变成`self.tail`，`self.tail`要求生命周期为`'a`），所以需要声明。

改成如下声明就可以抚慰编译器：

```rust
pub fn push(&'a mut self, elem: T) 
```

顺便加上`pop`的实现：

```rust
pub fn pop(&'a mut self) -> Option<T> {
    // Grab the list's current head
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        // If we're out of `head`, make sure to set the tail to `None`.
        if self.head.is_none() {
            self.tail = None;
        }

        head.elem
    })
}
```

再加上`test`代码（没错这里我特意添加上了，因为问题就出在这里）：

```rust
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);
    }
}
```

我们将会见证一大长串的错误:

```rust
cargo test

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:68:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
68 |         list.push(1);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:69:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
69 |         list.push(2);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:70:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
70 |         list.push(3);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here


....

** WAY MORE LINES OF ERRORS **

....

error: aborting due to 11 previous errors
```

这是为何呢？

> 我们犯下了一个Rust中的大罪：我们在自己内部存储了对自己的引用。
>
> （We just committed a cardinal Rust sin: we stored a reference to ourselves *inside ourselves*.）

即便我们写的`push`和`pop`并没有任何问题（从语法的角度来讲），但是我们写的东西并**不符合逻辑**。

> But as soon as we try to actually *use* the list, the compiler quickly goes "yep you've borrowed `self` mutably for `'a`, so you can't use `self` anymore until the end of `'a`" but *also* "because you contain `'a`, it must be valid for the entire list's existence".
>
> 当我们尝试使用list的时候，编译器说：
>
> 1. 你mutably借用了self，那么直到'a结束之前，你不能再使用self了。
> 2. 因为（生成的reference）的生命周期是`'a`的，它就必须在list存在的时候保持valid（不能被释放）

这样便造成了list将它自己“钉住”，不能被访问，也没法被释放。

要解决这个问题必须引入新的Layout：

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // DANGER DANGER
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

接下来我们用C-style代码去解决问题。



#### 6.2 Unsafe Rust

Unsafe实际上是一件非常复杂的事情，为此专门有[一本书](https://doc.rust-lang.org/nightly/nomicon/)。

在Unsafe Rust里面，你被允许做一些额外的事情。像C语言一样，它可能导致一些和C语言类似的可怕的行为（like double-free）。

在Unsafe Rust里面我们被允许使用raw pointer。

一个简单的第一印象：`*mut T == &unchecked mut T`。



#### 6.3 Basics

注意看到上面我们新给出的Layout：

```rust
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // DANGER DANGER
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

所以我们现在`new`函数得改一下：

```rust
use std::ptr;

// defns...

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: ptr::null_mut() }
    }
}
```

然后就到写`push`函数，因为`self.tail`现在需要的是`* mut Node<T>`而不是`Option<&mut Node<T>>`，所以需要一些更改。观察到`Box`里面其实装的是一个堆上的对象，我们直接取出来然后取他的地址。不过这样的做法**并不安全**，因为一旦Box drop掉了这篇内存，我们便是在指向一块被free掉的内存。

改掉之后就成了这样：

```rust
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    // .is_null checks for null, equivalent to checking for None
    if !self.tail.is_null() {
        // If the old tail existed, update it to point to the new tail
        self.tail.next = Some(new_tail);
    } else {
        // Otherwise, update the head to point to it
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

现在我们很清晰的观察到，是没有所谓自引用问题的。

这样写不行，因为Rust对于raw pointer有非常严格的书写规范：

```bash
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             self.tail.next = Some(new_tail);
   |             ----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; 
   |             try dereferencing it: `(*self.tail).next`
```

改掉之后再看看，发现是这样的：

```bash
> cargo build

error[E0133]: dereference of raw pointer is unsafe and requires 
              unsafe function or block

  --> src/fifth.rs:31:13
   |
31 |             (*self.tail).next = Some(new_tail);
   |             ^^^^^^^^^^^^^^^^^ dereference of raw pointer
   |
   = note: raw pointers may be NULL, dangling or unaligned; 
     they can violate aliasing rules and cause data races: 
     all of these are undefined behavior
```

这里其实只是在要求我们对这块unsafe的代码加上`unsafe`标签，我们就这样加上就好了。

```rust
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    if !self.tail.is_null() {
        // Hello Compiler, I Know I Am Doing Something Dangerous And
        // I Promise To Be A Good Programmer Who Never Makes Mistakes.
        unsafe {
            (*self.tail).next = Some(new_tail);
        }
    } else {
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

作者接下来又谈了谈unsafe**污染**的问题。一旦`unsafe`在某个模块中被使用，整个模块就变得被污染了。但由于隐私，这样的污染并不可怕。这里的隐私指的就是私有成员变量/方法。只要公开的API组合没有导致不良事件发生，外部调用者便完全可以认为这段代码是安全的。

再来看看`pop`的写法：

```rust
pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        if self.head.is_none() {
            self.tail = ptr::null_mut();
        }

        head.elem
    })
}
```

作者又额外的强调了这里`self.tail = ptr::null_mut()`的问题，如果不加上这一句，则有可能导致运行时的错误（野指针，dangling pointer）

> safety is stateful.（这里指对上述的边界情况处理不佳的话，会在边界条件发生的时候引发运行时错误）



#### 6.4 Miri

>  Miri是一个sanitizer。

由于编译器没法帮我们检查代码的正确性，我们写出的测试又恰（zhi）好（liang）正（hen）确（di），那我们可能没法检查出一些未定义的行为。

所以我们需要有人帮我们检测运行时是否遵守了规则（e.g. 是否出现了undefined behavior）

这就是miri。

简单说来，miri的职能包含如下：

> - Out-of-bounds memory accesses and use-after-free
> - Invalid use of uninitialized data
> - Violation of intrinsic preconditions (an unreachable_unchecked being reached, calling copy_nonoverlapping with overlapping ranges, ...)
> - Not sufficiently aligned memory accesses and references
> - Violation of some basic type invariants (a bool that is not 0 or 1, for example, or an invalid enum discriminant)
> - Experimental: Violations of the Stacked Borrows rules governing aliasing for reference types
> - Experimental: Data races (but no weak memory effects)
> - Memory leaks
>
> 但Miri没办法抓住全部未定义行为。

我们可以用Miri来进一步测试我们的 代码：

```rust
cargo +nightly-2022-01-21 miri test
```

然后Miri向我们抛出如下错误：

```bash
error: Undefined Behavior: trying to reborrow for Unique at alloc84055, but parent tag <209678> does not have an appropriate item in the borrow stack

   --> \lib\rustlib\src\rust\library\core\src\option.rs:846:18
    |
846 |             Some(x) => Some(f(x)),
    |                  ^ trying to reborrow for Unique at alloc84055, 
    |                    but parent tag <209678> does not have an 
    |                    appropriate item in the borrow stack
    |

    = help: this indicates a potential bug in the program: it 
      performed an invalid operation, but the rules it 
      violated are still experimental
    
    = help: see 
      https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md 
      for further information
```

但什么是`borrow stack`?

6.5节见分晓吧。



#### 6.5 Attempting To Understand Stacked Borrows(栈借用)

上一节中，miri给出了我们一些文档，但这些文档是给编译器开发者或者学者看的。

但想要描述我们（潜在）的错误，一句话：**“堆叠借用”**

要想了解“堆叠借用”是什么，需要了解什么事“指针别名”。

其实这就是指向有重叠的内存片段的两个指针罢了。因为内存片段重叠，所以可能会引发内存访问问题。

对于这样的危险情况，我们必须要让编译器追踪指向同一块内存的这些指针。

于是我们分别追踪这些指针的活动情况。一个朴素的想法就是在一个指针的**生命周期**内，没有其他指向同一块内存的指针在活动。

于是我们需要维护这个“最新”的活动的指针。因此涉及到了栈，就有了"Stacked Borrow"的概念。

Rust里面针对于此有一个重新借用（reborrow）的概念，具体的可以看下面的代码，它出现了两个mutable ref，但是仍然正确编译：

```rust
let mut data = 10;
let ref1 = &mut data;
let ref2 = &mut *ref1;

*ref2 += 2;
*ref1 += 1;

println!("{}", data);
```

不过下面的这个版本就不可以了：

```rust
let mut data = 10;
let ref1 = &mut data;
let ref2 = &mut *ref1;

// ORDER SWAPPED!
*ref1 += 1;
*ref2 += 2;
```

这是因为我们reborrow一个mutable指针的时候，更老的指针就不可以再使用了，直到最新的reborrowed副本使用完毕。

这就很类似于栈的概念。因此rust中是有这个堆叠借用的概念的。

但是对于`unsafe`代码块来说，使用的是raw pointer，所以rust编译器没有办法监测stack borrow。

Miri试图解决它。

由于作者也不知道Miri具体工作的细节，也不知道如果遇到`&mut -> *mut -> &mut -> *mut`这样的deref + ref模式的时候应该怎么做，所以只能暂时遵循一些**启发式方法**来写`unsafe`代码：

> So what we're going to do is:
>
> 1. At the start of a method, use the input references to get our raw pointers
> 2. Do our best to only use unsafe pointers from this point on
> 3. Convert our raw pointers back to safe pointers at the end if needed

最后，to wrap up，代码惹怒miri的原因是我们使用了Box。

> 事实上，我们犯下的部分大错误是继续使用 Box！Box 中有一个特殊的注释，告诉编译器“嘿，这很像`&mut`，因为它唯一拥有该指针”。
>
> 但是我们保持在列表末尾的原始指针指向一个 Box，所以每当我们正常访问 Box 时，我们可能会使原始指针的“reborrow”无效！



#### 6.6 Testing Stacked Borrows

首先是借用的规则需要遵循栈的规则（FILO）

##### 基本借用

简述了不要违背栈规则。对于raw pointer来说，你需要用miri去追踪它，而编译器不会报什么错误。



##### 测试数组

下面测试用指针借用数组

```rust
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // 获取第 1 个元素的引用
    let ptr2_at_0 = ref1_at_0 as *mut i32;  // 裸指针 ptr 指向第 1 个元素
    let ptr3_at_1 = ptr2_at_0.add(1);       // 对裸指针进行运算，指向第 2 个元素

    *ptr3_at_1 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [3, 3, 0, ...]
    println!("{:?}", &data[..]);
}
```

```bash
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting read access 
to tag <1619> at alloc748+0x4 found in borrow stack.
 --> src\main.rs:8:5
  |
8 |     *ptr3_at_1 += 3;
  |     ^^^^^^^^^^^^^^^ no item granting read access to tag <1619>
  |                     at alloc748+0x4 found in borrow stack.
```

这里很奇怪，因为我们是完全按照栈规则去进行borrow & use的

事实上，我们把`ptr3`的`add`去掉，让它也指向0位置就不会报错了，代码省略。

所以是`add`的问题？非也，下面的代码保持了指针指向一个位置但是调用`add`一样通过：

```rust
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];            // Reference to 0th element
    let ptr2_at_0 = ref1_at_0 as *mut i32;   // Ptr to 0th element
    let ptr3_at_0 = ptr2_at_0;               // Ptr to 0th element
    let ptr4_at_0 = ptr2_at_0.add(0);        // Ptr to 0th element
    let ptr5_at_0 = ptr3_at_0.add(1).sub(1); // Ptr to 0th element


    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ptr4_at_0 += 4;
    *ptr5_at_0 += 5;
    *ptr3_at_0 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [20, 0, 0, ...]
    println!("{:?}", &data[..]);
}
```

所以问题是什么？

其实是因为**不能对数组的字段进行独立的借用**。

对于某些数据结构是可以的，比如结构体。但这里的数组就不可以：

```rust
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // Reference to 0th element
    let ref2_at_1 = &mut data[1];           // Reference to 1th element
    let ptr3_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
    let ptr4_at_1 = ref2_at_1 as *mut i32;   // Ptr to 1th element

    *ptr4_at_1 += 4;
    *ptr3_at_0 += 3;
    *ref2_at_1 += 2;
    *ref1_at_0 += 1;

    // Should be [3, 3, 0, ...]
    println!("{:?}", &data[..]);
}
```

```bash
error[E0499]: cannot borrow `data[_]` as mutable more than once at a time
 --> src\main.rs:5:21
  |
4 |     let ref1_at_0 = &mut data[0];           // Reference to 0th element
  |                     ------------ first mutable borrow occurs here
5 |     let ref2_at_1 = &mut data[1];           // Reference to 1th element
  |                     ^^^^^^^^^^^^ second mutable borrow occurs here
6 |     let ptr3_at_0 = ref1_at_0 as *mut i32;  // Ptr to 0th element
  |                     --------- first borrow later used here
  |
  = help: consider using `.split_at_mut(position)` or similar method 
    to obtain two mutable non-overlapping sub-slices
```

事实证明Miri不会根据下标追踪数组的借用。但是它做出了对应的让步，可以用slice来办到这一点。

> slice的下标可以越界没错，但是必须配合get方法使用，检验Option的值。

提示告诉我们用`.split_as_mut(pos)`来将数组划分。ok。

```rust
unsafe {
    let mut data = [0; 10];

    let slice1 = &mut data[..];
    let (slice2_at_0, slice3_at_1) = slice1.split_at_mut(1); 
    
    let ref4_at_0 = &mut slice2_at_0[0];    // Reference to 0th element
    let ref5_at_1 = &mut slice3_at_1[0];    // Reference to 1th element
    let ptr6_at_0 = ref4_at_0 as *mut i32;  // Ptr to 0th element
    let ptr7_at_1 = ref5_at_1 as *mut i32;  // Ptr to 1th element

    *ptr7_at_1 += 7;
    *ptr6_at_0 += 6;
    *ref5_at_1 += 5;
    *ref4_at_0 += 4;

    // Should be [10, 12, 0, ...]
    println!("{:?}", &data[..]);
}
```

测试通过，这也告诉我们borrow is less of stack but more of tree。

```rust
#![allow(unused)]
fn main() {
  unsafe {
      let mut data = [0; 10];

      let slice1_all = &mut data[..];         // Slice for the entire array
      let ptr2_all = slice1_all.as_mut_ptr(); // Pointer for the entire array (should be slice)

      let ptr3_at_0 = ptr2_all;               // Pointer to 0th elem (the same)
      let ptr4_at_1 = ptr2_all.add(1);        // Pointer to 1th elem
      let ref5_at_0 = &mut *ptr3_at_0;        // Reference to 0th elem
      let ref6_at_1 = &mut *ptr4_at_1;        // Reference to 1th elem

      *ref6_at_1 += 6;
      *ref5_at_0 += 5;
      *ptr4_at_1 += 4;
      *ptr3_at_0 += 3;

      // 在循环中修改所有元素( 仅仅为了有趣 )
      // (可以使用任何裸指针，它们共享同一个借用!)
      for idx in 0..10 {
          *ptr2_all.add(idx) += idx;
      }

      // 同样为了有趣，再实现下安全版本的循环
      for (idx, elem_ref) in slice1_all.iter_mut().enumerate() {
          *elem_ref += idx; 
      }

      // Should be [8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
      println!("{:?}", &data[..]);
  }
}
```

这一次不会报错。注意到代码和一开始数组借用版本的相似性。

下面是最简化的代码对比，代码已经在**rust playground上用miri测试**无误。

```rust
// slice + pointer
#![allow(unused)]
fn main() {
    unsafe {
        let mut data = [0; 10];
    
        let slice1_all = &mut data[..];         // Slice for the entire array
        let ptr1_at_0 = slice1_all.as_mut_ptr(); // Pointer for the entire array
        let ptr2_at_0 = ptr1_at_0;               // Pointer to 0th elem (the same)
        let ptr3_at_1 = ptr2_at_0.add(1);        // Pointer to 1th elem
    
        *ptr3_at_1 += 3;
        *ptr2_at_0 += 2;
        *ptr1_at_0 += 1;
    
        // Should be [8, 12, 4, 6, 8, 10, 12, 14, 16, 18]
        println!("{:?}", &data[..]);
    }
}

// original version
unsafe {
    let mut data = [0; 10];
    let ref1_at_0 = &mut data[0];           // 获取第 1 个元素的引用
    let ptr2_at_0 = ref1_at_0 as *mut i32;  // 裸指针 ptr 指向第 1 个元素
    let ptr3_at_1 = ptr2_at_0.add(1);       // 对裸指针进行运算，指向第 2 个元素

    *ptr3_at_1 += 3;
    *ptr2_at_0 += 2;
    *ref1_at_0 += 1;

    // Should be [3, 3, 0, ...]
    println!("{:?}", &data[..]);
}
```



##### 测试不可变引用(shared ref)

```rust
#![allow(unused)]
fn main() {
  fn opaque_read(val: &i32) {
      println!("{}", val);
  }

  unsafe {
      let mut data = 10;
      let mref1 = &mut data;
      let sref2 = &mref1;
      let sref3 = sref2;
      let sref4 = &*sref2;

      // Random hash of shared reference reads
      opaque_read(sref3);
      opaque_read(sref2);
      opaque_read(sref4);
      opaque_read(sref2);
      opaque_read(sref3);

      *mref1 += 1;

      opaque_read(&data);
  }
}
```

乱序读，虽然没使用指针，但是由于是不可变引用，没有问题。

如果加入裸指针？

```rust
#![allow(unused)]
fn main() {
  fn opaque_read(val: &i32) {
      println!("{}", val);
  }

  unsafe {
      let mut data = 10;
      let mref1 = &mut data;
      let ptr2 = mref1 as *mut i32;
      let sref3 = &*mref1;
      let ptr4 = sref3 as *mut i32;

      *ptr4 += 4;
      opaque_read(sref3);
      *ptr2 += 2;
      *mref1 += 1;

      opaque_read(&data);
  }
}
```

```bash
$ cargo run

error[E0606]: casting `&i32` as `*mut i32` is invalid
  --> src/main.rs:11:20
   |
11 |         let ptr4 = sref3 as *mut i32;
   |                    ^^^^^^^^^^^^^^^^^             ^^^^^^^^^^^^^^^^^
```

我们无法将shared ref转为raw pointer，需要多一个步骤：

```rust
#![allow(unused)]
fn main() {
	let ptr4 = sref3 as *const i32 as *mut i32;
}
```

这样编译器是会满意的，但miri没有：

```bash
MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: no item granting write access to 
tag <1621> at alloc742 found in borrow stack.
  --> src\main.rs:13:5
   |
13 |     *ptr4 += 4;
   |     ^^^^^^^^^^ no item granting write access to tag <1621>
   |                at alloc742 found in borrow stack.
```

> No items granting write access ...

一句话总结：

> **在借用栈中，一个不可变引用，它上面的所有引用( 在它之后被推入借用栈的引用 )都只能拥有只读的权限**

但是我们可以这样做：

```rust
#![allow(unused)]
fn main() {
  fn opaque_read(val: &i32) {
      println!("{}", val);
  }

  unsafe {
      let mut data = 10;
      let mref1 = &mut data;
      let ptr2 = mref1 as *mut i32;
      let sref3 = &*mref1;
      let ptr4 = sref3 as *const i32 as *mut i32;

      opaque_read(&*ptr4);
      opaque_read(sref3);
      *ptr2 += 2;
      *mref1 += 1;

      opaque_read(&data);
  }
}
```

即可变引用的裸指针可写，而shared ref的裸指针只能读。这当然合理。

最后作者检测了错误的出栈顺序导致的问题：

```rust
fn opaque_read(val: &i32) {
    println!("{}", val);
}

unsafe {
    let mut data = 10;
    let mref1 = &mut data;
    let ptr2 = mref1 as *mut i32;
    let sref3 = &*mref1;

    *ptr2 += 2;
    opaque_read(sref3); // Read in the wrong order?
    *mref1 += 1;

    opaque_read(&data);
}
```

最后的结果是miri报错：

```bash
$ cargo run

12
13

MIRIFLAGS="-Zmiri-tag-raw-pointers" cargo +nightly-2022-01-21 miri run

error: Undefined Behavior: trying to reborrow for SharedReadOnly 
at alloc742, but parent tag <1620> does not have an appropriate 
item in the borrow stack

  --> src\main.rs:13:17
   |
13 |     opaque_read(sref3); // Read in the wrong order?
   |                 ^^^^^ trying to reborrow for SharedReadOnly 
   |                       at alloc742, but parent tag <1620> 
   |                       does not have an appropriate item 
   |                       in the borrow stack
   |
```

特别要强调的是这里miri给出了`reborrow`相关错误的提示。



##### 测试内部可变性

略（众人：？



##### 测试Box

> `Box` 在某种程度上类似 `&mut`，因为对于它指向的内存区域，它拥有唯一的所有权

先给出一个一眼顶针的例子：

```rust
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *data += 10;
    *ptr1 += 1;

    // Should be 21
    println!("{}", data);
}
```

显然他违反栈规则。下面给出正确版本：

```rust
unsafe {
    let mut data = Box::new(10);
    let ptr1 = (&mut *data) as *mut i32;

    *ptr1 += 1;
    *data += 10;

    // Should be 21
    println!("{}", data);
}
```



#### 6.7 Layout + Basics Redux

在新的Layout设计中，将Box去掉，留下raw pointer：

```rust
// original
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // 好人一枚
}

type Link<T> = Option<Box<Node<T>>>; // 恶魔一只

struct Node<T> {
    elem: T,
    next: Link<T>,
}

// now
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>,
}

type Link<T> = *mut Node<T>; // 嘀，新的好人卡，请查收

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

> 请大家牢记：当使用裸指针时，`Option` 对我们是相当不友好的，所以这里不再使用。在后面还将引入 `NonNull` 类型，但是现在还无需操心。

##### 基本操作

New很简单，换成`ptr::null_mut()`

```rust
use ptr;

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: ptr::null_mut(), tail: ptr::null_mut() }
    }
}
```

对于push来说，由于我们不再使用Box，所以push生成新节点的时候需要重新考虑写法。

作者表示可以使用`std::alloc::alloc`但没必要，所以给出了一个很离谱的layout：

```rust
struct Node<T> {
    elem: T,
    real_next: Option<Box<Node<T>>>,
    next: *mut Node<T>,
}
```

> 先创建一个 `Box` ，并使用一个裸指针指向 `Box` 中的 `Node`，然后就一直使用该裸指针直到我们处理完 `Node` 且可以销毁它的时候。最后，可以将 `Box` 从 `real_next` 中 `take` 出来，并 `drop` 掉。

这样的做法是符合我们提出的**借用栈模型**的。因为我们是先创建Box，然后用raw pointer去reborrow Box里面的元素。之后我们遵循出栈规则，先弹出raw pointer，再弹出Box。

但作者并没有给出这样的示范，而是采用了`Box::into_raw`函数：

> ```
> pub fn into_raw(b: Box<T>) -> *mut T
> ```
>
> 消费掉 `Box` (拿走所有权)，返回一个裸指针。该指针会被正确的对齐且不为 null
>
> 在调用该函数后，调用者需要对之前被 Box 所管理的内存负责，特别地，调用者需要正确的清理 `T` 并释放相应的内存。**最简单的方式是通过 `Box::from_raw` 函数将裸指针再转回到 `Box`，然后 `Box` 的析构器就可以自动执行清理了**。
>
> 注意：这是一个**关联函数**，因此 `b.into_raw()` 是不正确的，我们得使用 `Box::into_raw(b)`。因此该函数不会跟内部类型的同名方法冲突。

最后的实现如下：

```rust
pub fn push(&mut self, elem: T) {
    unsafe {
        // 一开始就将 Box 转换成裸指针
        let new_tail = Box::into_raw(Box::new(Node {
            elem: elem,
            next: ptr::null_mut(),
        }));

        if !self.tail.is_null() {
            (*self.tail).next = new_tail;
        } else {
            self.head = new_tail;
        }

        self.tail = new_tail;
    }
}
```

然后是pop，注意我们需要调用`Box::from_raw`来使得其自动析构：

```rust
pub fn pop(&mut self) -> Option<T> {
    unsafe {
        if self.head.is_null() {
            None
        } else {
            let head = Box::from_raw(self.head);
            self.head = head.next;

            if self.head.is_null() {
                self.tail = ptr::null_mut();
            }

            Some(head.elem)
        }
    }
}
```

drop则是直接调用了pop，和之前没区别：

```rust
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while let Some(_) = self.pop() { }
    }
}
```

至于最后的测试的话，miri也没有报错。



#### 6.8 Extras

先按照原来的思路写好`Iter`和`IterMut`：

```rust
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}
```

但是这里的Iter和IterMut不是裸指针版本的，需要先修改一下：

```rust
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: *mut Node<T>,
}

pub struct IterMut<'a, T> {
    next: *mut Node<T>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { next: self.head }
    }
}
```

编译器报错如下：

```bash
error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:17:17
   |
17 | pub struct Iter<'a, T> {
   |                 ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData`

error[E0392]: parameter `'a` is never used
  --> src\fifth.rs:21:20
   |
21 | pub struct IterMut<'a, T> {
   |                    ^^ unused parameter
   |
   = help: consider removing `'a`, referring to it in a field, 
     or using a marker such as `PhantomData
```

因为我们这里的`'a`从来没被用到，所以必须要用`PhantomData`的`marker`标记。

> PhantomData 是零大小的类型
>
> 在你的类型中添加一个 `PhantomData<T>` 字段，可以告诉编译器你的类型对 `T` 进行了使用，虽然并没有。说白了，就是让编译器不再给出 `T` 未被使用的警告或者错误。
>
> 如果想要更深入的了解，可以看下 [Nomicon](https://doc.rust-lang.org/nightly/nomicon/)

作者这里并没有带我们使用`PhantomData`，而是继续选择ref写法。

> 能使用引用的原因是：我们可以创建一个迭代器，在其中使用安全引用，然后再丢弃迭代器。一旦迭代器被丢弃后，就可以继续使用 `push` 和 `pop` 了。
>
> 事实上，在迭代期间，我们还是需要解引用大量的裸指针，但是可以把引用看作**裸指针的再借用**。

```rust
pub struct IntoIter<T>(List<T>);

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }

    pub fn iter(&self) -> Iter<'_, T> {
        unsafe {
            Iter { next: self.head.as_ref() }
        }
    }

    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        unsafe {
            IterMut { next: self.head.as_mut() }
        }
    }
}
```

> 为了存储引用，这里使用 `Option` 来包裹，并通过 [`ptr::as_ref`](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_ref-1) 和 [`ptr::as_mut`](https://doc.rust-lang.org/std/primitive.pointer.html#method.as_mut) 来将裸指针转换成引用。

上述引言教会我们要学会使用`ptr::as_ref`和`ptr::as_mut`完成指针和ref的转换。

但是`as_ref`和`as_mut`会导致**无界生命周期**:

```rust
pub unsafe fn as_mut<'a>(self) -> Option<&'a mut T>
```

一个例子是：

```rust
fn f<'a, T>(x: *const T) -> &'a T {
    unsafe {
        &*x
    }
}
```

> 参数 `x` 是一个裸指针，它并没有任何生命周期，然后通过 `unsafe` 操作后，它被进行了解引用，变成了一个 Rust 的标准引用类型，该类型必须要有生命周期，也就是 `'a`

原版教程中，作者说：

> The way you *deal* with that is by putting it somewhere that *is* bounded, which usually just means "return this from a function as soon as possible so that the function signature limits it".

应该把这样的unsafe操作放在有生命周期限制的地方，而事实上作者的代码中也确实是这么做的：

```rust
impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.pop()
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_deref(); // match funciton signature
            &node.elem
        })
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.take().map(|node| {
            self.next = node.next.as_deref_mut(); // match funciton signature
            &mut node.elem
        })
    }
}
```

上述的代码可以通过miri，就不多说了

还有`peek`和`peek_mut`的代码，如下：

```rust
pub fn peek(&self) -> Option<&T> {
    unsafe {
        self.head.as_ref().map(|node| &node.elem)
    }
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    unsafe {
        self.head.as_mut().map(|node| &mut node.elem)
    }
}
```

同样是通过Miri。