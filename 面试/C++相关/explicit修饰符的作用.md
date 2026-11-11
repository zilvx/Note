`explicit` 是 C++ 中的一个关键字，主要用于**修饰类的构造函数**，防止隐式类型转换。它的核心作用是**避免编译器在你不期望的情况下自动进行类型转换**。

---

## 一、核心作用

### 1. 阻止隐式转换（最常见）

**不加 `explicit` 时：**
```cpp
class MyInt {
public:
    MyInt(int n) { /* ... */ }  // 普通构造函数
};
void func(MyInt obj) { /* ... */ }
func(42);   // ✅ 隐式转换：int → MyInt（编译器自动调用构造函数）
MyInt m = 10; // ✅ 隐式转换
```

**加了 `explicit` 后：**
```cpp
class MyInt {
public:
    explicit MyInt(int n) { /* ... */ }  // 显式构造函数
};
void func(MyInt obj) { /* ... */ }
func(42);   // ❌ 编译错误：不能隐式转换
func(MyInt(42)); // ✅ 必须显式构造
MyInt m = 10;    // ❌ 编译错误：不能隐式转换
MyInt m(10);     // ✅ 必须用直接初始化
```

---

### 2. 阻止拷贝初始化中的隐式转换
```cpp
class String {
public:
    explicit String(const char* s) { /* ... */ }
};
String s1 = "hello";  // ❌ 错误：explicit 禁止隐式转换
String s2("hello");   // ✅ 正确：直接初始化
```

---

## 二、什么时候该用 `explicit`？

### ✅ 推荐使用场景

|场景|原因|
|---|---|
|**单参数构造函数**|最容易发生意外的隐式转换|
|**多参数构造函数（C++11起）**|防止列表初始化时的意外转换|
|**转换运算符（C++11起）**|`explicit operator bool()` 防止误用|

---

### 3. 多参数构造函数的 `explicit`（C++11 起）
```cpp
class Point {
public:
    explicit Point(int x, int y) : mx(x), my(y) {}
};
Point p1 = {1, 2};   // ❌ 错误：explicit 阻止列表初始化
Point p2(1, 2);      // ✅ 正确
Point p3 = Point(1,2); // ✅ 正确
```

---

## 三、实际应用案例

### ❌ 不加 explicit 的坑（经典错误）
```cpp
class Vector {
public:
    Vector(int size) : data(new int[size]) {}  // 未加 explicit
    ~Vector() { delete[] data; }
private:
    int* data;
};
void process(Vector v) { /* ... */ }
process(10);  // ⚠️ 本意是传 vector，结果编译器用 int 10 构造了一个 10 大小的 vector
```

### ✅ 加上 explicit 后，编译报错，及时发现问题
```cpp
class Vector {
public:
    explicit Vector(int size) : data(new int[size]) {}
    // ...
};
process(10);  // ❌ 编译错误，提醒你应该显式调用
process(Vector(10)); // ✅ 明确你的意图
```

---

## 四、`explicit` 用于转换运算符（C++11）
```cpp
class Status {
public:
    explicit operator bool() const { return is_ok; }  // 防止 if(status) 误用
private:
    bool is_ok = true;
};
Status s;
if (s) { /* ❌ 错误：explicit 阻止了 bool 转换 */ }
if (static_cast<bool>(s)) { /* ✅ 必须显式转换 */ }
```

---

## 五、总结：一个表格看懂

|情况|不加 `explicit`|加 `explicit`|
|---|---|---|
|`func(42)`|✅ 隐式转换|❌ 编译错误|
|`MyInt m = 10;`|✅ 隐式转换|❌ 编译错误|
|`MyInt m(10);`|✅ 正常|✅ 正常|
|`MyInt m = {10};`|✅ (C++11)|❌ 编译错误|
|`if (obj)`（转换运算符）|✅ 隐式转换|❌ 编译错误|

---

### 最佳实践建议

1. **除极少数情况外，单参数构造函数一律加 `explicit`**
    
2. **多参数构造函数，如果不想支持列表初始化，也加 `explicit`**
    
3. **`explicit operator bool()` 几乎是标准写法**，防止 `if(obj)` 误判
    

---