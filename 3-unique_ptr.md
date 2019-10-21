# C++11 std::unique_ptr

C++ 中的指针是一个强大并且危险的工具。

```
Widget* p1 = new Widget;
Widget* p2 = p1;
```

此时 `p1` 和 `p2` 指向同一个 `Widget` 实例。可以认为他们共享同一份资源。此时对资源的任何修改，`p1` 和`p2` 都能同时感知到。指针的拷贝开销非常小，所以在传递大型数据类型的时候，比起把整个数据类型都拷贝一份新的，使用指针可以极大的提升程序的效率。

但是，由于 C++ 中没有自动垃圾搜集机制，资源的释放需要显示的调用 `delete` ，这造成的问题有两种：

* 调用 `delete p1` 释放资源，但是程序中的其他地方继续使用 `p2` ，由于此时 `p2` 已经成为一个悬空指针（dangle），使用它通常意味着程序崩溃。
* 由于 `p1` 和 `p2` 可能位于程序中的任何地方，所以可能错误的多次调用 `delete`，结果就是多次释放同一资源，也意味着程序崩溃。

鉴于此，C++11 中引入智能指针的概念（Smart pointer），主要就是把 C++ 中的指针的各种功能进行了分类。其中 `std::unique_ptr` 是非常有特点的一种，它舍弃了指针中的共享同一份资源的能力，表示了对资源的独占式使用，除此之外，它的传递开销和普通指针一致，并且不再需要显示的释放资源。

## 基本使用

```c++
std::unique_ptr<Widget> p1(nullptr);
std::unique_ptr<Widget> p2(new Widget);
std::unique_ptr<Widget> p3 = make_unique<Widget>(); // C++14
```

这是 `std::unique_ptr` 的初始化方式。`p1` 指向空指针，`p2` 指向一个新创建出来的 `Widget` ，`p3` 使用 `std::make_unique` 创建。

当 `std::unique_ptr` 离开自己的作用域之后，即析构函数被调用，此时会自动调用 `delete` 释放资源。

```c++
// std::unique_ptr p4 = p3; 不合法的使用方式
std::unique_ptr p4 = std::move(p3);
```

`std::unique_ptr` 表示的是独占资源的语意，所以不允许拷贝。因为当拷贝发生也就意味着两者共同指向同一份资源。相反，`std::unique_ptr` 支持移动拥有权。上边的例子中，`p4` 将持有 `p3` 之前指向的资源，而 `p3` 会被设成空指针。资源的拥有权从 `p3` 移动到了 `p4` 。

## 自定义删除器（deleter）

我们在写代码的时候，需要管理的并不只是简单的内存资源。像打开的文件，socket，数据库连接等等都可以视为一种资源。而普通的内存我们可以使用 `delete` 来释放，像文件，socker，数据库连接等都有自己对应的资源释放方式。`std::shared_ptr` 默认使用 `delete` 来释放资源，但是也允许我们自定义资源删除器。

```c++
auto fileDeleter = [](File * f) {
  closeFile(f);
}

File * openFile = open_file();
std::unique_ptr<File, decltype(fileDeleter)> spf(openFile, fileDeleter);
```

上边这个例子，当 `spf` 离开作用域被销毁的时候，自定义 `fileDeleter` 就会调用，从而释放文件资源。这里我们使用了一个 lambda 表达式来自定义删除器，原因是如果使用不  capture 任何变量的 lambda 表达式，`std::unique_ptr` 的大小将和指针一样大小。相反，如果我们使用函数指针作为自定义删除器，`std::unique_ptr` 的大小会变为两倍指针大小。