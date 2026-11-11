## 一、核心概念

`= default` 表示让编译器**自动生成**默认版本的构造函数/赋值函数，而 `explicit` 控制**隐式转换**。两者可以组合使用，但行为取决于**函数类型**。

---

## 二、`explicit` + `= default` 的三种场景

### 1. 默认构造函数（`explicit default constructor`）

```cpp
class MyClass {
public:
    explicit MyClass() = default;  // 显式默认构造函数
};
// 使用：
MyClass c1;        // ✅ 正常（直接初始化）
MyClass c2 = {};   // ❌ 错误：explicit 阻止隐式转换（列表初始化）
MyClass c3 = MyClass(); // ✅ 可以（显式构造）
```


**作用**：阻止 `MyClass obj = {};` 这种隐式列表初始化，要求必须显式构造。

---

### 2. 拷贝构造函数（`explicit copy constructor`）

```cpp
class MyClass {
public:
    explicit MyClass(const MyClass&) = default;  // 显式拷贝构造函数
};
MyClass a;
MyClass b = a;   // ❌ 错误：拷贝初始化被 explicit 阻止
MyClass c(a);    // ✅ 正确：直接初始化
```

**作用**：禁止值传递（按值传参）和拷贝初始化，强制使用引用传递。

---

### 3. 移动构造函数（`explicit move constructor`）
```cpp
class MyClass {
public:
    explicit MyClass(MyClass&&) = default;  // 显式移动构造函数
};
MyClass a;
MyClass b = std::move(a);   // ❌ 错误：移动初始化被 explicit 阻止
MyClass c(std::move(a));    // ✅ 正确：直接初始化
```

---

## 三、结合 `= default` 的转换运算符（C++11 起）
```cpp
class Flag {
public:
    explicit operator bool() = default;  // 显式 bool 转换（C++20 支持）
};
Flag f;
if (f) { /* ❌ 错误：explicit 阻止隐式转换 */ }
if (static_cast<bool>(f)) { /* ✅ 必须显式转换 */ }
```

> **注意**：`explicit operator bool() = default` 在 C++20 才正式支持，早期版本需手动实现。

---

## 四、实际应用场景

### 场景 1：单例模式中的显式默认构造
```cpp
class Singleton {
public:
    static Singleton& getInstance() {
        static Singleton instance;
        return instance;
    }
    
    explicit Singleton() = default;  // 防止 Singleton s = {}; 的意外构造
    
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
};
```

---

### 场景 2：RAII 资源管理类
```cpp
class FileHandle {
public:
    explicit FileHandle() = default;      // 防止隐式创建空句柄
    explicit FileHandle(const char* path) : file_(fopen(path, "r")) {}
    
    // 拷贝构造函数显式化，防止意外拷贝
    explicit FileHandle(const FileHandle&) = default;
    
    ~FileHandle() { if (file_) fclose(file_); }
private:
    FILE* file_ = nullptr;
};
// 使用：
FileHandle f1;              // ✅ 显式构造（空句柄）
FileHandle f2 = FileHandle(); // ✅ 也可以
FileHandle f3 = f1;         // ❌ 错误：explicit 拷贝构造阻止隐式拷贝
FileHandle f4(f1);          // ✅ 直接拷贝
```

---

### 场景 3：防止值传递的性能陷阱
```cpp
class BigBuffer {
public:
    explicit BigBuffer() = default;
    
    // 拷贝构造函数加 explicit，强制用引用传递
    explicit BigBuffer(const BigBuffer& other) = default;
    
    // 移动构造函数加 explicit
    explicit BigBuffer(BigBuffer&& other) = default;
};
void process(BigBuffer buf) { /* 按值传递会拷贝 */ }
BigBuffer buffer;
process(buffer);          // ❌ 错误：explicit 拷贝构造阻止按值传递
process(std::move(buffer)); // ❌ 错误：explicit 移动构造阻止按值传递
process(BigBuffer(buffer)); // ✅ 必须显式构造拷贝
```

---

## 五、`explicit` 与 `= default` 的搭配规则总结

|构造函数类型|加 `explicit` 的效果|实际行为|
|---|---|---|
|**默认构造** `explicit MyClass() = default`|阻止 `MyClass obj = {};`|仅支持直接初始化 `MyClass obj;` 或 `MyClass obj{}`|
|**拷贝构造** `explicit MyClass(const MyClass&) = default`|阻止拷贝初始化 `MyClass b = a;`|支持直接初始化 `MyClass b(a);`|
|**移动构造** `explicit MyClass(MyClass&&) = default`|阻止移动初始化 `MyClass b = std::move(a);`|支持直接初始化 `MyClass b(std::move(a));`|
|**转换运算符** `explicit operator bool() = default`|阻止隐式 bool 转换|必须显式 `static_cast<bool>(obj)`|

---

## 六、`explicit` + `= default` 的进化（C++20 新语法）

**C++20 开始**，`explicit` 可以指定**条件显式**：
```cpp
template<typename T>
class Wrapper {
public:
    explicit(!std::is_convertible_v<T, int>) Wrapper(T value) : data_(value) {}
    // 如果 T 可转 int，则 non-explicit；否则 explicit
private:
    T data_;
};
// 使用：
Wrapper<int> w1 = 42;    // ✅ int → int，non-explicit，允许
Wrapper<double> w2 = 3.14; // ❌ double → int，explicit，阻止
Wrapper<double> w3(3.14);  // ✅ 显式构造
```

> 这种写法在 `= default` 构造中同样适用。

---

## 七、最佳实践建议

|场景|推荐写法|
|---|---|
|类的默认构造|除非需要列表初始化，否则加 `explicit`|
|资源管理类|拷贝构造加 `explicit`，防止意外拷贝|
|大对象类|拷贝/移动构造加 `explicit`，强制显式传递|
|转换运算符|一律加 `explicit`|
|模板类|用 `explicit(!条件)` 实现条件显式（C++20）|
