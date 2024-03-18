# `std::move`

作为本篇的引入，考察 `std::swap` 的实现——即如何交换两个变量的值。编程基础告诉我们，最简单的写法就是：

```cpp
template <typename T>
void swap(T& a, T& b) {
  T temp = a;
  a = b;
  b = temp;
}
```

但是这并不是正确的实现。如果试图用这样实现的 `std::swap` 交换两个 `std::unique_ptr`……能做到吗？我们先把模板的实例化写出来：

```cpp
void swap(std::unique_ptr<int>& a, std::unique_ptr<int>& b) {
  std::unique_ptr<int> temp = a;  // boom
  a = b;                          // boom!
  b = temp;                       // boom!!
}
```

然后你会发现整个函数就三条语句，然后三条语句全是编译错误。为什么？在上一篇中提到了，`std::unique_ptr` 删除了它的复制构造和复制赋值重载。因为没有复制构造，所以第一行试图复制初始化 `temp` 是做不到的；因为没有复制赋值重载，所以后面两行的赋值也是做不到的。这下傻眼了。

怎么办呢？我们说虽然 `std::unique_ptr` 没有复制构造，但是有移动构造；还有移动赋值重载。但是移动构造期望的是右值，我们这里的 `a` `b` `temp` 都是左值表达式。我们需要的是**将左值转换为右值**，然后传给移动构造或移动赋值重载。

需要强调的是，从左值到右值转换只能是从左值转换到亡值。一旦一个值有了关联对象就不可能再解除关联，所以再怎么转换也都是在泛左值的内部转来转去。那么手动转换到右值的语法是什么呢？其实就是显式类型转换，但是目标类型要写成 `T&&`。比如：

```cpp
int a{}; // a 是左值
static_cast<int&&>(a); // 整个表达式得到右值
// 或者你写成这样，也不是不行，就是不够现代
(int&&)(a);
```

下面的代码可供演示这一转换是成功的：

```cpp
#include <print>
void test(int& val) {
  std::println("this is lvalue");
}
void test(int&& val) {
  std::println("this is rvalue");
}
int main() {
  int a;
  test(a);                     // output lvalue
  test(static_cast<int&&>(a)); // output rvalue
  test((int&&)(a));            // output rvalue
}
```

通过将参数转换为右值（亡值），我们可以编写出正确的 `std::swap` 实现：

```cpp
template <typename T>
void swap(T& a, T& b) {
  T temp = static_cast<T&&>(a);
  a = static_cast<T&&>(b);
  b = static_cast<T&&>(temp);
}
```

有点麻烦。标准库提供了 **`std::move`** 来代替 `static_cast<T&&>`（具体来说，`std::move` 返回的是右值引用类型，然后根据标准规定，其调用表达式就是右值），所以进一步简写就是：

```cpp
template <typename T>
void swap(T& a, T& b) {
  T temp = std::move(a);
  a = std::move(b);
  b = std::move(temp);
}
```

停一下！为什么这里随随便便就转过去了？右值和左值有语义上的区别啊，怎么能说转就转呢？我们回顾一下语义上的区别：右值具有短生命周期，左值具有长生命周期。右值在使用完，其关联对象是被期望稍后立即释放的，从而我们才可以合理地“借用”其中的资源而不违背这些具有移动语义的类的设计。我们把左值转换到右值，意味着告诉要调用的函数：**尽管传入的参数的对象的生命周期还很长，但是请你将它视为和右值一样短**。

于是，函数就会认为，你传入的这个对象不会再被使用（顶多会自动析构），然后将对象里面的东西搞乱来实现移动语义（比如我们上一节提到的两个例子，把参数的对象内部的指针置空）。所以，得出以下重要的结论：

**当你将对象 `std::move` 并传入某个函数之后，你就不要再使用它了**；至少在下一次赋值之前不要读取它。或者用更确切的话说，一个对象经由 `std::move` 使用后，其状态是不确定的，再使用这个对象当前的值通常是未定义行为。

```cpp
void f(MyClass x);

MyClass obj;
f(std::move(obj));

// 在 obj 下一次被赋值之前，没人知道里面到底是什么个状态
std::cout << obj; // ??? 可能导致未定义行为
```

当然，我们这个“资源被移动走”的对象之后还能通过被赋值（出现在 `operator=` 左侧）回到合法的状态，比如刚刚 `std::swap` 中 `a` 被 `std::move` 到 `temp` 里，但是稍后通过 `a = ...` 又得到了合法的状态（从 `b` 移动赋值过来的）。
