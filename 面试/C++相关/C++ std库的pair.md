`std::pair` 是 C++ 标准库中的一个非常基础且实用的工具，定义在 `<utility>` 头文件中。它的核心作用是将**两个可能不同类型的值组合成一个单元**。

你可以把它想象成一个“通用容器”，专门用来存放一对数据。它也是 C++ 中许多更高级容器（如 `std::map`）的基石。

### 📦 核心特性

- **存储两个值**：`std::pair` 可以存储两个元素，类型可以相同，也可以不同。
    
- **类型灵活**：这两个元素（`first` 和 `second`）可以是任意类型，包括基本类型、自定义类、指针，甚至是另一个 `std::pair`。
    
- **固定大小**：创建后，它包含的元素数量固定为两个，不能增加或减少。
    
- **有序**：元素是顺序存储的，可以通过 `first` 和 `second` 成员变量访问。
    

### 🛠️ 基本用法

1. **包含头文件**：
    ```cpp
    #include <utility>
    ```
2. **创建 `std::pair` 对象**：
    ```cpp
    // 方法1：使用构造函数
    std::pair<int, std::string> person(25, "Alice");
    // 方法2：使用辅助函数 std::make_pair（推荐，可自动推导类型）
    auto another_person = std::make_pair(30, "Bob");
    // 方法3：使用 C++11 的初始化列表（需编译时支持）
    std::pair<int, std::string> yet_another_person{22, "Charlie"};
    ```

3. **访问元素**：通过公共成员变量 `first` 和 `second` 来访问。
    ```cpp
    #include <iostream>
    std::pair<int, std::string> person(25, "Alice");
    std::cout << "年龄: " << person.first << std::endl;   // 输出: 年龄: 25
    std::cout << "姓名: " << person.second << std::endl;  // 输出: 姓名: Alice
    // 也可修改其值
    person.first = 26;
    ```

### 💡 常用场景与示例

1. **从函数返回两个值**：这是最常见的用法，可以避免为了返回两个结果而专门定义一个结构体。
    ```cpp
    #include <utility>
    // 一个函数，返回一个整数和它的平方
    std::pair<int, int> getNumberAndSquare(int x) {
        return std::make_pair(x, x * x);
    }
    // 使用结构化绑定（C++17）直接解包
    auto [num, square] = getNumberAndSquare(5);
    std::cout << num << " 的平方是 " << square << std::endl; // 输出: 5 的平方是 25
    ```

2. **作为关联容器的元素类型**：以 `std::map` 为例，其每个元素就是一个 `std::pair<const Key, Value>`。
    ```cpp
    #include <map>
    #include <string>
    std::map<std::string, int> ages;
    ages["Alice"] = 25;
    // 遍历 map，item 的类型是 std::pair<const std::string, int>
    for (const auto& item : ages) {
        std::cout << "Key: " << item.first << ", Value: " << item.second << std::endl;
    }
    ```

3. **用于 `std::sort` 等算法**：`std::pair` 已经内置了比较运算符（`<`, `>`, `==` 等），比较时**优先比较 `first`，如果 `first` 相同则比较 `second`**。这使得它非常便于排序。
    ```cpp
    #include <vector>
    #include <algorithm>
    std::vector<std::pair<int, std::string>> vec;
    vec.emplace_back(2, "two");
    vec.emplace_back(1, "one");
    vec.emplace_back(2, "three");
    // 默认按 first 排序，再按 second 排序
    std::sort(vec.begin(), vec.end());
    // 结果: [(1, "one"), (2, "three"), (2, "two")]
    ```

### ⚠️ 注意事项

- **与 `std::tuple` 的区别**：`std::tuple` 可以存储**任意数量**的元素，而 `std::pair` 固定为两个。如果你觉得未来可能需要存储更多元素，或元素数量不确定，建议从一开始就考虑使用 `std::tuple`，以保持代码的可扩展性。
    
- **性能开销**：`std::pair` 是一个非常轻量级的封装，在大多数情况下不会引入额外的运行时性能开销，你可以放心使用。
    

总的来说，`std::pair` 是 C++ 中一个简洁、高效的工具，当你在编程中需要临时或简单地将两个数据组合在一起时，它往往是首选。如果你想了解它与 `std::tuple` 在更复杂场景下的使用细节，也可以继续问我。