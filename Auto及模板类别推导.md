## 模板类别推导

函数模板经常表示为以下形式：

```c++
template <typename T>
void f(ParamType param);
```

类别T和ParamType在编译期被推导，二者的类别往往不一致：

1. **ParamType是指针或左值引用**

```c++
template <typename T>
void f (T& param);
```

- 推导时忽略param的引用属性，const属性则被保留下来
- 若传入的param类型为T，则ParamType是T&
- 若传入的param类型为const T&，则ParamType是const T&
- 若传入的param是一个数组名，则ParamType被推导为数组类别，且包含数组的尺寸
- 若传入的param是一个函数名，则ParamType被推导为函数引用

<br>

2. **ParamType是通用引用**

```c++
template <typename T>
void f (T&& param);
```

- 若传入的param是左值，则ParamType是T&类别，即左值引用
- 若传入的param是右值，则ParamType是右值引用
- param的const属性同样被保留下来

<br>

3. **ParamType是值类型**

```c++
template <typename T>
void f (T param);
```

- 若传入的param具有引用或顶层const/volatile属性，则忽略
- 若传入的param是一个指针/引用，且具有底层const属性，则保留
- 若传入的param是一个数组名，则ParamType为指针
- 若传入的param是一个函数地址，则ParamType被推导为函数指针

<br>

<br>

## auto类别推导

auto类别推导在大部分情况下与模板类别推导一致：

1. **auto类型用指针或左值引用修饰**

```c++
const int x = 27;
auto& rx = x; // rx = const int&

template<typename T> void func_for_rx (T& param);
```

-  与模板类型推导中ParamType为指针或左值引用时等价
- 忽略param的引用属性，保留const属性

<br>

2. **auto类型用右值引用修饰**

```c++
auto&& uref = x; // x = int&
auto&& uref2 = 25; // x = int&&

template<typename T> void func_for_uref(T&& param);
```

- 与模板类型推导中ParamType为通用引用时等价
- const属性得以保留

<br>

3. **auto类型为值引用**

```C++
const char name[] = "sharingan";
auto arr = name; // x = const char*
int x = 5;
const int* const cp = &x;
auto ap = cp; // ap = const int*

template<typename T> void func_for_arr(T param);
```

- 与模板类型推导中ParamType为值类型时等价
- 数组退化为指针
- param为指针/引用时，顶层const被忽略，底层const被保留

<br>

**当使用列表初始化形式时，auto会推导为initializer_list，而模板类型则会推导失败，这是二者唯一存在区别的地方**。

```c++
auto x = {1, 2, 3}; // x = initializer_list<int>

template<typename T> void f(T param);
f({1, 2, 3, 4}); // compile error

template<typename T> void f(initializer_list<T> param);
f({1, 2, 3, 4}); // OK
```

**在C++14中，auto可以用在函数定义的返回值处，C++11则必须使用尾置返回值**。

- 尽管使用auto，编译器实际进行的是模板类型推导。

<br>

<br>

## decltype

- 绝大多数情况下，decltype会得出的变量或表达式的类型而不做任何修改
- 保留引用或const属性
- 对于类型为T的表达式，除非它有一个名字，否则decltype总是得出T&
- c++14支持decltype(auto)

为了查看decltype的推导类型，可以声明一个模板类但不去实现它；实例化时为模板参数T传入decltype表达式，根据引发的编译错误来查看decltype的推导结果。

```c++
template<typename T> class TD; // don't define it
TD<decltype(express)> Type; // thorw compile error info, which containing the result of decltype
```