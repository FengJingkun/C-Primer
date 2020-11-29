## 资源管理

### 以对象管理资源(RAII )

```c++
class Investment {}; // base class
Investment* createInvestment(); // 返回指针,指向Investment继承体系内的动态分配对象

void f() {
	Investment* pInv = createInvestment();
    ... ; // 若抛出异常或提前return,则pInv所指对象不会被释放,造成内存泄漏
    delete pInv;
}
```

createInvestment返回的资源位于heap上，为了确保这些资源总能被释放，需要将其放入一个管理对象，然后通过管理对象的析构函数来自动释放资源。即：在管理对象的析构函数中调用delete。

具体地，使用智能指针对象来管理资源。

```c++
void f() {
    // 函数返回时调用auto_ptr的析构函数,后者会调用delete释放investment资源
    auto_ptr<Investment> pInv(createInvestment());
}
```

- **获得资源后应立即放入管理对象(wrapper)内，即RAII**

  每一笔资源在获得后应立即初始化一个管理对象来存放这笔资源

- **通过管理对象的析构函数来确保资源被释放**

由于auto_ptr对象被销毁时会自动删除它存放的资源，因此将多个auto_ptr指向同一资源对象是非常危险的。为了避免该问题，管理对象的copy构造函数或赋值操作符应该在被调用时将原指针置空。即确保在任何时刻，受auto_ptr管理的资源不会存在多个auto_ptr同时指向它。

然而，STL容器要求其元素需要具有正常的复制行为，因此auto_ptr就无法使用了。替代方案即**引用计数指针shared_ptr**。

**由于智能指针的析构函数内调用的是delete而不是delete[]，因此无法在动态分配的array上使用智能指针。**

<br>

- **为了防止内存泄漏，应使用RAII对象来管理资源，它们在构造函数中获得资源，并在析构函数中释放资源。**
- **auto_ptr和shared_ptr是两个经常被使用的RAII类，但它们不能用于动态分配的array。**

<br>

<br>

### 资源管理类中小心copying行为

对于非heap类资源，则不太适合通过智能指针来管理它们，比如Mutex类型的互斥锁对象。

```c++
void lock(Mutex* pm);
void unlock(Mutex* pm);
```

为了确保锁在使用完成后能被释放，需要应用RAII原则，使用一个管理类来管理锁资源。

```c++
class Lock {
public:
    explicit Lock(Mutex* pm): mutexPtr(pm) { // 禁止隐含类型转换
        lock(mutexPtr); // 资源在构造期间获得
    }
    ~Lock() {
        unlock(mutexPtr); // 资源在析构期间释放
    }
private:
    Mutex* mutexPtr;
};
```

**当RAII类发生copy行为时，应该在以下可能中做选择：**

- **禁止复制**

  ```c++
  Lock(const Mutex* pm) = delete;
  Lock& operator=(const Mutex* pm) = delete;
  ```

- **对底层资源进行引用计数**

  保有资源，由最后一个使用资源的类负责销毁。具体可通过在RAII类中内含一个shared_ptr成员。

  shared_ptr在初始化时可以指定一个函数或仿函数来作为删除器。由于Mutex只需要释放而不需要删除，因此应将删除器指定为unlock。

  ```c++
  class Lock{
  public:
      explicit Lock(Mutex* pm): mutexPtr(pm, unlock) {
          lock(mutexPtr.get());
      }
      // 无需定义析构函数, 因为当引用计数为0时mutexPtr会自动调用unlock
  private:
      shared_ptr<Mutex> mutexPtr;
  };
  ```

- **复制底层资源**

  有时需要拥有一份资源的多个副本，使用RAII类确保当副本不再需要时被释放。特别地，复制RAII对象时，需要使用**深拷贝**。

- **转移底部资源的拥有权**

  确保一份资源在任何时刻只有一个RAII对象指向它。复制时资源所有权从一个RAII对象转移到另一个。

<br>

- **资源的copying行为决定RAII的copying行为**
- **RAII类常见的copying行为是：删除copying，使用引用计数**

<br>

<br>

### 在RAII类中提供对资源的原始访问

 有时需要对原生资源进行访问，这就要求RAII类能够提供对资源的原始访问

```c++
shared_ptr<Investment> pInv(createInvestment()); // RAII类管理资源
void process(Investment* pi); // 资源处理函数需要使用原始资源
process(pInv.get()); // RAII类提供资源的原始访问方式, 显式转换
```

 显式转换

```c++
class Investment {
public:
    bool isTaxFree() const;
};
Investment* createInvestment(); // 资源factory函数
shared_ptr<Investment> pInv(createInvestment()); // RAII
// 两种显式转换方式
bool taxable1 = pInv->isTaxFree();
bool tabable2 = !((*pInv).isTaxFree());
```

隐式转换

```c++
FontHandle getFont(); // Font资源factory函数
void releaseFont(FontHandle fh);

class Font { // RAII
public:
    explicit Font(FontHandle fh): handle(fh) {}
    ~Font() { releaseFont(handle); }
private:
    FontHandle handle;
};
```

假设有大量API，它们接收FontHandle作为参数，那么将Font转为FontHandle将是一种很频繁的请求，此时Font可以提供一个显示转换函数：

```c++
FontHandle Font::get() const {
    return this->handle;
}

// 用户使用
void changeFontSize(FontHandle f, int newSize);
Font f(getFont()); // 使用RAII管理资源
int newFontSize;
changeFontSize(f.get(), newFontSize); // get访问原始资源
```

也可以提供隐式转换函数

```c++
class Font {
public:
    // 隐式转换操作符, 将Font转换为FontHandle
    operator FontHandle() const { return this->handle; }
};

changeFontSize(f, newFontSize);
```

但可能会增加错误发生的几率：

```c++
Font f1(getFont());
FontHandle f2 = f1; // 原意是拷贝一个Font对象, 但却将f1隐式转换为FontHandle再复制！
```

此时若f1被析构，则f2将处于空悬状态。

- **API通常要求访问原始资源，因此RAII类应提供资源的原始访问方式**
- **显式转换更安全，但隐式转换更方便**

<br>

<br>

### 成对使用new和delete并采取相同形式

单一对象的内存布局与数组对象的内存布局不同。数组所用内存通常还包括数组大小的记录，单一对象的内存则没有这笔记录。

 ``` c++
typedef string AddressLines[4]; // AddressLines类型为一个长度为4的数组, 数组元素为string
string* pal = new AddressLines;
delete[] pal; // 尽管未使用new [], 但仍应使用delete []
 ```

- **new中使用[]，那么delete也应该使用[]**
- **尽量不对数组做typedef动作**

<br>

<br>

### 以独立语句将newed对象置入智能指针

```c++
int priority();
void processWidget(shared_ptr<Widget> pw, int priority);

processWidget(shared_ptr<Widget>(new Widget), priority());
```

进入processWidget前：

- 调用priority
- 执行new Widget
- 调用shared_ptr构造函数

C++编译器并不会以特定次序完成上述三种操作，唯一可以确定的是new Widget一定执行于shared_ptr构造函数调用前。

若使用以下次序执行：

- 执行new Widget
- 调用priority
- 调用shared_ptr构造函数

此时，**若priority函数产生异常，那么new Widget返回的指针将会丢失，因为它还未放入shared_ptr中，这无疑会导致资源泄露**。

使用分离语句则可以完全避免上述问题。

```c++
shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority);
```

- **以独立语句将newed对象存储于智能指针内，由此可避免异常抛出导致的内存泄漏。**