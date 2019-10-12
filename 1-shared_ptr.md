# 智能指针 `std::shared_ptr`

`shared_ptr` 是 C++11 中引入的智能指针。在共享资源的时候使用。底层实现是通过引用计数的方式来管理资源。当拷贝 `shared_ptr` 的时候，引用计数会自动加一。而当调用 `shared_ptr` 的析构函数时引用计数减一，如果引用计数为 0，则自动释放资源。

## 用于替换 `new` 和 `delete` 

使用传统的 `new` 和 `delete` 的代码通常如下

```c++
Widget * p1 = new Widget;
Widget * p2 = p1; 
delete p1;
// delete p2; error, double delete
```

其中 `p1` 和 `p2` 指向同一份 `Widget` ，可以理解为他们共享都一份资源。这里常见的问题有两个。一是如果忘记调用 `delete` 那么就会产生内存泄漏；二是如果多次调用 `delete` 等于在已删除的资源上再次调用，通常都会是程序崩溃退出。如果只是我们示例代码中，一般并不会有太大的问题，但考虑到指针可能会遍布不同的类，不同的文件，所以出现问题的概率非常之大，很多软件的 bug 都来源于此。

使用 `std::shared_ptr` 可以很好的避免这种情况

```c++
std::shared_ptr<Widget> sp1(new Widget);
std::shared_ptr<Widget> sp2 = sp1;
```

不再需要 `delete` ，在 `sp1` 和 `sp2` 作用域结束的时候，`std::shared_ptr` 会自动帮我们调用 `delete`。

当然，在这个例子中并没有特别的体现出 `std::shared_ptr` 的优势，但是，考虑到指针要传递到不同的类，横跨很多不同的文件。使用 `std::shared_ptr` 来表示共享同一个资源就变得非常有必要。试想一下，如果 `sp1` 和 `sp2` 分别在不同的类中，在这里使用普通的指针就会面临很头痛的谁负责 `delete` 的问题。

## Control block

`std::shared_ptr` 底层的引用计数机制使用一个额外数据结构实现。我们可以称其为 Control block。`std::shared_ptr` 中有一个指针指向 Control block，所以通常来说，`std::shared_ptr` 是普通指针的两倍大小（原始资源的指针，和指向 Control block 的指针，一共两个指针）。

当然这只是指 `std::shared_ptr` 本身的大小。Control block 本身还会需要额外的存储空间。其中主要存储了

* `std::shared_ptr` 的引用计数器。
* `std::weak_ptr` 的引用技术器。（`std::weak_ptr` 主要用于打破循环引用）
* 其他，比如自定义的删除器（deleter），内存分配器（alloctor）等。

上边例子中的 `sp1` 和 `sp2` 中都会指向一个独立的 Control block。这其实就是 `std::shared_ptr` 中的 shared（共享）的由来，只要是针对同一个资源，`std::shared_ptr` 共享同一个管理模块。

## 自定义删除器（deleter）

我们在写代码的时候，需要管理的并不只是简单的内存资源。像打开的文件，socket，数据库连接等等都可以视为一种资源。而普通的内存我们可以使用 `delete` 来释放，像文件，socker，数据库连接等都有自己对应的资源释放方式。`std::shared_ptr` 默认使用 `delete` 来释放资源，但是也允许我们自定义资源删除器。

```c++
void fileDeleter(File * f) {
  closeFile(f);
}
std::shared_ptr<File> spf(openFile, fileDeleter);
```

上边这个例子，当 `spf` 离开作用域被销毁的时候，自定义 `fileDeleter` 就会调用，从而释放文件资源。

## `std::make_shared` 

`std::make_shared` 用于生成 `std::shared_ptr` ，因此，生成 `std::shared_ptr` 有两种方法：

```c++
std::shared_ptr<Widget> sp1(new Widget);

auto sp2 = std::make_shared<Widget>();
```

使用 `std::make_shared` 有两个好处。

**可以防止内存泄漏**

```c++
void process(std::shared_ptr<Widget> spw, int state);
int currentState();

process(std::shared_ptr<Widget>(new Widget), currentState());
```

上边的例子看起来完全正常，但是有一个非常隐秘的内存泄漏的可能。

首先，在 C++ 中发起函数调用的时候，在函数开始执行之前，函数参数会先求值，具体到前边的例子，在 `process` 的函数体执行之前，会先执行 `std::shared_ptr<Widget>(new Widget)` 以及调用 `currentState()`。同时 `new Widget` 作为 `std::shared_ptr` 构造函数的参会，会早于构造函数的执行。但是比较特殊的是，由于 `std::shared_ptr` 的构造函数和 `currentState()` 是在同一执行层面，在 C++ 中并没有要求他们的先后执行顺序。所以有一种可能的执行顺序是

`new Widget --> currentState() --> std::shared_ptr()`

这在 C++ 中完全合法。这时候如果在执行 `currentState()` 时抛出异常。由于 `std::shared_ptr` 的构造函数还没有被调用，所以 `new Widget` 生成的对象并没有被管理起来。结果就是对应的内存不会被释放，造成了内存泄漏。使用 `std::make_shared` 可以避免这种情况。

```c++
process(std::make_shared<Widget>(), currentState());
```

如果 `std::make_shared<Widget>()` 先执行，而 `currentState()` 抛出了异常，由于此时 `std::shared_ptr` 已经构造完成，所以在异常抛出之后，析构函数也会被调用，资源得到正常释放。

**执行速度相对会快一些**

回到我们前边所说，`std::shared_ptr` 通过一个指针指向 Control block 来管理引用计数。所以在直接生成 `std::shared_ptr` 时会发生两次内存分配，一次是 `new Widget` ，另一次是生成 Control block 时。而 `std::make_shared` 会把这两块内存合并到一起，使用一次 `new` 操作完成内存分配，节省一次内存分配的时间。

**问题**

`std::make_shared` 的问题是无法接收用户自定义的删除器。所以这时候我们必须得回到直接构造的方式。为了避免异常发生时的内存泄漏，我们修改代码如下：

```c++
std::shared_ptr<Widget> sp1(new Widget, customDeleter);
process(sp1, currentState());
```

## 如何把 `std::shared_ptr` 作为参数传递

C++ 中的参数传递主要有三种方式

```c++
void share(std::shared_ptr<widget>);            // share -- "will" retain refcount

void may_share(const std::shared_ptr<widget>&); // "might" retain refcount

void reseat(std::shared_ptr<widget>&);          // "might" reseat ptr
```

三种方式都很常见，虽然在 `std::shared_ptr` 中使用起来并没有特别大的区别。但对于一个好的接口设计，区分三种方式还是很有意义。

第一种方式，传值。这种方式在拷贝 `std::shared_ptr` 时会使引用计数加一，这也意味着在函数调用的时候会持有对应资源，资源不会被释放掉。因此，传值意味着共享资源的语意。

第二种方式，传 const 引用。这种方式比较特殊，在参数中并不会发生拷贝操作，因此也不会导致引用计数加一。这时候我们就需要审视函数的实现，如果在实现中没有把参数拷贝给一个新的 `std::shared_ptr` 那么这种参数传递并不合理。因为 `std::shared_ptr` 表示的是一种共享语意。而不把 const 引用拷贝给新的 `std::shared_ptr` 则表示这个函数只是短暂的使用一下智能指针所指向的类 `Widget` 的某个功能。这时候为了函数更广法的适用性，我们应该把参数变为 `Widget*` 或者 `Widget&` 即可。

第三种方式，传引用。传引用在语意上表示我要修改参数。注意这里的参数是 `std::shared_ptr` 而不是 `Widget` ，因此，这种传递方式表示我们 要修改 `std::shared_ptr` 本身。所以在函数体中，我们要么给它赋新的值，要么调用 `reset` 等操作，让传进来的 `std::shared_ptr` 发生改变。如果没有这些操作，那么就需要考虑其他的参数传递方式。