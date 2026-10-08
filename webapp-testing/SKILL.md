---
name: webapp-testing
description: 使用 Playwright 与本地 Web 应用交互并测试的工具包。支持验证前端功能、调试 UI 行为、截取浏览器截图、查看浏览器日志。
license: 完整条款见 LICENSE.txt
---

# Web 应用测试

测试本地 Web 应用时，请编写原生 Python Playwright 脚本。

**可用辅助脚本**：
- `scripts/with_server.py` — 管理服务生命周期（支持多服务）

**务必先对脚本运行 `--help`** 查看用法。**在尝试阅读源码之前不要读源码**，除非确认需要定制方案且确有必要。这些脚本可能很大，会占满上下文窗口；它们设计为**黑盒直接调用**，而非整段读入上下文。

## 决策树：如何选择方式

```
用户任务 → 是否为静态 HTML？
    ├─ 是 → 直接读取 HTML 文件以识别选择器
    │         ├─ 成功 → 用选择器编写 Playwright 脚本
    │         └─ 失败/不完整 → 按动态应用处理（见下）
    │
    └─ 否（动态 Web 应用）→ 服务是否已在运行？
        ├─ 否 → 运行：python scripts/with_server.py --help
        │        然后使用该辅助脚本 + 编写简化的 Playwright 脚本
        │
        └─ 是 → 先侦察再操作：
            1. 导航并等待 networkidle
            2. 截图或检查 DOM
            3. 从渲染结果识别选择器
            4. 用得到的选择器执行操作
```

## 示例：使用 with_server.py

启动服务前先运行 `--help`，再使用辅助脚本：

**单服务：**
```bash
python scripts/with_server.py --server "npm run dev" --port 5173 -- python your_automation.py
```

**多服务（例如后端 + 前端）：**
```bash
python scripts/with_server.py \
  --server "cd backend && python server.py" --port 3000 \
  --server "cd frontend && npm run dev" --port 5173 \
  -- python your_automation.py
```

编写自动化脚本时只包含 Playwright 逻辑（服务由脚本自动管理）：
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True) # 始终无头启动 chromium
    page = browser.new_page()
    page.goto('http://localhost:5173') # 服务已运行并就绪
    page.wait_for_load_state('networkidle') # 关键：等待 JS 执行完毕
    # ... 你的自动化逻辑
    browser.close()
```

## 先侦察再操作模式

1. **检查渲染后的 DOM**：
   ```python
   page.screenshot(path='/tmp/inspect.png', full_page=True)
   content = page.content()
   page.locator('button').all()
   ```

2. **从检查结果识别选择器**

3. **用识别出的选择器执行操作**

## 常见陷阱

❌ **不要**在动态应用上于等待 `networkidle` 之前检查 DOM  
✅ **要**在检查前先 `page.wait_for_load_state('networkidle')`

## 最佳实践

- **将捆绑脚本当黑盒使用** — 完成任务时先看 `scripts/` 下是否有脚本可直接帮忙。这些脚本覆盖常见复杂流程且可靠，且不挤占上下文。用 `--help` 查看用法后直接调用。
- 同步脚本使用 `sync_playwright()`
- 结束时务必关闭浏览器
- 使用清晰的选择器：`text=`、`role=`、CSS 选择器或 ID
- 适当等待：`page.wait_for_selector()` 或 `page.wait_for_timeout()`

## 参考文件

- **examples/** — 常见模式示例：
  - `element_discovery.py` — 发现页面上的按钮、链接与输入框
  - `static_html_automation.py` — 对本地 HTML 使用 `file://` URL
  - `console_logging.py` — 在自动化过程中捕获控制台日志
