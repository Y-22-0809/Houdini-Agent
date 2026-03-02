## v1.3.4 — macOS Crash Fix & Rules UI Redesign

### 🐛 Bug Fixes

#### macOS 崩溃修复 (Critical)
- **移除 `processEvents()` 递归重入** — 修复 AI 执行节点操作时 macOS 上频繁崩溃的问题
  - 根本原因：`_on_execute_tool_main_thread` 槽函数通过 `BlockingQueuedConnection` 触发，在其内部调用 `QApplication.processEvents()` 会导致 Cocoa 事件循环递归重入，触发 `EXC_BAD_ACCESS`
  - 修复：移除 `processEvents()` 调用，`BlockingQueuedConnection` 返回后主线程事件循环会自然处理队列事件
- **增加主线程安全断言** — 在工具执行入口检测是否在主线程，输出警告日志辅助调试
- **工具执行超时从 30s 增加到 60s** — 避免 `execute_python` 等长时间运行的工具误触超时

### 🎨 UI Improvements

#### 自定义规则编辑器重新设计
- 使用 `QStackedWidget` 替代 `setParent/setGeometry` 覆盖层，修复空状态提示不可见的问题
- 新增 Header 栏 + Footer 栏分区布局
- 左侧面板固定 200px 宽度，右侧编辑区自适应
- 空状态页显示 📝 图标 + 引导文案 + 居中新建按钮
- 全部 QSS 样式重写，匹配暖色调主题

---

**Full Changelog**: https://github.com/Kazama-Suichiku/Houdini-Agent/compare/v1.3.3...v1.3.4
