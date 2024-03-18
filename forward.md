# 转发引用

这一篇我们来解决之前说的“万能引用”——开发者希望在用引用接受参数时，只需写一份实现，就可以同时处理左值和右值的输入。当然，只读左值引用是能做到的，比如：

```cpp
void f(const int& x) {
  std::println("ok!")
}

int main() {
  int a{};
  x(a);
  x(1);
}

```

但是这有缺陷：`f` 函数内部再也无从得知原始的值类别。假设它需要用 `test` 函数处理 `x`，而 `test` 对不同值类别的参数有不同的处理（比如对于短命的右值表达式，会移动某些资源），那么 `f` 无法最优化地调用 `test`：

```cpp
void test(int&) {
  std::println("lvalue");
}
void test(int&&) {
  std::println("rvalue");
}

void f(const int& x) {
  test(x);
}

int main() {
  int a{};
  x(a); // 输出 lvalue，说明 test 调用左值版本
  x(1); // 还是输出 lvalue，调用的仍然是左值版本
}
```

我们期望的是，在调用 `x(a)` 的时候调用 `test(int&)`，而调用 `x(1)` 的时候，则将值类别**转发**给 `test` 从而调用 `test(int&&)`。这就是**转发引用**需要出场的时刻。

转发引用（Forward reference）是指**函数模板中的引用类型形参** `T&&`，其中 `T` 必须是在模板推导中**被推导的类型**。

```cpp
template <typename T>
void f(T&& x) {} // x 是转发引用
// 或者
void f(auto&& x) {} // x 是转发引用
```

注意 `T` 必须是被推导的类型，如果 `T` 是固定的类型，则 `T&&` 只是普通的右值引用类型。转发引用类型的形参，在传入左值表达式做为实参时，推导为左值引用类型；在传入右值表达式作为实参时，推导为右值引用类型。

```cpp
void f(auto&& x) {
  // 单个括号的 decltype 输出变量的声明类型
  if constexpr (std::is_lvalue_reference_v<decltype(x)>) {
    std::println("lvalue ref");
  } else if constexpr (std::is_rvalue_reference_v<decltype(x)>) {
    std::println("rvalue ref");
  }
}

int main() {
  int a{};
  f(a); // 输出 lvalue ref
  f(1); // 输出 rvalue ref
}
```

但是现在我们只是确定了 `x` 声明的类型是左值引用类型还是右值引用类型，在使用表达式 `x` 时，仍然总是得到左值表达式。（第三次重复：所有引用类型变量作为表达式都是左值。）

```cpp
void test(int&) {
  std::println("lvalue");
}
void test(int&&) {
  std::println("rvalue");
}

void f(auto&& x) {
  test(x); // 总是调用 test(int&);
}

int main() {
  int a{};
  x(a); // 输出 lvalue，说明 test 调用左值版本
  x(1); // 还是输出 lvalue，调用的仍然是左值版本
}
```

此时我们需要用 `std::forward`。这个函数的用法很古怪，是函数模板但是不能推导，需要显式传入一个类型参数 `T`，即转发引用 `T&&` 中的 `T`：

```cpp
template <typename T>
void f(T&& x) {
  test(std::forward<T>(x));
}
// 或者
void f(auto&& x) {
  test(std::forward<decltype(x)>(x));
}
```
这样，`std::forward<T>(x)` 表达式的左右值类别就能对应于转发引用 `x` 原本的值类别。现在，`f(a)` 和 `f(1)` 能够正确调用对应值类别的 `test`。

这里有一些细节，叫做“引用折叠规则”。它可以解释为何转发引用 `T&&` 能通过实参值类别推导出正确的引用类型，也能解释 `std::forward` 是怎样实现的。但是我认为它对于大部分 C++ 开发者来说不重要，所以我只会在注释里面用最简单的文字描述一遍。

## 注释
- 引用折叠规则是指，若 `T` 是左值引用类型， `T&&` 会视为 `T` 类型；若 `T` 不是左值引用类型，那 `T&&` 就是字面意思上的右值引用类型。
- 转发引用的原理是，对于转发引用 `T&&`，若实参是 `A` 类型的左值，`T` 被推导为 `A&` 类型；否则实参是 `A` 类型的右值，`T` 被推导为 `A` 类型。从而，`T&&` 根据实参的左右值计算为 `A&` 或 `A&&` 类型（应用引用折叠规则）。
- `std::forward` 的原理是，它总是将形参转换为 `T&&` 类型，其中 `T` 是模板形参。因此，如果 `x` 具有左值引用类型，则 `T` 或 `decltype(x)` 具有左值引用类型，`std::forward` 返回的 `T&&` 也是左值引用类型。否则，`std::forward` 返回右值引用类型。根据前文说的引用类型到调用表达式的关系，`std::forward<T>(x)` 在变量 `x` 具有 `T&` 类型时得到左值表达式，在变量 `x` 具有 `T&&` 类型时得到亡值表达式。
