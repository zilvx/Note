# 样式问题
**现象**：在 Windows 下界面布局正常，到 Linux 上按钮变小、文字挤在一起，`QGroupBox` 的边框完全看不到，或者标签的边框样式和 Windows 下完全不同。但如果在 Linux 上用 Qt Creator 直接运行源码，界面又是正常的——只有部署后运行才出问题[](https://stackoverflow.com/feeds/question/15793184)。

**根因**：Qt 在不同平台上使用不同的 `QStyle` 实现。Windows 用 `QWindowsStyle` 或 `QFusionStyle`，Linux 默认用基于系统主题的 `QGtkStyle`。不同 Style 对控件的**默认边距（margin）、间距（spacing）、像素度量（pixelMetric）** 定义不同[](https://stackoverflow.com/feeds/question/31823202)[](https://stackoverflow.com/questions/31823202/qt-gui-looks-different-on-linux-mac-and-windows?rq=4#1)。

`QGroupBox` 边框消失，是因为 Linux Style 可能把 frame 的绘制逻辑改了。`QFrame` 的 `frameStyle` 在 Windows 和 Linux 下表现不一致[](https://stackoverflow.com/feeds/question/15793184)

**解决方案**：

1. **强制统一 Style**（最简单）。在 `main()` 里加一行：
    
    cpp
    
    QApplication::setStyle("Fusion");
    
    Fusion 是 Qt 自带的跨平台 Style，在所有平台外观一致。代价是失去原生外观[](https://stackoverflow.com/feeds/question/31823202)[](https://stackoverflow.com/questions/31823202/qt-gui-looks-different-on-linux-mac-and-windows?rq=4#1)。
    
2. **给控件设置 `minimumSize`**。这能保证控件不会因为平台字体度量差异被压得太小，但要注意这也会影响 Windows 版本的外观[](https://stackoverflow.com/posts/4068227/edit)。
    
3. **平台条件编译**。对于必须区分的场景：
    
    cpp
    
    #ifdef Q_OS_WIN
        label->setFrameStyle(QFrame::Panel | QFrame::Raised);
    #else
        label->setFrameStyle(QFrame::Box | QFrame::Sunken);
    #endif
    
    这是 Stack Overflow 上被采纳的答案之一，用于解决 `QGroupBox` 边框在 Linux 下不可见的问题[](https://stackoverflow.com/feeds/question/15793184)。
    

**选择建议**：如果产品对原生外观没有强要求，直接用 Fusion；如果必须保留原生外观，就针对性地用条件编译处理个别控件。

# 弹框问题
问题：点击模态框之后会出现页面无法点击的bug
### 核心原因：模态窗口的“抓取”机制在 Linux 下可能失效

当你打开一个模态对话框（比如 `QFileDialog`），Qt 的本意是**独占输入焦点**，让主窗口暂时“冻结”，只能操作弹窗。在 Windows 上，这套机制由系统底层强制执行，很可靠。

但在 Linux 的 **Wayland** 显示服务器下，情况就变了。模态窗口的行为**最终取决于 Wayland 合成器（Compositor，即窗口管理器）**。如果合成器没有完全实现相关协议，或者 Qt 的 Wayland 插件处理有差异，模态窗口就可能“抓不住”输入事件，导致**主窗口和弹窗同时失去响应**，或者点击事件被错误地路由到了主窗口上，让你感觉“点哪里都没用”[](https://runebook.dev/zh/docs/qt/qwaylandxdgtoplevel/modal-prop#1)。