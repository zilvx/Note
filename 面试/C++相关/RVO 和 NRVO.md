## 1. 什么是 RVO 和 NRVO？

**RVO（Return Value Optimization，返回值优化）** 和 **NRVO（Named Return Value Optimization，具名返回值优化）** 是C++编译器对**函数返回对象**时进行的**强制或允许的拷贝消除优化**。

> **一句话核心：** 它们让函数返回大对象时，**不触发拷贝构造函数和析构函数**，直接在调用者的内存位置上构造对象，达到"零拷贝"的效果。

---

## 2. 先看问题：没有优化时会发生什么？
```cpp
class BigObject {
public:
    BigObject() { std::cout << "构造\n"; }
    BigObject(const BigObject&) { std::cout << "拷贝构造\n"; }
    ~BigObject() { std::cout << "析构\n"; }
};
BigObject createObject() {
    BigObject local;  // 构造1
    return local;     // 理论上会拷贝构造一个临时对象
}  // local 析构
int main() {
    BigObject obj = createObject();  // 理论上会从临时对象拷贝构造 obj
    return 0;
}

```

**理论上（无优化）的执行顺序：**

1. `createObject()` 中构造 `local`
    
2. `return local` 时，拷贝构造**临时对象**（返回值）
    
3. `local` 析构
    
4. `main` 中 `obj` 从临时对象**拷贝构造**
    
5. 临时对象析构
    
6. `main` 结束，`obj` 析构
    

**总共：1次构造 + 2次拷贝构造 + 3次析构**，对大型对象来说性能灾难。

---

## 3. RVO（返回值优化）：消除临时对象

**RVO** 针对的是返回**临时对象**（无名对象）的场景：
```cpp
BigObject createObject() {
    return BigObject();  // 返回一个临时对象（无名）
}  // RVO 优化生效
int main() {
    BigObject obj = createObject();
    return 0;
}

```

**RVO 优化后的实际执行：**

- 编译器在 `main` 的栈帧中为 `obj` 预留空间。
    
- 调用 `createObject()` 时，编译器**把 `obj` 的地址作为隐藏参数**传给函数。
    
- 函数内部直接在 `obj` 的地址上构造临时对象。
    
- **没有任何拷贝构造发生！**
    

**执行次数：** 1次构造 + 0次拷贝 + 1次析构

---

## 4. NRVO（具名返回值优化）：消除具名对象的拷贝

**NRVO** 针对的是返回**具名对象**的场景（你之前可能疑惑过"返回局部变量会不会拷贝"，这里就是答案）：
```cpp
BigObject createObject() {
    BigObject local;   // 具名对象
    // ... 对 local 做一些操作
    return local;      // 返回具名对象
}  // NRVO 优化生效（大部分编译器都支持）
int main() {
    BigObject obj = createObject();
    return 0;
}
```

**NRVO 优化后的实际执行：**

- 同样，编译器在 `main` 中为 `obj` 预留空间，地址传给 `createObject()`。
    
- `createObject()` 中的 `local` **直接就是 `obj` 的别名**（在 `obj` 的内存上构造）。
    
- `return local` 时，编译器知道 `local` 就是返回值，**无需任何拷贝**。
    

**执行次数：** 1次构造 + 0次拷贝 + 1次析构

---

## 5. RVO vs NRVO：区别对比

|维度|RVO|NRVO|
|---|---|---|
|**返回对象类型**|**无名临时对象**（如 `return T();`）|**具名局部对象**（如 `return local;`）|
|**编译器支持**|**C++17 标准强制要求**|编译器**允许但不保证**，由编译器优化决定|
|**可靠性**|100% 保证生效（C++17起）|依赖编译器和优化级别（`-O2` 通常开启）|
|**实现难度**|简单，一眼看出返回值是临时对象|复杂，需要数据流分析，确保返回的总是同一个对象|

---

## 6. C++17 的重大变化：强制 RVO

在 **C++17** 之前，RVO 和 NRVO 都是"允许的优化"，即使编译器没做优化，代码也必须可编译。

但在 **C++17** 之后，**RVO 被强制标准化**：
```cpp
struct NonCopyable {
    NonCopyable() = default;
    NonCopyable(const NonCopyable&) = delete;  // 禁止拷贝
};
NonCopyable create() {
    return NonCopyable();  // C++17 强制 RVO，编译通过
}
int main() {
    NonCopyable obj = create();  // ✅ 编译通过，因为 RVO 保证不拷贝
}
```

**注意：** NRVO 依然不是强制的。如果一个类型不可拷贝，但函数使用 NRVO，编译器可能拒绝编译（如果优化失败）。

---

## 7. 什么时候 RVO/NRVO 会失效？（避免踩坑）

以下情况**会阻止**拷贝消除优化：

### ① 返回条件分支中不同对象（NRVO 失效）
```cpp
BigObject createObject(bool flag) {
    BigObject a, b;
    if (flag) 
        return a;   // 返回 a
    else
        return b;   // 返回 b
}  // NRVO 可能失效，编译器不知道返回哪个对象，无法在同一地址构造
```

### ② 返回函数参数（传入的对象）
```cpp
BigObject createObject(BigObject param) {
    return param;  // 参数是外部传入的，不在当前函数栈帧上，无法优化
}
```

### ③ 返回全局或静态对象
```cpp
BigObject createObject() {
    static BigObject global;
    return global;  // 静态对象生命周期不随函数结束，不能构造在调用者栈上
}
```

---

## 8. 实战建议（现代C++写法）

### ✅ 推荐写法（让优化更容易生效）
```cpp
// 1. 尽量直接返回临时对象（RVO 保证）
BigObject create() {
    return BigObject{...};
}
// 2. 如果需要构造复杂对象，使用 NRVO 友好写法
BigObject create() {
    BigObject result;
    result.setup(...);
    // ... 各种初始化
    return result;  // NRVO 通常生效
}
// 3. 如果 NRVO 可能失败，使用移动语义兜底（C++11）
BigObject create() {
    BigObject a, b;
    if (flag)
        return a;   // 即使 NRVO 失败，也会调用移动构造（而非拷贝）
    else
        return b;
}
```

### ❌ 避免的写法（画蛇添足）
```cpp
// 多此一举：显式使用 std::move 会破坏 RVO/NRVO！
BigObject create() {
    BigObject result;
    return std::move(result);  // ❌ 强制移动构造，阻止 NRVO！
}
// 错误观念：以为用指针能避免拷贝（实际上更糟）
BigObject* create() {
    return new BigObject();  // ❌ 堆分配 + 手动管理，不如 RVO 安全高效
}
```

---

## 9. 一个直观的验证代码（看编译器行为）
```cpp
#include <iostream>
using namespace std;
class Test {
public:
    Test() { cout << "构造\n"; }
    Test(const Test&) { cout << "拷贝构造\n"; }
    Test(Test&&) { cout << "移动构造\n"; }
    ~Test() { cout << "析构\n"; }
};
// 场景1：RVO（返回临时对象）
Test create_rvo() {
    return Test();
}
// 场景2：NRVO（返回具名对象）
Test create_nrvo() {
    Test local;
    return local;
}
int main() {
    cout << "=== RVO ===\n";
    Test t1 = create_rvo();
    
    cout << "\n=== NRVO ===\n";
    Test t2 = create_nrvo();
    
    return 0;
}
```
**在 `-O2` 优化下输出（GCC/Clang）：**
```text
=== RVO ===
构造
=== NRVO ===
构造

```
（只有构造，没有任何拷贝/移动构造！）

**在 `-O0`（无优化）下输出（C++11/14）：**
```text
=== RVO ===
构造
=== NRVO ===
构造
移动构造
析构
```
（NRVO 在无优化下可能不生效，退化为移动构造）

---

## 10. 总结：一张图理解全貌
```text
函数返回对象 → 编译器尝试拷贝消除
                ↓
        返回临时对象？（RVO）
                ↓
        ✅ C++17 强制 RVO → 零拷贝
                ↓
        返回具名对象？（NRVO）
                ↓
        ✅ 编译器优化允许 → 零拷贝（通常 -O2 开启）
                ↓
        优化失败 → 退化为 移动构造（C++11）或 拷贝构造（C++98）


```

---

### 最后，检验你理解深度的一道题：
```cpp
std::string getString() {
    std::string s = "hello";
    return s + " world";  // 这个会触发 RVO 还是 NRVO？还是两者都触发？
}
```
**提示：** `s + " world"` 会生成一个临时对象。想想这个临时对象是"具名"还是"无名"的，编译器能不能优化？