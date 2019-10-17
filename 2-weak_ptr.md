# C++11 `std::weak_ptr`

我们知道 `std::shared_ptr` 像指针一样，不同的 `std::shared_ptr` 指向同一份资源，共享这一份资源。同时所有的 `std::shared_ptr` 拥有了这份资源的管理权。在最后一个 `std::shared_ptr` 出作用域的时候，会销毁他们所指向的资源。所以我们可以说 `std::shared_ptr` 是共享了资源的拥有权。

但是，有时候我们会需要一个类似 `std::shared_ptr` ，指向同一份资源，但是并不共享资源的拥有权。 `std::weak_ptr` 就是这样的指针。 `std::weak_ptr` 支持的操作非常有限，既不能提取内部的资源以调用其方法，甚至不能测试是否为空。但是当所有指向同一资源的 `std::shared_ptr` 都出了作用域，对应的资源都被销毁后，我们可以使用 `std::weak_ptr` 去检测这一变化。 这是所有其他指针都不具备的能力。如果从传统指针的角度出发，我们可以认为 `std::weak_ptr` 可以检测出指针是否悬空（dangle）。

## 基本使用

`std::weak_ptr` 可以看成是 `std::shared_ptr` 的伴生指针，从使用接口上我们就能看得出来。

```c++
auto sp = std::make_shared<Widget>();
std::weak_ptr<Widget> wp(sp);
```

`std::weak_ptr` 通过 `std::shared_ptr` 来构造。之后 `std::weak_ptr` 和 `std::shared_ptr` 指向相同的资源。但是 `std::weak_ptr` 不影响 `std::shared_ptr` 的引用计数。

```c++
sp = nullptr;
if (wp.expired()) {
  // wp is dangle
}
```

`sp` 赋值为空之后，`std::shared_ptr` 的引用计数为 0，指向的资源被销毁。此时 `wp` 成为悬空指针（dangle）。如果是普通的指针，在这种情况下将非常危险，因为普通指针并没办法知道自己指向的资源已经销毁。直接使用悬空指针通常会导致程序崩溃。而 `std::weak_ptr` 可以直接通过 `expired()` 方法来检测对应的资源是否已经销毁。

```c++
std::shared_ptr<Widget> sp2 = wp.lock();
if (sp2) {
  sp2->doSomthing();
}
```

当我们需要使用 `std::weak_ptr` 以及对应的 `std::shared_ptr` 共同指向的资源的时候。`lock()` 方法可以构造一个新的 `std::shared_ptr`。之后正常使用即可。

## 打破循环引用

`std::weak_ptr` 最大的作用是打破 `std::shared_ptr` 中的循环引用。

我们来看一个极度简化的例子：

```c++
struct Person {
  std::shared_ptr<BankAccount> account;
}
struct BankAccount {
  std::shared_ptr<Person> person;
}
```

其中 `Person` 通过 `std::shared_ptr` 持有 `BankAccount` 这一份资源。`BankAccount` 也通过 `std::shared_ptr` 持有 `Person` 。

```c++
std::shared_ptr<Person> spp;
std::shared_ptr<BankAccount> spa;
```

我们通过 `std::shared_ptr` 分别构造了 `Person` 和 `BankAccount` ，此时 `spp` 和 `spa` 所指向资源的引用计数都为 1。

```c++
spp.account = spa;
spa.person = spp;
```

分别将上述资源赋值给彼此。毕竟我们通过一个人（Person）应该能查到她的银行账户（BankAccount），反过来，也应该能够通过银行账户（BankAccount）查到对应的持有人（Person）。

此时，`spp` 和 `spa` 所指向资源的引用计数都为 2。

```c++
spp = nullptr;
spa = nullptr;
```

销毁 `spp` 和 `spa` ，此时他们指向资源的引用计数分别减 1，结果都为 1。这就造成了资源泄漏，因为 `spp` 和 `spa` 都已经被我们销毁，所以我们没办法再访问到他们所指向的资源。而引用计数不为 0 也意味着他们所指向的资源不会释放。

**在引用计数的资源管理中，循环引用会导致资源无法正确释放。**

本例中，`Person` 和 `BankAccount` 互相持有对方，所以产生了「循环」引用。

解决方案如下：

```c++
struct Person {
  std::shared_ptr<BankAccount> account;
}
struct BankAccount {
  std::weak_ptr<Person> person;
}

std::shared_ptr<Person> spp;
std::shared_ptr<BankAccount> spa;

spp.account = spa;
spa.person = spp;

spp = nullptr;
spa = nullptr;
```

把 `BankAccount` 指向的 `Person` 改为 `std::weak_ptr`。`spp` 和 `spa` 创建出来的时候引用计数都为 1。互相赋值之后，因为 `spa` 通过 `std::weak_ptr` 持有 `spp` ，并不会改变其引用计数，`spp` 指向资源的引用计数仍然为 1。而 `spa` 指向的资源的引用计数和之前一样，仍然为 2 。

当销毁 `spp` 的时候， `spp` 指向资源的引用计数减 1 变为 0 。触发资源释放操作，具体来说就是调用 `Person` 的析构函数，而 `Person` 的析构函数调用结束，也就意味着 `Person` 所指向的 `account` 被销毁。此时 `account` 就是 `spa` ，销毁也就意味着 `spa` 的引用计数减 1 变为 1 。最后，销毁 `spa` 导致引用计数再减 1 变为 0 。`spa` 引用计数为 0 也就意味着 `spa` 所指向的资源被销毁。

所有资源都得到了正常销毁。

## 其他用法

虽然 `std::weak_ptr` 的主要是用于打破 `std::shared_ptr` 的循环引用。但如开头所说，`std::weak_ptr` 最大的优势是可以访问共享资源，同时又可以检测资源是否被销毁。有一些其他的场景也很适合使用。

### 场景1 缓存（cache）

```c++
std::map<id, std::weak_ptr<Widget>> cache;
```

在 `cache` 中，我们把创建好的资源用 `std::weak_ptr` 持有一份引用。这样如果系统中其他部分的对应资源还在使用中，再次请求使用的时候，我们就可以从 `cache` 中几乎无成本的再次取出。不用 `std::shared_ptr` 的原因是我们并不希望资源永远的存在于系统中。当系统中其他地方对资源已经没有引用，就会释放资源。下次再请求的时候，我们就可以通过 `std::weak_ptr` 判断出对应资源已经释放，此时就需要新构造一份。

实际工程中的缓存是一个很复杂的话题，有各种各样的管理策略。如果管理策略正好和我们这里讲的相符合，那么 `std::weak_ptr` 就是一个很好的解决方案。

### 场景2 观察者模式（observer patter）

Observer 模式通常用于通知。当 Observable 的状态变化的时候，会通知一系列相关的 Observer。此时 Observable 并不管理 Observer 的生命周期，但是又需要知道 Observer 是否有效，以决定是否需要继续通知。`std::weak_ptr` 就是一个非常好的选择。

```c++
struct Observable {
  std::vector<std::weak_ptr<Observer>> obervers_;
  void notify() {
    for (auto& o : obervers_) {
      std::shared_ptr sp = o.lock();
      if（sp) {
        sp->update();
      }
    }
}
```



