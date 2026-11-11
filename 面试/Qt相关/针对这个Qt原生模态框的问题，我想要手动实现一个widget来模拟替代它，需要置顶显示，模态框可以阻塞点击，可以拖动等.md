# 手写一个 Qt 原生模态遮罩 Widget（置顶、阻塞点击、可拖动）

下面给你一个**不依赖系统模态机制**、纯客户端模拟的模态框方案。核心思路是：**在父窗口（或顶层窗口）上覆盖一个半透明的全屏遮罩 Widget，遮罩内部放一个可拖动的对话框 Widget**。因为遮罩本身拦截了所有鼠标事件，所以父窗口的控件自然点不到，相当于模拟了模态。

---

## 一、整体设计

text

┌─────────────────────────────────────────────┐
│  父窗口 / 顶层窗口                            │
│  ┌───────────────────────────────────────┐  │
│  │  ModalOverlay (半透明遮罩, 全屏)        │  │
│  │                                        │  │
│  │      ┌─────────────────────┐          │  │
│  │      │  ModalDialog        │          │  │
│  │      │  (可拖动, 置顶)      │          │  │
│  │      │  [标题栏] [关闭]     │          │  │
│  │      │  [内容区]            │          │  │
│  │      │  [确定] [取消]       │          │  │
│  │      └─────────────────────┘          │  │
│  │                                        │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘

- **ModalOverlay**：全屏半透明遮罩，负责拦截所有鼠标事件、绘制背景暗化、响应父窗口 resize。
    
- **ModalDialog**：真正的对话框内容，放在遮罩内部，可拖动、可置顶。
    
- 两者都是普通 `QWidget`，用 `raise()` 保证在最上层。
    

**关键优势**：完全不依赖 Wayland 合成器对模态协议的支持，在任何平台、任何显示服务器下行为一致。

---

## 二、ModalOverlay 实现

### 2.1 头文件

cpp

// modaloverlay.h
#pragma once
#include <QWidget>
#include <QPointer>
class ModalDialog;
class ModalOverlay : public QWidget {
    Q_OBJECT
public:
    explicit ModalOverlay(QWidget *parent = nullptr);
    ~ModalOverlay() override;
    // 显示遮罩并返回内部的对话框，方便调用方设置内容
    ModalDialog *showDialog(const QString &title);
    // 主动关闭
    void closeDialog();
signals:
    void accepted();
    void rejected();
protected:
    void paintEvent(QPaintEvent *event) override;
    void resizeEvent(QResizeEvent *event) override;
    void mousePressEvent(QMouseEvent *event) override;
    void mouseMoveEvent(QMouseEvent *event) override;
    void mouseReleaseEvent(QMouseEvent *event) override;
    void keyPressEvent(QKeyEvent *event) override;
private:
    void centerDialog();
    QPointer<ModalDialog> m_dialog;
    bool m_dragging = false;
    QPoint m_dragOffset;
};

### 2.2 实现

cpp

// modaloverlay.cpp
#include "modaloverlay.h"
#include "modaldialog.h"
#include <QPainter>
#include <QMouseEvent>
#include <QKeyEvent>
#include <QApplication>
ModalOverlay::ModalOverlay(QWidget *parent)
    : QWidget(parent)
{
    // 关键：让遮罩自己接收鼠标事件，不穿透到下层
    setAttribute(Qt::WA_NoSystemBackground);
    setAttribute(Qt::WA_TranslucentBackground);
    setMouseTracking(true);
    // 让遮罩可以获得键盘焦点（响应 ESC）
    setFocusPolicy(Qt::StrongFocus);
    if (parent) {
        // 跟随父窗口大小
        setGeometry(parent->rect());
        parent->installEventFilter(this); // 见下方说明
    }
}
ModalOverlay::~ModalOverlay() = default;
ModalDialog *ModalOverlay::showDialog(const QString &title)
{
    if (m_dialog) {
        return m_dialog;
    }
    m_dialog = new ModalDialog(this);
    m_dialog->setTitle(title);
    m_dialog->adjustSize();
    centerDialog();
    // 让遮罩显示在最上层
    raise();
    show();
    setFocus();
    return m_dialog;
}
void ModalOverlay::closeDialog()
{
    if (m_dialog) {
        m_dialog->deleteLater();
        m_dialog = nullptr;
    }
    hide();
    deleteLater();
}
void ModalOverlay::paintEvent(QPaintEvent *)
{
    // 半透明黑色背景，暗化下层内容
    QPainter p(this);
    p.fillRect(rect(), QColor(0, 0, 0, 120));
}
void ModalOverlay::resizeEvent(QResizeEvent *)
{
    centerDialog();
}
void ModalOverlay::centerDialog()
{
    if (!m_dialog) return;
    const QSize s = m_dialog->size();
    m_dialog->move((width() - s.width()) / 2,
                   (height() - s.height()) / 2);
}
void ModalOverlay::mousePressEvent(QMouseEvent *event)
{
    // 如果点击在对话框外部，可以选择不关闭（严格模态）
    // 如果想点击外部关闭，在这里判断并 closeDialog()
    event->accept();
}
void ModalOverlay::mouseMoveEvent(QMouseEvent *event)
{
    event->accept();
}
void ModalOverlay::mouseReleaseEvent(QMouseEvent *event)
{
    event->accept();
}
void ModalOverlay::keyPressEvent(QKeyEvent *event)
{
    if (event->key() == Qt::Key_Escape) {
        emit rejected();
        closeDialog();
        return;
    }
    QWidget::keyPressEvent(event);
}

**要点说明：**

1. **`WA_NoSystemBackground` + `WA_TranslucentBackground`**：让遮罩能绘制半透明背景。
    
2. **遮罩默认接收所有鼠标事件**，不调用 `ignore()`，事件就不会传给父窗口，实现“阻塞点击”。
    
3. **`raise()`**：保证遮罩在父窗口所有子控件之上。
    
4. **`setFocus()` + `keyPressEvent`**：支持 ESC 关闭。
    

---

## 三、ModalDialog 实现（可拖动）

### 3.1 头文件

cpp

// modaldialog.h
#pragma once
#include <QWidget>
class QLabel;
class QPushButton;
class ModalDialog : public QWidget {
    Q_OBJECT
public:
    explicit ModalDialog(QWidget *parent = nullptr);
    void setTitle(const QString &title);
    void setContentWidget(QWidget *content);
signals:
    void accepted();
    void rejected();
protected:
    void paintEvent(QPaintEvent *event) override;
    void mousePressEvent(QMouseEvent *event) override;
    void mouseMoveEvent(QMouseEvent *event) override;
    void mouseReleaseEvent(QMouseEvent *event) override;
private:
    QLabel *m_titleLabel = nullptr;
    QWidget *m_contentArea = nullptr;
    QPushButton *m_okBtn = nullptr;
    QPushButton *m_cancelBtn = nullptr;
    bool m_dragging = false;
    QPoint m_dragOffset;
};

### 3.2 实现

cpp

// modaldialog.cpp
#include "modaldialog.h"
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QLabel>
#include <QPushButton>
#include <QMouseEvent>
#include <QPainter>
#include <QStyleOption>
ModalDialog::ModalDialog(QWidget *parent)
    : QWidget(parent)
{
    setAttribute(Qt::WA_StyledBackground, true);
    setObjectName("ModalDialog");
    // 整体样式：白底、圆角、阴影
    setStyleSheet(R"(
        #ModalDialog {
            background: #ffffff;
            border: 1px solid #c0c0c0;
            border-radius: 6px;
        }
        #ModalDialogTitle {
            background: #f0f0f0;
            border-top-left-radius: 6px;
            border-top-right-radius: 6px;
            padding: 6px 10px;
            font-weight: bold;
        }
    )");
    auto *root = new QVBoxLayout(this);
    root->setContentsMargins(0, 0, 0, 0);
    root->setSpacing(0);
    // 标题栏
    m_titleLabel = new QLabel("提示", this);
    m_titleLabel->setObjectName("ModalDialogTitle");
    m_titleLabel->setFixedHeight(32);
    root->addWidget(m_titleLabel);
    // 内容区（外部可替换）
    m_contentArea = new QWidget(this);
    m_contentArea->setMinimumSize(280, 100);
    root->addWidget(m_contentArea, 1);
    // 按钮区
    auto *btnRow = new QHBoxLayout();
    btnRow->setContentsMargins(10, 8, 10, 10);
    btnRow->addStretch();
    m_okBtn = new QPushButton("确定", this);
    m_cancelBtn = new QPushButton("取消", this);
    btnRow->addWidget(m_okBtn);
    btnRow->addWidget(m_cancelBtn);
    root->addLayout(btnRow);
    connect(m_okBtn, &QPushButton::clicked, this, [this] {
        emit accepted();
        // 通知遮罩关闭
        if (auto *overlay = parentWidget()) {
            overlay->deleteLater();
        }
    });
    connect(m_cancelBtn, &QPushButton::clicked, this, [this] {
        emit rejected();
        if (auto *overlay = parentWidget()) {
            overlay->deleteLater();
        }
    });
    setMinimumWidth(320);
}
void ModalDialog::setTitle(const QString &title)
{
    m_titleLabel->setText(title);
}
void ModalDialog::setContentWidget(QWidget *content)
{
    auto *lay = qobject_cast<QVBoxLayout*>(m_contentArea->layout());
    if (!lay) {
        lay = new QVBoxLayout(m_contentArea);
        lay->setContentsMargins(12, 12, 12, 12);
    }
    lay->addWidget(content);
}
void ModalDialog::paintEvent(QPaintEvent *)
{
    QStyleOption opt;
    opt.initFrom(this);
    QPainter p(this);
    style()->drawPrimitive(QStyle::PE_Widget, &opt, &p, this);
}
// ---------- 拖动逻辑：只在标题栏区域响应 ----------
void ModalDialog::mousePressEvent(QMouseEvent *event)
{
    if (event->button() == Qt::LeftButton) {
        // 判断点击位置是否在标题栏内
        const QPoint pos = event->pos();
        if (pos.y() <= m_titleLabel->height()) {
            m_dragging = true;
            m_dragOffset = pos;
            event->accept();
            return;
        }
    }
    QWidget::mousePressEvent(event);
}
void ModalDialog::mouseMoveEvent(QMouseEvent *event)
{
    if (m_dragging && (event->buttons() & Qt::LeftButton)) {
        // 相对于父窗口（遮罩）移动
        QPoint newPos = mapToParent(event->pos() - m_dragOffset);
        // 限制在遮罩范围内，防止拖出屏幕
        QWidget *p = parentWidget();
        if (p) {
            newPos.setX(qBound(0, newPos.x(), p->width() - width()));
            newPos.setY(qBound(0, newPos.y(), p->height() - height()));
        }
        move(newPos);
        event->accept();
        return;
    }
    QWidget::mouseMoveEvent(event);
}
void ModalDialog::mouseReleaseEvent(QMouseEvent *event)
{
    m_dragging = false;
    QWidget::mouseReleaseEvent(event);
}

**要点说明：**

1. **拖动只在标题栏生效**：通过判断 `event->pos().y() <= m_titleLabel->height()` 限制拖动区域，避免在内容区误拖。
    
2. **边界限制**：用 `qBound` 把对话框限制在遮罩内，防止拖出可视区域。
    
3. **父子关系**：`ModalDialog` 的 parent 是 `ModalOverlay`，所以 `mapToParent` 得到的是遮罩坐标系。
    
4. **关闭时删除遮罩**：按钮点击后 `parentWidget()->deleteLater()`，遮罩析构时子对话框一起销毁。
    

---

## 四、如何在业务代码中使用

cpp

// 在某个按钮的槽函数里
void MainWindow::onShowModalClicked()
{
    // 1. 创建遮罩，parent 传当前窗口
    auto *overlay = new ModalOverlay(this);
    overlay->setGeometry(this->rect());  // 覆盖整个窗口
    // 2. 显示对话框
    ModalDialog *dlg = overlay->showDialog("确认操作");
    // 3. 自定义内容
    auto *content = new QWidget;
    auto *lay = new QVBoxLayout(content);
    lay->addWidget(new QLabel("确定要删除这条记录吗？", content));
    dlg->setContentWidget(content);
    // 4. 连接结果
    connect(dlg, &ModalDialog::accepted, this, [] {
        qDebug() << "用户点击了确定";
    });
    connect(dlg, &ModalDialog::rejected, this, [] {
        qDebug() << "用户点击了取消";
    });
}

---

## 五、几个关键细节

### 5.1 遮罩如何跟随父窗口 resize

在 `ModalOverlay` 构造时给父窗口安装事件过滤器：

cpp

bool ModalOverlay::eventFilter(QObject *obj, QEvent *event)
{
    if (obj == parentWidget() && event->type() == QEvent::Resize) {
        setGeometry(parentWidget()->rect());
    }
    return QWidget::eventFilter(obj, event);
}

并在头文件加上 `bool eventFilter(QObject *obj, QEvent *event) override;`。

### 5.2 遮罩如何真正拦截点击

- **不要**给遮罩设置 `Qt::WA_TransparentForMouseEvents`，否则事件会穿透。
    
- 遮罩的 `mousePressEvent` 里调用 `event->accept()`，不调用 `ignore()`。
    
- 遮罩必须是父窗口的**直接子控件**，且 `raise()` 到最上层。
    

### 5.3 如果父窗口不是顶层窗口

如果业务场景是在一个子 Widget 上弹模态，遮罩的 parent 传那个子 Widget 即可，坐标系统自动适配。

### 5.4 多显示器 / 全屏窗口

如果父窗口本身是全屏的，遮罩覆盖整个屏幕，效果就是全屏模态。如果要在整个应用级别模态（跨多个顶层窗口），需要把遮罩挂到 `QApplication::activeWindow()` 或用一个独立的顶层 `QWidget` 并设置 `Qt::Tool | Qt::FramelessWindowHint`。

### 5.5 与系统模态的区别

|特性|系统模态 `QDialog::exec()`|手写遮罩|
|---|---|---|
|依赖合成器|是（Wayland 下可能失效）|否|
|跨平台一致性|有差异|完全一致|
|阻塞方式|嵌套事件循环|事件拦截|
|能否拖动|系统决定|完全可控|
|实现复杂度|低|中|

---

## 六、进阶优化

1. **动画淡入淡出**：用 `QPropertyAnimation` 对遮罩的 `windowOpacity` 或自定义属性做动画。
    
2. **阴影效果**：给 `ModalDialog` 加 `QGraphicsDropShadowEffect`。
    
3. **点击遮罩外部关闭**：在 `ModalOverlay::mousePressEvent` 里判断点击位置是否在对话框矩形外，是则 `closeDialog()`。
    
4. **Tab 键焦点循环**：重写 `focusNextPrevChild` 或安装事件过滤器，让 Tab 在对话框内部循环，不跑到遮罩外。
    
5. **多级模态**：遮罩可以嵌套，每个遮罩管理自己的对话框，形成栈式结构。
    
6. **屏幕居中**：如果父窗口很大，`centerDialog()` 可以改成相对屏幕居中。
    

---

## 七、一句话总结

> 手写模态的核心是 **“全屏遮罩拦截事件 + 内部可拖动对话框”**。遮罩用 `WA_TranslucentBackground` 画半透明背景，靠 `accept()` 鼠标事件阻塞下层；对话框用标题栏区域判断实现拖动，用 `qBound` 限制边界。整套方案不依赖任何系统模态协议，在 X11、Wayland、Windows、macOS 下行为完全一致。

如果你需要，我可以进一步给出：

- **带淡入淡出动画的完整版本**
    
- **支持点击遮罩外部关闭的版本**
    
- **支持多级嵌套模态的版本**
    
- **把 ModalDialog 内容区做成可传入任意 QWidget 的通用接口**