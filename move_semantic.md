# 移动语义

之所以 C++ 大费周章将表达式分为左值和右值两类（或者说泛左值和纯右值两类），就是为了引入移动语义。

我们先来考察 `std::unique_ptr`，简单来说它就是指针的包装类，但是期望同一个对象任何时刻只能被唯一一个 `std::unique_ptr` 所指。比如下面的代码是错误：

```cpp
std::unique_ptr<int> pa(new int{});
std::unique_ptr<int> pb(pa); // 错误
pb = pa;                     // 错误
```

因为这里试图从 `pa` 复制初始化 `pb`，然而如果允许这样做的话，`pa` 和 `pb` 就会指向同一个 `int` 对象。所以，`std::unique_ptr` 的复制构造函数（以及复制赋值运算符重载）被设置为 `= delete` 的。

```cpp
template <typename T>
class unique_ptr {
  unique_ptr(const unique_ptr<T>&) = delete;
  unique_ptr<T>& operator=(const unique_ptr<T>&) = delete;
};
```

但是，我们可以写出如下的代码：

```cpp
std::unique_ptr<int> pa;
pa = std::make_unique<int>();
```

其中 `std::make_unique` 是返回 `std::unique_ptr` 类型的函数。为什么这里的 `operator=` 就能通过编译，明明刚才说删除了 `operator=` 啊？这是因为 `std::unique_ptr` 的 `operator=` 是**移动**赋值运算符重载：

```cpp
template <typename T>
class unique_ptr {
  unique_ptr<T>& operator=(unique_ptr<T>&& other) {
    // [...]
  }
};
```

当我们说“移动”赋值运算符时，指的就是形参类型是**右值引用**而非左值引用。于是，出现在 `std::unique_ptr` 赋值运算符右侧的操作数只能是右值。于是，赋值给 `std::unique_ptr` 的对象必然是短生命周期的（这是右值设计的定义）。因此，`std::unique_ptr` 允许了这种 `operator=`；尽管赋值，反正右侧操作数是马上就析构的右值，一旦析构完成，仍然是只有唯一一个指针指向 `int`。

在实现上，`std::unique_ptr` 会做如下事：

```cpp
unique_ptr<T>& operator=(unique_ptr<T>&& other) {
  this->_ptr = other._ptr;
  other._ptr = nullptr;
  return *this;
}
```
其中 `_ptr` 即 `std::unique_ptr` 包装的指针成员。整个赋值的操作就是将自己指向右侧操作数所指的对象，然后将右侧操作数的指向置为空指针。（因为 `std::unique_ptr` 析构的时候会自动 `delete _ptr`，所以需要置空避免重复释放。）

再来看 `std::string`。众所周知，`std::string` 的实现是“胖指针”——即至少保存两个成员，一个是指针指向数据块，另一个是长度信息。我们就用 `_data` 和 `_size` 两个成员来代表。那么，通常 `std::string` 的复制赋值运算符重载是这样做的：

```cpp
string& operator=(const string& other) {
  if (this == &other) return *this;
  delete[] this->_data;
  this->_data = new char[other._size];
  this->_size = other._size;
  std::memcpy(this->_data, other._data, other._size);
  return *this;
}
```

即：释放原始数据内存，分配新内存，然后复制内存数据。然而这对于移动赋值的情形是不必要的：如果我将纯右值 `std::string` （记这个表达式为 `e`）赋值给对象 `a`，如果按照刚刚的实现，就会发生如下事：
- 首先调用复制赋值：
  - 释放 `a._data`；
  - 分配 `a._data`；
  - 从 `e._data` 复制内存；
- 由于 `e` 是纯右值，执行赋值完成后立即调用 `e` 的析构：
  - 释放 `e._data`。

但是我们明明可以有更简单的实现：
- 释放 `a._data`。
- 将 `e._data` 赋值给 `a._data`，即改变 `a._data` 的指向。
- 将 `e._data` 置空。

这种实现下，不需要任何大块内存的复制，只需要改变一个指针指向就可以做到修改 `std::string a` 的值。之所以能这样做，是因为我们在刚才的过程中**复用了已有的存储**，而能复用的前提则是，原来使用这片内存的对象接近生存周期的末尾，可以安全地解除其对这片内存的“所有权”。

因此，可以这样实现 `std::string` 的移动赋值运算符重载：

```cpp
string& operator=(string&& other) {
  if (this == &other) return *this;
  delete[] this->_data;
  this->_data = other._data;
  this->_size = other._size;
  other._data = nullptr;
  return *this;
}
```

另一种更巧妙的实现是使用交换，原理留作思考。

```cpp
string& operator=(string&& other) {
  std::swap(this->_data, other._data);
  std::swap(this->_size, other._size);
  return *this;
}
```

类似移动赋值运算符重载，还有**移动构造函数**，也即将复制构造函数中的 `const T& other` 改为 `T&& other`，并在实现上“窃取” `other` 的资源以“移动”它们。定义移动构造函数和移动赋值运算符重载，以实现某个类“禁止复制”或“优化复制”的手段，就称为该类的**移动语义**。

## 注释

- 本文中使用 `std::unique_ptr` 或 `std::string` 仅作为移动语义演示，不代表任何标准库实现，也不保证完全符合标准。
