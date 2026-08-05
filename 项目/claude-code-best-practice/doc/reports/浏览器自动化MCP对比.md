# 浏览器自动化 MCP 对比（中文翻译）

> 中文翻译自 [reports/claude-in-chrome-v-chrome-devtools-mcp.md](../reports/claude-in-chrome-v-chrome-devtools-mcp.md)

# 浏览器自动化 MCP 全面对比报告

<table width="100%">
<tr>
<td><a href="../">← 返回 Claude Code 最佳实践</a></td>
<td align="right"><img src="../!/claude-jumping.svg" alt="Claude" width="60" /></td>
</tr>
</table>

## 执行摘要

基于广泛的研究，我分析了您截图中展示的两种工具以及第三种主要竞争者。以下是我全面的分析，帮助您为自动化测试工作选择最佳方案。

---

## 1. 三个竞争者

### **A. Chrome DevTools MCP**（您的截图 #1）
- **来源：** Google Chrome 官方团队
- **发布时间：** 2025 年 9 月公开预览
- **架构：** 基于 Chrome DevTools 协议（CDP）+ Puppeteer
- **令牌使用：** 约 19.0k 令牌（占上下文的 9.5%）
- **工具：** 6 大类别中 26 个专业化工具

### **B. Claude in Chrome**（您的截图 #2）
- **来源：** Anthropic 官方扩展
- **发布时间：** Beta 版，向所有付费计划（Pro、Max、Team、Enterprise）逐步推出
- **架构：** 具有计算机使用能力的浏览器扩展
- **令牌使用：** 约 15.4k 令牌（占上下文的 7.7%）
- **工具：** 16 个工具，包括计算机使用能力

### **C. Playwright MCP**（强力替代方案）
- **来源：** Microsoft（官方 + 社区实现）
- **架构：** 基于无障碍树的自动化
- **令牌使用：** 约 13.7k 令牌（占上下文的 6.8%）
- **工具：** 21 个工具

---

## 2. 详细功能对比

| 功能 | Chrome DevTools MCP | Claude in Chrome | Playwright MCP |
|---------|---------------------|------------------|----------------|
| **主要用途** | 调试与性能 | 通用浏览器自动化 | UI 测试与 E2E |
| **浏览器支持** | 仅 Chrome | 仅 Chrome | Chromium、Firefox、WebKit |
| **令牌效率** | 19.0k（9.5%） | 15.4k（7.7%） | 13.7k（6.8%） |
| **元素选择** | CSS/XPath 选择器 | 视觉 + DOM | 无障碍树（语义） |
| **性能追踪** | ✅ 优秀 | ❌ 无 | ⚠️ 有限 |
| **网络检查** | ✅ 深度分析 | ⚠️ 基础 | ⚠️ 基础 |
| **控制台日志** | ✅ 全部访问 | ✅ 全部访问 | ⚠️ 有限 |
| **跨浏览器** | ❌ 否 | ❌ 否 | ✅ 是 |
| **CI/CD 集成** | ✅ 优秀 | ❌ 差（需要登录） | ✅ 优秀 |
| **无头模式** | ✅ 是 | ❌ 否 | ✅ 是 |
| **身份验证** | 需要设置 | 使用您的会话 | 需要设置 |
| **定时任务** | ❌ 否 | ✅ 是 | ❌ 否 |
| **费用** | 免费 | 需要付费计划 | 免费 |
| **本地设置** | 需要 Node.js | 浏览器扩展 | 需要 Node.js |

---

## 3. 工具细分

### Chrome DevTools MCP（26 个工具）

```
输入自动化（8 个）：      click、drag、fill、fill_form、handle_dialog、
                          hover、press_key、upload_file

导航（6 个）：             close_page、list_pages、navigate_page、
                          new_page、select_page、wait_for

模拟（2 个）：            emulate、resize_page

性能（3 个）：            performance_analyze_insight、
                          performance_start_trace、performance_stop_trace

网络（2 个）：            get_network_request、list_network_requests

调试（5 个）：            evaluate_script、get_console_message、
                          list_console_messages、take_screenshot、
                          take_snapshot
```

### Claude in Chrome（16 个工具）

```
浏览器控制：              navigate、read_page、find、computer
                          （click、type、scroll）

表单交互：                form_input、javascript_tool

媒体：                    upload_image、get_page_text、gif_creator

标签管理：                tabs_context_mcp、tabs_create_mcp

开发工具：                read_console_messages、read_network_requests

实用工具：                shortcuts_list、shortcuts_execute、
                          resize_window、update_plan
```

### Playwright MCP（21 个工具）

```
导航：                    navigate、goBack、goForward、reload

交互：                    click、fill、select、hover、press、
                          drag、uploadFile

元素查询：                getElement、getElements、waitForSelector

断言：                    assertVisible、assertText、assertTitle

页面状态：                screenshot、getAccessibilityTree、
                          evaluateScript

浏览器管理：              newPage、closePage
```

---

## 4. 自动化测试用例分析

### **Chrome DevTools MCP 最适合：**

✅ **性能测试**
- 记录包含 Core Web Vitals 的性能追踪
- 识别渲染瓶颈和布局偏移
- 内存泄漏检测和 CPU 分析

✅ **深度调试**
- 网络请求检查（请求头、负载、时序）
- 控制台错误分析和堆栈追踪
- 实时 DOM 检查

✅ **CI/CD 流水线**
- 支持无头执行
- 稳定的、基于脚本的自动化
- 无身份验证状态依赖

**理想工作流：** "找出页面加载慢的原因"或"调试这个 API 调用"

---

### **Claude in Chrome 最适合：**

✅ **手动测试辅助**
- 登录到您的帐户后进行测试
- 带有视觉上下文的探索性测试
- 记录可重放的工作流

✅ **快速验证**
- 设计验证（比较 Figma 与实际输出）
- 新功能的抽查
- 开发期间读取控制台错误

✅ **定期浏览器任务**
- 定时自动化检查
- 多标签工作流管理
- 从您录制的操作中学习

**理想工作流：** "检查我的更改看起来是否正确"或"使用我的登录状态测试这个表单"

---

### **Playwright MCP 最适合：**

✅ **E2E 测试自动化**
- 跨浏览器测试（Chrome、Firefox、Safari）
- 生成可复用的测试脚本
- 页面对象模型生成

✅ **可靠 UI 测试**
- 无障碍树 = 没有易失效的选择器
- 确定性交互
- 不易因 UI 变更而中断

✅ **CI/CD 集成**
- 面向流水线的无头模式
- 从自然语言生成 Playwright 测试文件
- 与测试管理工具集成

**理想工作流：** "为这个用户流程编写 E2E 测试"或"跨浏览器测试这个"

---

## 5. 令牌效率分析

| 工具 | 令牌使用 | 占上下文百分比 | 效率评级 |
|------|-------------|--------------|-------------------|
| Playwright MCP | 约 13.7k | 6.8% | ⭐⭐⭐⭐⭐ 最佳 |
| Claude in Chrome | 约 15.4k | 7.7% | ⭐⭐⭐⭐ 良好 |
| Chrome DevTools MCP | 约 19.0k | 9.5% | ⭐⭐⭐ 可接受 |

**影响：** 在 200k 令牌上下文中：
- Playwright 为工作留下 186.3k 令牌
- Claude in Chrome 留下 184.6k 令牌
- Chrome DevTools 留下 181k 令牌

Playwright 与 Chrome DevTools 之间约 5.3k 令牌的差异对于需要大量代码上下文的复杂会话可能很重要。

---

## 6. 安全考虑

### Chrome DevTools MCP
- ✅ 默认使用隔离的浏览器配置文件
- ✅ 无云依赖
- ✅ 完全本地控制
- ⚠️ 远程调试端口安全（使用隔离的配置文件）

### Claude in Chrome
- ⚠️ 无防护措施时**攻击成功率为 23.6%**（有防御措施时降至 11.2%）
- ⚠️ 使用您实际的浏览器会话（存在 Cookie 暴露风险）
- ⚠️ 被阻止访问金融/成人/盗版网站
- ⚠️ 仍处于 Beta 版，存在已知漏洞

### Playwright MCP
- ✅ 隔离的浏览器上下文
- ✅ 无云依赖
- ✅ 成熟的安全模型（Microsoft 支持）
- ✅ 可以安全地处理身份验证

---

## 7. 安装命令

### Chrome DevTools MCP

```bash
claude mcp add chrome-devtools npx chrome-devtools-mcp@latest
```

### Claude in Chrome

```
从 Chrome 网上应用店安装（需要 Pro/Max/Team/Enterprise 计划）
```

### Playwright MCP（推荐）

```bash
# 首先，安装浏览器
npx playwright install

# 然后添加到 Claude Code（用户作用域 = 所有项目）
claude mcp add playwright -s user -- npx @playwright/mcp@latest
```

---

## 8. 推荐方案

### **针对您的自动化测试工作流：**

#### 🥇 **主要工具：Playwright MCP**

**用于：** 日常 E2E 测试、跨浏览器验证、生成测试脚本

**原因：**
- 最低的令牌使用量（为您的代码留出更多上下文）
- 跨浏览器支持（Chrome、Firefox、Safari）
- 无障碍树方法 = 更可靠的选择器
- 优秀的 CI/CD 集成
- 可以生成实际的 Playwright 测试文件
- 免费，无需订阅

#### 🥈 **辅助工具：Chrome DevTools MCP**

**用于：** 性能调试、网络分析、Core Web Vitals

**原因：**
- 在性能追踪和调试方面无与伦比
- 深度网络请求检查
- Google 官方工具，具有长期支持
- 当您需要回答"这个为什么慢？"时至关重要

#### 🥉 **场景工具：Claude in Chrome**

**用于：** 登录状态下快速手动验证、探索性测试、设计验证

**原因：**
- 适合开发期间的快速视觉检查
- 可以读取您的登录状态
- 对于"这个看起来对吗？"的验证很有用
- 在 CI/CD 或严肃的测试自动化中可跳过

---

## 9. 推荐设置

```bash
# 同时安装 Playwright 和 Chrome DevTools MCP
npx playwright install
claude mcp add playwright -s user -- npx @playwright/mcp@latest
claude mcp add chrome-devtools -s user -- npx chrome-devtools-mcp@latest
```

### 建议的工作流

```
1. 开发     → Claude Code（终端）
2. 测试     → Playwright MCP（E2E、跨浏览器）
3. 调试     → Chrome DevTools MCP（性能、网络）
4. 验证     → Claude in Chrome（快速视觉检查）
5. CI/CD    → Playwright MCP（无头、自动化）
```

---

## 10. 最终结论

| 如果您需要... | 使用这个 |
|----------------|----------|
| 跨浏览器 E2E 测试 | **Playwright MCP** |
| 性能分析 | **Chrome DevTools MCP** |
| 网络调试 | **Chrome DevTools MCP** |
| 快速视觉验证 | **Claude in Chrome** |
| CI/CD 自动化 | **Playwright MCP** |
| 测试脚本生成 | **Playwright MCP** |
| 最低令牌使用量 | **Playwright MCP** |
| 已登录会话测试 | **Claude in Chrome** |
| 控制台日志调试 | **Chrome DevTools MCP** |

### **一句话总结：**

**同时安装 Playwright MCP 和 Chrome DevTools MCP。** 将 Playwright 作为主要测试工具（它更具令牌效率、支持跨浏览器、更适合 E2E）。在需要深度性能分析或网络调试时使用 Chrome DevTools。仅在需要登录会话进行快速手动验证时使用 Claude in Chrome。

---

## 来源

- [Chrome DevTools MCP - GitHub](https://github.com/ChromeDevTools/chrome-devtools-mcp)
- [Anthropic - Piloting Claude in Chrome](https://claude.com/blog/claude-for-chrome)
- [Claude in Chrome Help Center](https://support.claude.com/en/articles/12012173-getting-started-with-claude-in-chrome)
- [Playwright MCP - GitHub](https://github.com/microsoft/playwright-mcp)
- [Simon Willison - Using Playwright MCP with Claude Code](https://til.simonwillison.net/claude-code/playwright-mcp-claude-code)
- [Testomat.io - Playwright MCP Claude Code](https://testomat.io/blog/playwright-mcp-claude-code/)
- [MCP Integration Guide - Scrapeless](https://www.scrapeless.com/en/blog/mcp-integration-guide)
- [Chrome DevTools MCP Guide - Vladimir Siedykh](https://vladimirsiedykh.com/blog/chrome-devtools-mcp-ai-browser-debugging-complete-guide-2025)
- [Addy Osmani - Give your AI eyes](https://addyosmani.com/blog/devtools-mcp/)

---

*本报告由 Claude Code 使用 Opus 4.5 模型于 2025 年 12 月 19 日生成。*
