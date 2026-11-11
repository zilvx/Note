`unordered_set` 和 `unordered_map` 是 C++ 标准库中两个非常重要的**关联容器**，它们都基于**哈希表（Hash Table）**实现。与 `set`/`map`（基于红黑树）相比，它们的核心优势是**平均 O(1) 的查找、插入和删除时间复杂度**。

下面帮你梳理一下它们的核心特点和区别。

### 核心区别：存什么 vs 映射什么

这是两者最根本的不同：

|容器|存储内容|本质|典型用途|
|---|---|---|---|
|**`unordered_set`**|存储**唯一的键（Key）**|一个**集合**，只关心元素在不在|去重、快速判断元素是否存在|
|**`unordered_map`**|存储 **键值对（Key-Value Pair）**|一个**字典**，通过键查找对应的值|建立映射关系，如：单词 → 词频|

---

### 1. `unordered_set` 详解

它就像是一个“无序的、不允许重复元素的袋子”。

- **特点**：
    
    - 所有元素都是唯一的，插入重复元素时，插入操作会失败。
        
    - 元素是无序的，遍历时元素的顺序和插入顺序无关（取决于哈希结果）。
        
    - 只存储键值，没有附加的“值”信息。
        
- **常用操作**：
    ```cpp
    #include <unordered_set>
    #include <iostream>
    std::unordered_set<int> s;
    // 插入元素，返回 pair<iterator, bool>，bool 表示是否插入成功
    auto [it, inserted] = s.insert(10);  
    s.insert(20);
    s.insert(10);  // 重复插入，操作被忽略
    // 查找元素，返回迭代器；若不存在，返回 end()
    if (s.find(20) != s.end()) {
        std::cout << "找到了 20" << std::endl;
    }
    // 或者使用 count() 检查存在性（由于不重复，返回值是 0 或 1）
    if (s.count(30) == 0) {
        std::cout << "30 不存在" << std::endl;
    }
    // 遍历（顺序不可预测）
    for (int num : s) {
        std::cout << num << " ";
    }
    ```
    
- **适用场景**：
    
    - 快速去重（如过滤数组中的重复项）。
        
    - 快速查询（如检查一个字符串是否在黑名单中）。
        

---

### 2. `unordered_map` 详解

它存储的是键值对，通过**唯一的键（Key）**来访问对应的**值（Value）**。

- **特点**：
    
    - 每个键在容器中只能出现一次。
        
    - 通过键直接访问值，无需遍历。
        
    - 如果尝试用 `[]` 访问一个不存在的键，会自动插入该键并生成一个默认值（如 int 会变成 0）。
        
- **常用操作**：
    ```cpp
    #include <unordered_map>
    #include <string>
    #include <iostream>
    std::unordered_map<std::string, int> wordCount;
    // 插入键值对的几种方式
    wordCount["apple"] = 5;           // 方式1：使用 []，若键不存在则创建
    wordCount.insert({"banana", 3});  // 方式2：使用 insert
    // 查找元素
    auto it = wordCount.find("apple");
    if (it != wordCount.end()) {
        std::cout << "apple 出现 " << it->second << " 次" << std::endl;
    }
    // 使用 [] 访问不存在的键会插入默认值 (int 为 0)
    std::cout << wordCount["orange"] << std::endl; // 输出 0，并插入 "orange":0
    // 遍历键值对
    for (const auto& pair : wordCount) {
        std::cout << pair.first << ": " << pair.second << std::endl;
    }
    ```

    
- **适用场景**：
    
    - 统计词频（单词 → 计数）。
        
    - 缓存机制（ID → 对应数据对象）。
        
    - 快速配置查询（参数名 → 参数值）。
        

---

### 3. 对比总结

除了存储内容，以下是它们的综合对比，方便你快速决策：

|特性|`unordered_set`|`unordered_map`|
|---|---|---|
|**头文件**|`<unordered_set>`|`<unordered_map>`|
|**存储结构**|集合（只有键）|字典（键 + 值）|
|**插入方式**|`insert(val)`|`insert({key, val})` 或 `emplace(key, val)`|
|**查找元素**|`.find(key)` 或 `.count(key)`|`.find(key)`，或直接用 `[]`|
|**访问值**|不适用|`.at(key)`（带边界检查）或 `[]`（会插入）|
|**遍历**|元素（键）|`pair<const Key, Value>`|
|**内存开销**|较小|更大（因为要额外存储值）|

---

### 4. ⚠️ 重要注意事项

1. **自定义类型作为键**：如果键是自定义结构体/类，你需要**提供哈希函数**和**相等比较函数**，否则编译报错。因为标准库不知道如何计算自定义类型的哈希值。
    ```cpp
        struct MyKey {
        int id;
        std::string name;
        // 需要重载 operator==
        bool operator==(const MyKey& other) const {
            return id == other.id && name == other.name;
        }
    };
    // 自定义哈希函数
    struct MyKeyHash {
        std::size_t operator()(const MyKey& k) const {
            return std::hash<int>()(k.id) ^ (std::hash<std::string>()(k.name) << 1);
        }
    };
    std::unordered_set<MyKey, MyKeyHash> mySet;
    
    ```

2. **迭代器稳定性**：`unordered_set` 和 `unordered_map` 的迭代器在**重新哈希（rehash）**后可能会失效。而 `map`/`set` 的迭代器在插入操作时不会失效。如果你的程序频繁插入且需要长期持有迭代器，可以考虑 `map`/`set`。
    
3. **性能退化的可能**：在最坏情况下（所有元素哈希冲突到同一个桶），时间复杂度会退化为 **O(n)**。不过实际工程中很少发生，除非你用了极不理想的哈希函数或者被恶意攻击（如哈希 DoS）。
    

---

### 💡 选用建议

- **只是想知道“在不在”？** → 用 `unordered_set`。
    
- **需要根据一个键“找到对应值”？** → 用 `unordered_map`。
    
- **如果对顺序有要求（比如需要有序遍历）** → 考虑 `set` 或 `map`（基于红黑树，O(log n)）。
    
- **如果是百万级以下的数据**，`unordered_set`/`unordered_map` 的速度优势非常明显，优先使用它们即可。