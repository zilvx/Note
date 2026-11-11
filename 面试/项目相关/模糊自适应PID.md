一句话概括：**用模糊推理系统，根据当前误差 e 和误差变化率 ec，在线动态调整 PID 的三个参数 Kp、Ki、Kd**，从而兼顾快速性、稳定性和鲁棒性。
## 一、整体结构
```text
                    ┌──────────────────────────────┐
                    │        模糊推理系统           │
                    │                              │
   e  ──► 量化 ──►  │  模糊化 → 模糊规则 → 解模糊   │ ──► ΔKp, ΔKi, ΔKd
   ec ──► 量化 ──►  │                              │
                    └──────────────────────────────┘
                                    │
                                    ▼
                    Kp = Kp0 + ΔKp
                    Ki = Ki0 + ΔKi      ──►  常规 PID  ──►  u(t)
                    Kd = Kd0 + ΔKd
```

模糊系统直接输出 ΔKp、ΔKi、ΔKd，叠加到初始值上。
 
## 二、输入输出变量设计

### 1. 输入变量

|变量|含义|计算方式|
|---|---|---|
|**e**|误差|e(k) = SP - y(k)|
|**ec**|误差变化率|ec(k) = e(k) - e(k-1)|

### 2. 输出变量

|变量|含义|
|---|---|
|**ΔKp**|比例增益修正量|
|**ΔKi**|积分增益修正量|
|**ΔKd**|微分增益修正量|

### 3. 量化因子与比例因子

实际物理量需要映射到模糊论域：
```
E  = Ke · e          （e 的量化因子）
EC = Kec · ec        （ec 的量化因子）
ΔKp = K1 · ΔKp_fuzzy  （解模糊后乘比例因子）
ΔKi = K2 · ΔKi_fuzzy
ΔKd = K3 · ΔKd_fuzzy
```

**量化因子选择：**
- Ke = n / e_max（n 为论域最大值，通常取 6）
- Kec = m / ec_max
- 输出比例因子决定修正幅度，需整定

**关键点**：量化因子和比例因子本身就是"粗调"，模糊规则是"细调"。

## 三、模糊化（Fuzzification）

### 1. 模糊集划分

将输入变量划分为若干模糊子集，常用 **7 个语言变量**：
```
{NB, NM, NS, ZO, PS, PM, PB}
负大 负中 负小 零 正小 正中 正大
```

也可用 5 个（NB, NS, ZO, PS, PB）或 9 个，精度与计算量折中。

### 2. 隶属度函数（Membership Function）

常用形状：

|形状|特点|适用|
|---|---|---|
|**三角形**|计算简单，最常用|通用|
|**梯形**|平顶，抗噪|边界模糊集|
|**高斯型**|平滑，连续可导|高精度|
|**Sigmoid**|不对称|特殊场景|

**三角形隶属度函数示例（论域 [-6, 6]）：**
```
NB: (-6, -6, -4)
NM: (-6, -4, -2)
NS: (-4, -2,  0)
ZO: (-2,  0,  2)
PS: ( 0,  2,  4)
PM: ( 2,  4,  6)
PB: ( 4,  6,  6)
```

**隶属度计算（三角形）：**
```
def triangular(x, a, b, c):
    if x <= a or x >= c:
        return 0.0
    elif x <= b:
        return (x - a) / (b - a)
    else:
        return (c - x) / (c - b)
```

**实际代码中常用查表法**：把论域离散化（如 -6 到 6 共 13 个点），预先算好隶属度表，运行时查表，避免浮点运算。

## 四、模糊规则库设计（核心）

### 1. 规则形式
```
IF e is A_i AND ec is B_j THEN ΔKp is C_ij, ΔKi is D_ij, ΔKd is E_ij
```
共 7×7 = 49 条规则（每个输出参数一张规则表）。

### 2. 参数调整原则（经验总结）

这是模糊自整定 PID 的灵魂，必须理解以下规律：

#### （1）Kp 的调整原则

|阶段|e|ec|Kp 调整|原因|
|---|---|---|---|---|
|响应初期|大|—|**大**|加快响应|
|超调阶段|中|大负|**小**|抑制超调|
|稳态附近|小|小|**大**|提高精度|
|接近稳态|小|中|**中**|平衡|

#### （2）Ki 的调整原则

|阶段|e|ec|Ki 调整|原因|
|---|---|---|---|---|
|初期|大|—|**小/零**|防积分饱和|
|中期|中|—|**中**|消除稳态误差|
|后期|小|小|**大**|提高精度|
|超调|中|大负|**小**|防积分过冲|

#### （3）Kd 的调整原则

|阶段|e|ec|Kd 调整|原因|
|---|---|---|---|---|
|初期|大|—|**小**|防微分噪声放大|
|中期|中|中|**中**|提供阻尼|
|后期|小|小|**大**|抑制振荡|
|超调|中|大负|**大**|提前制动|

### 3. 典型 ΔKp 规则表（7×7）

行 = e，列 = ec，值 = ΔKp 语言变量：

|e\ec|NB|NM|NS|ZO|PS|PM|PB|
|---|---|---|---|---|---|---|---|
|**NB**|PB|PB|PM|PM|PS|ZO|ZO|
|**NM**|PB|PB|PM|PS|PS|ZO|NS|
|**NS**|PM|PM|PM|PS|ZO|NS|NS|
|**ZO**|PM|PM|PS|ZO|NS|NM|NM|
|**PS**|PS|PS|ZO|NS|NS|NM|NM|
|**PM**|PS|ZO|NS|NM|NM|NM|NB|
|**PB**|ZO|ZO|NM|NM|NM|NB|NB|

**读法**：当 e 为 NB（负大）、ec 为 NB 时，ΔKp 为 PB（正大），即误差负得很大且还在负向变化时，需要大幅增大 Kp 来纠偏。

### 4. 典型 ΔKi 规则表

|e\ec|NB|NM|NS|ZO|PS|PM|PB|
|---|---|---|---|---|---|---|---|
|**NB**|NB|NB|NM|NM|NS|ZO|ZO|
|**NM**|NB|NB|NM|NS|NS|ZO|ZO|
|**NS**|NB|NM|NS|NS|ZO|PS|PS|
|**ZO**|NM|NM|NS|ZO|PS|PM|PM|
|**PS**|NM|NS|ZO|PS|PS|PM|PB|
|**PM**|ZO|ZO|PS|PS|PM|PB|PB|
|**PB**|ZO|ZO|PS|PM|PM|PB|PB|

**原则**：误差大时 Ki 取小（防饱和），误差小时 Ki 取大（消静差）。

### 5. 典型 ΔKd 规则表

|e\ec|NB|NM|NS|ZO|PS|PM|PB|
|---|---|---|---|---|---|---|---|
|**NB**|PS|NS|NB|NB|NB|NM|PS|
|**NM**|PS|NS|NB|NM|NM|NS|ZO|
|**NS**|ZO|NS|NM|NM|NS|NS|ZO|
|**ZO**|ZO|NS|NS|NS|NS|NS|ZO|
|**PS**|ZO|ZO|ZO|ZO|ZO|ZO|ZO|
|**PM**|PB|PS|PS|PS|PS|PS|PB|
|**PB**|PB|PM|PM|PM|PM|PS|PB|

**原则**：误差变化率大时增大 Kd 提供阻尼，误差小时适当增大 Kd 抑制振荡。

---

## 五、模糊推理（Inference）

### 1. Mamdani 推理（最常用）

**步骤：**
1. **计算每条规则的激活强度（firing strength）**：
```
μ_rule = min(μ_A(e), μ_B(ec))    （取小）
或 μ_rule = μ_A(e) · μ_B(ec)      （乘积）
```

2. **裁剪输出模糊集**：
```
μ_C'(z) = min(μ_rule, μ_C(z))  
```

3. **聚合所有规则输出**：
```
μ_out(z) = max over all rules μ_C'(z)  
```

### 2. Takagi-Sugeno 推理（计算更简单）

输出为**精确值**而非模糊集：
```
IF e is A AND ec is B THEN ΔKp = p·e + q·ec + r
```

**输出计算：**
```
ΔKp = Σ(μ_i · z_i) / Σ(μ_i)
```
**优点**：无需解模糊，计算量小，适合嵌入式。  
**缺点**：规则后件需要整定线性系数。

### 3. 常用简化：查表法

由于输入论域离散、规则固定，可**预先算出所有 (E, EC) 组合的输出**，形成一张查找表。运行时只需：
```
index_e  = quantize(e)    # 映射到 0~12
index_ec = quantize(ec)
delta_kp = table_kp[index_e][index_ec]
```

**这是工程上最常用的实现**，速度快，适合 MCU。

## 六、解模糊（Defuzzification）

### 1. 重心法（COG，最精确）
```
z* = ∫ z·μ(z) dz / ∫ μ(z) dz
```

离散形式：
```
z* = Σ z_i · μ(z_i) / Σ μ(z_i)
```
### 2. 加权平均法（Sugeno 常用）

```
z* = Σ μ_i · z_i / Σ μ_i
```

### 3. 最大隶属度法（最简单）

取隶属度最大的那个输出值，速度快但精度低。

### 4. 高度法 / 面积中心法

折中方案，工程中也有使用。

## 七、完整实现流程

### 1. 算法主循环
```
class FuzzyPID:
    def __init__(self):
        # 初始 PID 参数（可用 Z-N 得到）
        self.Kp0, self.Ki0, self.Kd0 = 1.0, 0.1, 0.05
        self.Kp, self.Ki, self.Kd = self.Kp0, self.Ki0, self.Kd0
        # 量化因子
        self.Ke  = 6.0 / self.e_max
        self.Kec = 6.0 / self.ec_max
        # 输出比例因子
        self.K1 = 0.3   # ΔKp 比例
        self.K2 = 0.01  # ΔKi 比例
        self.K3 = 0.02  # ΔKd 比例
        # 预计算规则表
        self.table_kp = build_rule_table("Kp")
        self.table_ki = build_rule_table("Ki")
        self.table_kd = build_rule_table("Kd")
        self.prev_e = 0.0
    def update(self, sp, y, dt):
        # 1. 计算 e, ec
        e = sp - y
        ec = (e - self.prev_e) / dt
        self.prev_e = e
        # 2. 量化
        E  = clamp(self.Ke  * e,  -6, 6)
        EC = clamp(self.Kec * ec, -6, 6)
        # 3. 查表得 ΔKp, ΔKi, ΔKd（已解模糊）
        dKp = self.table_kp[quantize(E)][quantize(EC)] * self.K1
        dKi = self.table_ki[quantize(E)][quantize(EC)] * self.K2
        dKd = self.table_kd[quantize(E)][quantize(EC)] * self.K3
        # 4. 更新 PID 参数
        self.Kp = self.Kp0 + dKp
        self.Ki = self.Ki0 + dKi
        self.Kd = self.Kd0 + dKd
        # 5. 参数限幅（重要！）
        self.Kp = clamp(self.Kp, self.Kp_min, self.Kp_max)
        self.Ki = clamp(self.Ki, self.Ki_min, self.Ki_max)
        self.Kd = clamp(self.Kd, self.Kd_min, self.Kd_max)
        # 6. 常规 PID 计算
        return self.pid_compute(e, dt)
```

### 2. 规则表构建（离线预计算）
```
python
```
def build_rule_table(param):
    table = [[0.0]*13 for _ in range(13)]  # 论域离散为 13 点
    rules = RULE_TABLE[param]  # 7x7 语言变量表
    for i, E in enumerate(range(-6, 7)):
        for j, EC in enumerate(range(-6, 7)):
            # 找到 E, EC 对应的模糊集
            mu_E  = fuzzify(E)   # 返回各模糊集隶属度
            mu_EC = fuzzify(EC)
            # 遍历所有规则，Mamdani 推理
            output = mamdani_infer(mu_E, mu_EC, rules)
            # 解模糊
            table[i][j] = defuzzify(output)
    return table

### 3. 量化函数

python

def quantize(x):
    # x 已在线性论域 [-6, 6]，映射到索引 [0, 12]
    return int(round(x)) + 6

---

## 八、工程实现中的关键问题

### 1. 量化因子与比例因子的整定

这是模糊 PID **最影响效果**的部分：

|因子|增大效果|减小效果|
|---|---|---|
|Ke|误差敏感，响应快，易振荡|响应慢，稳态精度低|
|Kec|对变化敏感，阻尼强|阻尼弱，易超调|
|K1 (ΔKp)|修正幅度大，响应快|修正弱|
|K2 (ΔKi)|消差快，易饱和|消差慢|
|K3 (ΔKd)|阻尼强，抗噪差|阻尼弱|

**整定顺序**：先定 Ke、Kec（使 E、EC 分布合理），再调 K1、K2、K3。

### 2. 参数限幅

必须限制 Kp、Ki、Kd 的范围，防止模糊输出异常导致系统失稳：

python

Kp = clamp(Kp, 0.1*Kp0, 5.0*Kp0)
Ki = clamp(Ki, 0.0, 3.0*Ki0)
Kd = clamp(Kd, 0.0, 3.0*Kd0)

### 3. 抗积分饱和

误差大时，模糊规则通常让 Ki 变小甚至为 0，这本身就是一种抗饱和。但还需配合：

- 积分分离：`|e| > 阈值` 时 Ki = 0
    
- 积分限幅：`I = clamp(I, I_min, I_max)`
    
- 反算抗饱和（back-calculation）
    

### 4. 噪声处理

ec 对噪声非常敏感，需：

- 对 y 做低通滤波（一阶惯性）
    
- 对 ec 做限幅或死区
    
- 用 `ec = (e - e_prev) / dt` 时，dt 要稳定
    

python

ec_raw = (e - self.prev_e) / dt
ec = lowpass(ec_raw, alpha=0.3)   # 一阶滤波
if abs(ec) < ec_deadzone:
    ec = 0.0

### 5. 无扰切换

手动/自动切换时，让积分项 = 当前输出，避免跳变：

python

if just_switched_to_auto:
    self.integral = u_manual - self.Kp*e - self.Kd*ec

### 6. 计算量优化

嵌入式场景：

- **查表法**替代实时模糊推理
    
- 论域离散为 13 或 25 点
    
- 隶属度用整数运算
    
- 规则表用 `const` 数组存 Flash
    

c

const int8_t KP_TABLE[13][13] = { ... };

### 7. 规则表的对称性

大多数规则表是**反对称**的：

text

ΔKp(e, ec) = -ΔKp(-e, -ec)

可只存一半，节省空间。

---

## 九、改进方向

### 1. 变论域模糊 PID

论域随误差大小自动伸缩：

text

论域 = [-α(e)·6, α(e)·6]

误差大时论域扩展，误差小时收缩，提高精度。

### 2. 自适应量化因子

text

Ke(t) = Ke0 · f(|e|)

误差大时减小 Ke，误差小时增大 Ke。

### 3. 模糊 + 神经网络

用 NN 在线调整模糊规则或隶属度函数，适应对象变化。

### 4. 模糊 + 遗传算法

用 GA 离线优化规则表和量化因子。

### 5. 二型模糊（Type-2 Fuzzy）

隶属度本身也是模糊的，抗噪更强，但计算量大。

### 6. 与 Smith 预估器结合

针对大滞后对象，模糊 PID + Smith 预估。

---

## 十、调参实战步骤

1. **确定初始 PID**：用 Z-N 或手动给一组 Kp0、Ki0、Kd0。
    
2. **确定 e、ec 范围**：根据实际工况定 e_max、ec_max。
    
3. **选量化因子**：Ke = 6/e_max，Kec = 6/ec_max。
    
4. **选规则表**：从经典表开始，按对象特性微调。
    
5. **调输出比例因子**：K1、K2、K3 从 0 开始逐渐增大，观察响应。
    
6. **加限幅和滤波**：防止异常。
    
7. **现场验证**：阶跃、扰动、设定值变化三种工况测试。
    

---

## 十一、优缺点总结

|优点|缺点|
|---|---|
|不依赖精确数学模型|规则依赖经验，整定麻烦|
|鲁棒性强，适应非线性|计算量比常规 PID 大|
|可在线调整参数|规则表设计无统一理论|
|兼顾快速性与稳定性|量化因子难调|
|易于与传统 PID 结合|极端工况可能失效|

---

## 十二、总结

模糊自整定 PID 的实现逻辑可归纳为：

text

1. 采样 e, ec
2. 量化 → E, EC
3. 模糊化 → 隶属度
4. 查规则表 / 模糊推理 → ΔKp, ΔKi, ΔKd
5. 解模糊 → 精确修正量
6. 叠加到初始值 → Kp, Ki, Kd
7. 限幅 → 常规 PID 计算 → 输出

**核心难点**：

- 规则表设计（依赖经验）
    
- 量化因子与比例因子整定
    
- 参数限幅与抗饱和
    
- 噪声处理
    

**工程建议**：

- 优先用**查表法**，离线算好规则表
    
- 初始值用 Z-N，模糊系统只做微调
    
- 一定要加**限幅、滤波、抗饱和**
    
- 规则表从经典表出发，现场微调