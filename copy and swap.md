### Copy and Swap

copy and swap主要用来实现拷贝赋值操作符，即：

利用copy constructor生成一个临时拷贝对象，然后使用swap函数将临时对象与当前对象的旧数据进行交换。然后临时对象被析构，旧数据消失，当前对象就拥有了新数据的拷贝。

因此需要提供以下三个函数：

- **拷贝构造函数**
- **析构函数**
- **swap函数**

<br>

### Why use?

假设现在需要在一个类中管理动态数组：

```c++
class Dumb_Array {
public:
    Dumb_Array(size_t size = 0): mSize(size), mArray(nullptr) {
        if (mSize != 0)
            mArray = new int[mSize];
    }

    Dumb_Array(const Dumb_Array& other): mSize(other.mSize){
        mArray = new int[mSize];
        std::copy(other.mArray, other.mArray + mSize, mArray);
    }

    ~Dumb_Array() {
        delete[] mArray;
    }

    Dumb_Array& operator=(const Dumb_Array& rhs);
private:
    size_t mSize;
    int* mArray;
};
```

以下是赋值运算符函数的一个实现:

```c++
Dumb_Array& Dumb_Array::operator=(const Dumb_Array &rhs) {
    if (this != &rhs) {
        delete[] mArray;
        mSize = rhs.mSize;
        mArray = mSize? new int[mSize]: nullptr;
        std::copy(rhs.mArray, rhs.mArray + mSize, mArray);
    }
    return *this;
}
```

上述实现存在以下问题：

1. **需要进行自我赋值判断**

2. **不满足异常安全**

   如果new int[mSize]失败，那么当前对象的mArray就丢失了

针对问题2，一个异常安全的实现为：

```c++
Dumb_Array& Dumb_Array::operator=(const Dumb_Array &rhs) {
    if (this != &rhs) {
        size_t tmpSize = rhs.mSize;
        int* tmpArray = tmpSize? new int[tmpSize]: nullptr;
        std::copy(rhs.mArray, rhs.mArray + tmpSize, tmpArray);

        delete[] mArray;
        mSize = rhs.mSize;
        mArray = tmpArray;
    }
    return *this;
}
```

然后，这又会导致代码冗余问题。可以看出核心代码实际上只有两行，即**分配空间和拷贝**。如果是复杂的资源管理，那么代码将急剧膨胀。

**copy and swap可以用来解决该问题。**

<br>

### Solve

首先需要添加swap函数：

```c++
// 可以定义成友元函数，也可以定义成类成员函数
friend void swap(Dumb_Array& first, Dumb_Array& second) {
    std::swap(first.mSize, second.mSize);
    std::swap(first.mArray, second.mArray);
}
```

赋值运算函数的实现：

```c++
Dumb_Array& Dumb_Array::operator=(const Dumb_Array rhs) { // pass by value
    swap(*this, rhs);
    return *this;
}
```

注意这里使用了pass by value，在传递参数时由编译器调用**拷贝构造函数**来生成一个临时对象。一旦进入函数体，就表明新数据的分配和拷贝都已经完成了，接下来只需要调用**swap**将新数据放入当前对象即可。

这提供了强烈的异常安全保证：**如果拷贝失败，那么就不会进入函数体，当前对象的内容也不会改变。**