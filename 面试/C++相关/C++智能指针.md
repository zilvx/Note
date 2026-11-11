C++ 智能指针是面试必考且实际开发中不可或缺的工具。它用 RAII 机制将裸指针的生命周期与对象绑定，自动释放内存，极大减少了内存泄漏和悬垂指针问题。

以下从原理、用法到最佳实践，系统梳理 C++11 起提供的三种核心智能指针。

---

### 一、为什么需要智能指针

手动 `new` / `delete` 极易出错：

- 忘记 `delete` 导致内存泄漏
    
- 提前 `return`、抛出异常导致 `delete` 被跳过
    
- 多次 `delete` 或对已释放内存解引用（悬垂指针）
    
- 所有权不清晰，难以维护
    

智能指针在构造时获取资源，在析构时自动释放资源，将资源生命周期绑定在栈对象上，即使异常退出也能保证释放，符合 RAII 原则。

---

### 二、`std::unique_ptr`（独占所有权）

#### 特征

- **独占**：同一时刻只有一个 `unique_ptr` 拥有对象。
    
- **不可拷贝**，只能**移动**，移动后源指针变为 `nullptr`。
    
- **零开销**（通常与裸指针相同大小，默认删除器无额外存储）。
    

#### 基本用法

```cpp

#include <memory>
std::unique_ptr<int> p1(new int(42));
// 更推荐 C++14 的 make_unique
auto p2 = std::make_unique<int>(42);
// p1 不可拷贝，只能移动
std::unique_ptr<int> p3 = std::move(p1);  // p1 变为空
// 直接访问管理对象
*p3 = 10;
std::cout << *p3 << '\n';
// 释放所有权但不释放对象
int* raw = p3.release();
delete raw;  // 需手动释放
```

#### 自定义删除器

当资源需要特定的清理方式（如 `FILE*`）时，可传入删除器。注意，删除器类型会成为模板参数的一部分（除非使用 `std::function` 或 lambda 无捕获的删除器可消除额外大小）。

```cpp

std::unique_ptr<FILE, decltype(&fclose)> fp(fopen("test.txt", "r"), fclose);
// 或使用无捕获 lambda（无额外存储开销）
auto fclose_wrapper = [](FILE* f) { if(f) fclose(f); };
std::unique_ptr<FILE, decltype(fclose_wrapper)> fp2(fopen("test.txt", "r"), fclose_wrapper);
```

#### 管理数组

`unique_ptr` 有对数组的特化版本，自动调用 `delete[]`。

```cpp

auto arr = std::make_unique<int[]>(10);
arr[0] = 1;
```

---

### 三、`std::shared_ptr`（共享所有权）

#### 特征

- **共享**：通过引用计数记录有多少个 `shared_ptr` 指向同一对象，最后一个 `shared_ptr` 销毁时释放对象。
    
- **额外开销**：需要分配**控制块**存储引用计数、弱引用计数和删除器等，`shared_ptr` 本身大小通常是裸指针的两倍（一个指向对象，一个指向控制块）。
    

#### 基本用法

```cpp

auto sp1 = std::make_shared<int>(42);
std::shared_ptr<int> sp2 = sp1;  // 引用计数 +1
sp1.use_count();                 // 返回 2
// 解引用
*sp2 = 100;
```

#### 控制块与 `make_shared`

- `std::make_shared<T>(args...)` 一次分配同时创建对象和控制块所在内存，减少分配次数、提高缓存局部性、异常安全。
    
- 直接使用 `new` 构造 `shared_ptr` 会先分配对象，再分配控制块，两次分配且可能因异常造成泄漏。
    

#### 自定义删除器

与 `unique_ptr` 不同，`shared_ptr` 的删除器**不**影响类型签名，因为删除器存储在控制块中。

```cpp

auto deleter = [](int* p) { delete p; };
std::shared_ptr<int> sp(new int(10), deleter);
// 使用 make_shared 无法指定删除器，需用构造函数

#### 引用计数的线程安全

- **引用计数本身是原子操作**：拷贝、移动、析构 `shared_ptr` 在多线程中安全。
    
- **管理对象本身不是线程安全**：多个线程读写同一对象仍需同步。
    
```

#### 循环引用问题

两个对象互相持有对方的 `shared_ptr`，引用计数永远无法归零，导致内存泄漏。

```cpp

struct B;
struct A {
    std::shared_ptr<B> bptr;
    ~A() { std::cout << "A destroyed\n"; }
};
struct B {
    std::shared_ptr<A> aptr;  // 若改为 weak_ptr 可打破循环
    ~B() { std::cout << "B destroyed\n"; }
};
int main() {
    auto a = std::make_shared<A>();
    auto b = std::make_shared<B>();
    a->bptr = b;
    b->aptr = a;   // 循环引用，A 和 B 的析构函数永远不会被调用
}
```

解决方法：将其中一个改为 `weak_ptr`。

---

### 四、`std::weak_ptr`（弱引用）

#### 特征

- **不增加引用计数**，仅观察 `shared_ptr` 管理的对象。
    
- 必须从 `shared_ptr` 或另一个 `weak_ptr` 构造。
    
- 不能直接解引用，需调用 `lock()` 获取临时 `shared_ptr` 检查对象是否仍存活。
    

#### 主要用途

1. **打破循环引用**：如上述例子，将 `B::aptr` 改为 `weak_ptr<A>`。
    
2. **观察者 / 缓存**：持有一个弱引用，需要时 `lock()`，若对象已释放则重新创建或跳过。
    

#### 示例

```cpp

std::shared_ptr<int> sp = std::make_shared<int>(42);
std::weak_ptr<int> wp = sp;
if (auto tmp = wp.lock()) {   // 如果对象仍然存在，tmp 是一个 shared_ptr
    std::cout << *tmp << '\n';
} else {
    std::cout << "object has been destroyed\n";
}
sp.reset();                    // 释放对象
if (wp.expired()) {
    std::cout << "weak_ptr expired\n";
}
```

---

### 五、已废弃的 `std::auto_ptr`（C++98）

- `auto_ptr` 采用“所有权转移”语义，但拷贝和赋值时会将源指针置空，破坏性移动，不符合常规期望。
    
- 无法存储在容器中（因为容器要求元素可拷贝且行为一致）。
    
- C++11 中已被 `unique_ptr` 彻底取代，面试中几乎不会考察，但知道其存在和缺陷即可。
    

---

### 六、智能指针的转换

- `std::static_pointer_cast<T>(sp)`
    
- `std::dynamic_pointer_cast<T>(sp)`
    
- `std::const_pointer_cast<T>(sp)`
    
- `std::reinterpret_pointer_cast<T>(sp)`（C++17）
    

它们返回正确类型的 `shared_ptr`，内部会复制引用计数，保证类型转换后仍安全共享对象。

---

### 七、使用建议与最佳实践

1. **优先使用栈对象**，无需指针更安全。
    
2. **需要指针时，默认用 `unique_ptr`**：轻量、独占、可轻松转为 `shared_ptr`。
    
3. **只有确实需要共享所有权时才用 `shared_ptr`**：引用计数有开销，且可能引起循环引用。
    
4. **创建智能指针优先用 `make_unique` / `make_shared`**：安全、高效。
    
5. **避免裸指针与智能指针混用**：尤其不要用同一裸指针初始化多个 `shared_ptr`，会多次释放。
    
    cpp
    
    int* raw = new int(10);
    std::shared_ptr<int> sp1(raw);
    std::shared_ptr<int> sp2(raw);  // 错误！两个独立引用计数，分别 delete
    
6. **自定义删除器时注意**：`unique_ptr` 的删除器是模板参数一部分，可能增大体积；`shared_ptr` 的删除器存储在控制块，更灵活。
    
7. **警惕循环引用**：在涉及双向关联的场景主动使用 `weak_ptr`。
    
8. **`shared_ptr` 与 `this` 的问题**：如果类需要从 `this` 生成 `shared_ptr`，应继承 `std::enable_shared_from_this<T>`，并使用 `shared_from_this()`，而非用裸 `this` 构造新的 `shared_ptr`，否则同样会创建独立控制块。
    
    cpp
    
    class Good : public std::enable_shared_from_this<Good> {
    public:
        std::shared_ptr<Good> getShared() {
            return shared_from_this();
        }
    };
    

---

### 八、性能考量

- `unique_ptr`：与裸指针同等性能，无额外内存或运行开销（前提是无状态删除器）。
    
- `shared_ptr`：
    
    - 内存：额外分配控制块（原子计数器、删除器等），`make_shared` 可合并分配。
        
    - 时间：拷贝/析构时原子增减引用计数，在多线程下有一定竞争开销。
        
- `weak_ptr`：使用 `lock()` 会原子地检查引用计数并增加强计数，开销与 `shared_ptr` 相似。
    

在性能热点路径，可考虑传递 `const shared_ptr<T>&` 避免不必要的引用计数操作；但通常保持值传递即可，`shared_ptr` 的拷贝比想象中轻量。

---

掌握智能指针的内部机制与使用原则，不仅能写出安全优雅的代码，更是衡量 C++ 开发者现代语言掌握程度的重要标尺。建议在面试前动手实现一个简化的 `shared_ptr`，加深对控制块和 RAII 的理解。