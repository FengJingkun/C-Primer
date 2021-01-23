## std::declval

**Prototype**

```c++
#include <utility>

template<class T>
typename std::add_value_reference<T>::type declval() noexcept;
```

declval通常用于模板中，与decltype结合使用，可以在不构造对象的情况下，推断模板参数T的某个成员的类型，即使T没有构造函数。

**Example**

```c++
struct Foo {
    Foo() = delete;
    int func() { return 0; }
};

decltype(Foo().func()) n1; // error, no default constructor
decltype(declval<Foo>().func()) n2; // n2 is a int
```

**Practice**

```c++
typedef unordered_map<string, int> object;

// var is string
decltype(std::declval<object>().begin()->first) var;
```

**declval不能用于求值语境**

<br>

## std::is_constructible

**Prototype**

```c++
#include <type_traits>

template<class T, typename... Args>
struct is_constructible;
```

若T是对象或引用类型，且变量定义T obj(declval<Args>()...)为良构，则提供等于**true**的成员常量**value**，即：

**如果类型T的对象可以通过Args来构造，即T拥有一个参数为Args的构造函数，那么该模板的value为true。**

**Example**

```c++
class Foo {
public:
    Foo() {};
    Foo(int) {};
};

// value is true
is_constructible<Foo, int>::value;

// value is false
is_constructible<Foo, string>::value;
```

**Practice**

```c++
typedef unordered_map<string, int> object;

// value is true
is_constructible<string, decltype(string, declval<object>().begin()->first)>::value;

// value is false
is_constructible<string, decltype(int, declval<object>().begin()->first)::value;
```

<br>

## std::enable_if

**Prototype**

```c++
#include <type_traits>

template<bool B, class T = void>
struct enable_if;

template<class T>
struct enable_if<true, T> {
    using type = T;
}

template<class T>
struct enable_if<false, T> { }
```

若模板参数B为true，则enable_if拥有一个等同于T的公开成员，该成员被typedef为type；若B为false，则没有type成员。

**成员变量type完全依赖于模板参数B的值，若B为true，则type为类型T的成员；若B为false，则type未定义。**

- **类型偏特化**

  根据实际传入的模板参数类型来选择使用不同的模板，或者在编译模板时验证某些参数的特性

  ```c++
template<typename T, typename Enable = void>
  struct check;

  template<typename T>
struct check<T, typename std::enable_if<T::val>::type> {
  	static constexpr bool value = T::value;  
}; // 当且仅当类型T的成员变量val为true时，check模板才能被正确编译
  ```

- **控制函数的返回类型**

  对于模板函数，有时需要根据不同的模板参数返回不同类型的值。**tuple获取第k个元素的get函数是一个典型例子。**

  ```c++
template<size_t k, class T, typename... Ts>
  typename std::enable_if<k == 0>::type get(tuple<T, Ts...> &t) {
    return t.tail;
  } // 返回值类型用typename修饰，表示enable&lt;&gt;::type是一个类型而不是成员
  
  template<size_t k, class T, typename... Ts>
  typename std::enable_if<k != 0>::type get(tuple<T, Ts...> &t) {
      tuple<Ts...> &base = t;
      return get<k - 1>(base);
  } // 若k = 0，enable_if::type未定义，编译不通过
  ```

- **校验函数模板参数类型**

  允许特定类型调用某些模板

  ```c++
  template<typename T>
  typename std::enable_if<std::is_integral<T>::value, bool>::type is_odd(T t) {
      return static_cast<bool>(t % 2);
  } // 通过返回值类型校验模板参数T是否为int
  
  template<typename T, typename = typename std::enable_if<std::is_integral<T>::value>::type>
  bool is_even(T t) {
      return !is_odd(t);
  } // 通过默认模板参数来校验模板参数T是否为int
  ```

**Practice**

```c++
typedef pair<char, int> couple;

template<class M, 
		typename = enable_if<
                    is_constructible<char,decltype(declval<M>().first)>::value &&
                    is_constructible<int,decltype(declval<M>().second)>::value>::type>
Json(const M& m) {
    pair<char, int> obj(m.first, m.second);
}
```