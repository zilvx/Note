## 一、Qt 对象模型：`QObject` + 父子树

Qt 不是标准 C++ 的简单封装，它有一套自己的**对象模型**，所有 Qt 类都直接或间接继承 `QObject`。

C++

```cpp
// QWidget → QObject → 继承链
class TableWidget : public QWidget {  // QWidget 继承 QObject
    Q_OBJECT
    // ...
};
```

### 父子所有权（核心内存管理机制）

C++

```cpp
// TableWidget.cpp:18
auto *layout = new QVBoxLayout(this);   // this 是 parent
// TableWidget.cpp:21
m_table = new QTableWidget(0, 0, this); // this 是 parent
// TableWidget.cpp:33
m_table->setItemDelegate(new PenIconDelegate(m_table)); // m_table 是 parent
```

Qt 的规则：**指定了 parent 的对象不需要手动 delete**。当 parent 析构时，会自动递归删除所有子对象。这是 Qt 管理内存的核心方式，替代了 C++ 的 `new`/`delete` 配对。

Plain Text

```
TableWidget (parent)
├── QVBoxLayout (child)
│   └── QTableWidget (child of layout, parent of delegate)
│       └── PenIconDelegate (child)
├── EditController (child)
└── 析构时: TableWidget 销毁 → 递归销毁所有子对象
```

---

## 二、`Q_OBJECT` 宏与 MOC（元对象编译器）

C++

```cpp
class TableWidget : public QWidget {
    Q_OBJECT   // ← 必须放在类定义最前面
public:
    // ...
signals:
    void cellEditingStarted(int row, int col);  // 信号声明
    void cellEditingFinished();
public slots:
    void handleKeyInput(const QString &key);     // 槽声明
};
```

**`Q_OBJECT` 做了什么？**

1. 声明一个 `staticMetaObject` 静态成员，存储类的元信息
2. 声明 `metaObject()` 虚函数，运行时返回元信息
3. 声明 `qt_metacall()` 虚函数，运行时调用信号/槽
4. 声明 `qt_metacast()` 虚函数，运行时类型转换

**MOC（Meta-Object Compiler）**：Qt 的代码生成器，在编译前扫描含有 `Q_OBJECT` 的头文件，生成 `moc_*.cpp`，其中包含信号/槽的实现代码。CMake 中通过 `AUTOMOC ON` 自动触发：

cmake

```cmake
# CMakeLists.txt:61
set_target_properties(qt_table_app PROPERTIES AUTOMOC ON ...)
```

**不使用 `Q_OBJECT` 的后果**：信号/槽不会工作，`connect()` 在运行时找不到信号/槽函数，编译通过但运行时静默失败。

---

## 三、信号与槽（Signal & Slot）— Qt 的核心通信机制

信号和槽是 Qt 最重要的特性，用于对象间通信，替代回调函数和观察者模式。

### 3.1 基本语法

C++

```cpp
// 声明信号（只声明不实现，MOC 自动生成实现）
signals:
    void dataReady(const TableData &data);    // 可以带参数
    void errorOccurred(const QString &msg);

// 声明槽（普通成员函数，可以手动实现）
public slots:
    void handleKeyInput(const QString &key);
```

### 3.2 `connect()` 的三种写法

C++

```cpp
// 写法 1: 函数指针（推荐，编译期类型检查）
connect(m_editCtrl, &EditController::editingStarted,
        this,       &TableWidget::cellEditingStarted);

// 写法 2: Lambda（灵活，适合简单逻辑）
connect(keyboard, &VirtualKeyboard::keyPressed,
        table, [table](const QString &key) {
    table->handleKeyInput(key);
});

// 写法 3: 旧式字符串（不推荐，运行时检查，容易写错）
connect(sender, SIGNAL(valueChanged(int)), receiver, SLOT(onValue(int)));
```

### 3.3 信号与槽的参数规则

C++

```cpp
// 规则: 信号的参数可以比槽多，但不能少
// 信号: void editingStarted(int row, int col);
// 槽:   void onEdit(int row, int col);   // ✅ 完全匹配
// 槽:   void onEdit();                   // ✅ 忽略多余参数
// 槽:   void onEdit(int row, int col, int extra); // ❌ 槽参数比信号多
```

### 3.4 项目中的信号槽连接图

Plain Text

```
IDataProvider::dataReady(TableData)  ──→  TableWidget::loadData(TableData)
IDataProvider::errorOccurred(QString) ──→  Lambda: qWarning()

TableWidget::cellEditingStarted(int,int) ──→  Lambda: keyboard->setVisible(true)
TableWidget::cellEditingFinished()     ──→  Lambda: keyboard->setVisible(false)
VirtualKeyboard::keyPressed(QString)   ──→  TableWidget::handleKeyInput(QString)

EditController::editingStarted(int,int) ──→  TableWidget::cellEditingStarted(int,int)
EditController::editingFinished()      ──→  TableWidget::cellEditingFinished()

QComboBox::currentIndexChanged(int)    ──→  Lambda: updateWave()
QNetworkAccessManager::finished(QNetworkReply*) ──→  HttpDataProvider::onReplyFinished()
```

---

## 四、Qt 布局系统

Qt 的布局管理器负责自动计算控件的位置和大小，响应窗口缩放。

C++

```cpp
// 垂直布局: 子控件从上到下排列
auto *layout = new QVBoxLayout(this);
layout->setContentsMargins(0, 0, 0, 0);  // 四周边距 = 0
layout->setSpacing(0);                     // 控件间间距 = 0
layout->addWidget(keyboard);               // 键盘在表格下方

// 水平布局: 子控件从左到右排列
auto *ctrlBar = new QHBoxLayout();
ctrlBar->addWidget(new QLabel("波形:"));
ctrlBar->addWidget(waveCombo);
ctrlBar->addWidget(new QLabel("  频率:"));
ctrlBar->addWidget(freqCombo);
ctrlBar->addStretch();  // 弹簧: 把右边控件推到最右
```

**布局嵌套示例**：

Plain Text

```
QVBoxLayout (mainLayout)
├── QTabWidget (tabs)
│   ├── 表格页 (QWidget)
│   │   └── QVBoxLayout
│   │       ├── QTableWidget (表格)
│   │       └── VirtualKeyboard (键盘)
│   └── 波形页 (QWidget)
│       └── QVBoxLayout
│           ├── QHBoxLayout (控制栏)
│           │   ├── QLabel
│           │   ├── QComboBox
│           │   ├── QLabel
│           │   ├── QComboBox
│           │   └── stretch
│           └── WaveformChart (波形图, stretch=1)
```

---

## 五、QSS（Qt Style Sheets）— 类 CSS 样式

QSS 语法几乎与 CSS 相同，用于控制控件的外观。

### 5.1 选择器

CSS

```css
/* 类型选择器 */
QTableWidget { background-color: #323649; }

/* 子控件选择器 (::) */
QHeaderView::section { background-color: #181c34; }
QScrollBar::handle:vertical { background-color: rgba(255,255,255,0.15); }

/* 状态伪类 (:) */
QScrollBar::handle:vertical:hover { background-color: rgba(255,255,255,0.25); }
QPushButton:pressed { background-color: #1f2233; }

/* 属性选择器 ([]) */
QPushButton[keyClass="special"] { background-color: #252839; }
```

### 5.2 加载方式

C++

```cpp
// 方式 1: 从 .qrc 资源文件加载（本项目使用）
QFile styleFile(":/table.qss");     // : 前缀表示 Qt 资源文件
styleFile.open(QFile::ReadOnly);
m_table->setStyleSheet(styleFile.readAll());

// 方式 2: 直接写字符串
m_table->setStyleSheet("QTableWidget { background-color: red; }");
```

### 5.3 样式继承

控件样式**不会**自动继承父控件的样式。子控件需要单独设置：

C++

```cpp
// ScrollBarStyler::applyTo() 的思路:
// 将滚动条样式追加到控件已有的样式表后面
const QString combined = currentStyle + scrollbarStyleSheet();
area->setStyleSheet(combined);
```

---

## 六、Qt 资源系统（`.qrc`）

将静态资源文件（图片、QSS、字体等）编译进可执行文件。

XML

```xml
<!-- resources.qrc -->
<RCC>
    <qresource prefix="/">
        <file alias="table.qss">styles/table.qss</file>
        <file alias="keyboard_style.qss">styles/keyboard_style.qss</file>
        <file alias="scrollbar_style.qss">styles/scrollbar_style.qss</file>
    </qresource>
</RCC>
```

编译后通过 `:/` 前缀访问：

C++

```cpp
QFile file(":/table.qss");           // 用 alias 访问
QFile file(":/styles/table.qss");    // 用路径访问（如果没设 alias）
```

---

## 七、Qt 的自绘系统（`QPainter`）

`WaveformChart` 是项目中自绘组件的最佳范例。

C++

```cpp
void WaveformChart::paintEvent(QPaintEvent *) {
    QPainter p(this);                              // 1. 创建画笔
    p.setRenderHint(QPainter::Antialiasing, true); // 2. 开启抗锯齿

    const QRect cr = chartRect();                 // 3. 计算绘图区域

    drawBackground(p);   // 填充背景色
    drawGrid(p, cr);     // 画网格线
    drawCurve(p, cr);    // 画平滑曲线
    drawMarker(p, cr);   // 画拖动标记
    drawAxisLabels(p, cr); // 画坐标轴标签
}
```

**核心绘图类**：

|类|作用|项目中的用法|
|---|---|---|
|`QPainter`|画笔，执行所有绘制操作|`p.drawLine()`, `p.drawText()`, `p.drawPath()`|
|`QPainterPath`|复杂路径（贝塞尔曲线）|`buildSmoothPath()` 构建 Catmull-Rom 曲线|
|`QPen`|线条样式（颜色、粗细、线型）|`QPen(QColor(...), 2.0, Qt::SolidLine)`|
|`QBrush`|填充样式|`p.fillRect(rect(), QColor(...))`|
|`QColor`|颜色（支持 RGBA 透明度）|`QColor(255, 255, 255, 160)`|
|`QFont`|字体|`QFont("Monospace", 9)`|
|`QFontMetrics`|字体度量（计算文本宽高）|`fm.horizontalAdvance(text)`|

---

## 八、Qt 事件系统

Qt 的事件处理通过重写虚函数实现：

C++

```cpp
// 绘图事件 — 每次需要重绘时调用
void paintEvent(QPaintEvent *event) override;

// 鼠标事件
void mousePressEvent(QMouseEvent *event) override;   // 按下
void mouseMoveEvent(QMouseEvent *event) override;    // 移动
void mouseReleaseEvent(QMouseEvent *event) override; // 释放

// 启用鼠标追踪（不需要按下也能收到 move 事件）
setMouseTracking(true);
```

事件处理示例：

C++

```cpp
void WaveformChart::mousePressEvent(QMouseEvent *event) {
    if (event->button() != Qt::LeftButton) {  // 判断按下的键
        QWidget::mousePressEvent(event);       // 不处理的事件传给父类
        return;
    }
    // 处理左键按下...
    m_dragging = true;
    update();  // 触发重绘 → 调用 paintEvent()
}
```

---

## 九、Model/View/Delegate 模式

Qt 的表格控件使用 MVC 架构：

Plain Text

```
Model (数据)  →  View (显示)  ←  Delegate (编辑 + 自定义绘制)
    ↑                                  ↑
TableData                          PenIconDelegate
```

C++

```cpp
// 设置自定义委托
m_table->setItemDelegate(new PenIconDelegate(m_table));

// PenIconDelegate 重写 paint() 方法
void PenIconDelegate::paint(QPainter *painter,
                            const QStyleOptionViewItem &option,
                            const QModelIndex &index) const override {
    // 先调用父类绘制默认内容
    QStyledItemDelegate::paint(painter, option, index);

    // 如果可编辑，在右下角画钢笔图标
    if (index.flags() & Qt::ItemIsEditable) {
        painter->drawPixmap(iconRect, m_penPixmap);
    }
}
```

---

## 十、Qt Network 异步网络

C++

```cpp
// 1. 创建管理器（通常作为成员变量）
m_nam = new QNetworkAccessManager(this);

// 2. 连接 finished 信号
connect(m_nam, &QNetworkAccessManager::finished,
        this, &HttpDataProvider::onReplyFinished);

// 3. 发起异步 GET 请求
void fetchData() {
    QNetworkRequest req(m_url);
    req.setHeader(QNetworkRequest::ContentTypeHeader, "application/json");
    m_nam->get(req);  // 不阻塞，立即返回
}

// 4. 请求完成后自动回调
void onReplyFinished(QNetworkReply *reply) {
    reply->deleteLater();  // 延迟删除，避免在信号处理中析构
    QByteArray body = reply->readAll();
    // 解析 body...
}
```

**关键点**：

- `get()` 是**异步**的，不会阻塞 UI 线程
- 响应通过 `finished` 信号回调
- `reply->deleteLater()` 而不是 `delete reply`：在事件循环的下一个周期安全删除

---

## 十一、Qt 容器类

项目中使用 Qt 容器替代 STL 容器：

C++

```cpp
// QString — Qt 的字符串类（支持 UTF-8）
QString name = QStringLiteral("姓名");  // 编译期字符串，零运行时开销
QString msg = QString("HTTP %1: %2").arg(error).arg(errorString);  // 格式化
QString num = QString::number(3.14, 'f', 2);  // 数字格式化

// QVector — Qt 的动态数组
QVector<QPointF> m_points;

// QStringList — 字符串列表
QStringList columns;
columns.append("姓名");

// QJsonObject / QJsonArray / QJsonDocument — JSON 操作
QJsonDocument doc = QJsonDocument::fromJson(rawData);
QJsonObject root = doc.object();
QJsonArray rows = root["rows"].toArray();
```

---

## 十二、常用 Qt 惯用法

### 12.1 `QStringLiteral` vs `QString`

C++

```cpp
QString name1("姓名");                   // 运行时从 char* 构造
QString name2 = QStringLiteral("姓名");  // 编译期生成，零开销，推荐
```

### 12.2 `QOverload` 解决重载信号歧义

C++

```cpp
// QComboBox::currentIndexChanged 有两个重载版本: int 和 QString
// 明确指定使用 int 版本
connect(waveCombo, QOverload<int>::of(&QComboBox::currentIndexChanged),
        updateWave);
```

### 12.3 `qBound()` 限制值范围

C++

```cpp
m_markerX = qBound(m_data.timeRange().min, d.x(), m_data.timeRange().max);
// 等价于: m_markerX = max(min, min(d.x(), max));
```

### 12.4 `qWarning()` / `qDebug()` 日志

C++

```cpp
qWarning("TableWidget: failed to load :/table.qss");
qDebug().noquote() << "Proto HTTP Error:" << msg;  // noquote() 避免引号
```

---

## 总结：Qt 与标准 C++ 的关键区别

|概念|标准 C++|Qt|
|---|---|---|
|内存管理|`new`/`delete`，`shared_ptr`/`unique_ptr`|父子树自动管理|
|字符串|`std::string`|`QString`（UTF-8，COW）|
|容器|`std::vector`, `std::list`|`QVector`, `QList`（隐式共享）|
|通信|回调、观察者模式|信号/槽（类型安全，松耦合）|
|事件|无标准事件模型|事件系统（`QEvent`，事件循环）|
|图形|第三方库|内置 `QPainter` 自绘系统|
|网络|`libcurl` 等|内置 `QNetworkAccessManager`（异步）|
|序列化|第三方 JSON 库|内置 `QJsonDocument`|
|编译|标准编译|需要 MOC 预处理（CMake AUTOMOC 自动处理）|
