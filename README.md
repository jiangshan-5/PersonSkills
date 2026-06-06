# PersonSkills - 个人 AI Agent 技能库 🚀

本项目是为个人 AI 编码助手（如 Antigravity, Claude Code 等 Agent 框架）定制的技能（Skills）托管库。通过将特定的工作流规范、设计系统和测试框架沉淀为系统的 `SKILL.md`，能够使 Agent 在编写代码时严格遵循最佳实践，避免模板化的平庸产物。

---

## 📂 技能库目录架构

项目根目录下的 `skills/` 文件夹根据应用场景对所有技能进行了归类整理：

```text
PersonSkills/
├── README.md               # 技能库总览与使用说明
└── skills/                 # 技能文件夹
    ├── ui-design/          # 界面设计与动效类 (UI & Motion)
    ├── testing-qa/         # 测试与质量保障类 (Testing & QA)
    └── development/        # 核心开发最佳实践类 (Development)
```

---

## 🛠 技能清单与说明

### 1. 🎨 界面设计与视觉动效 (`skills/ui-design/`)
这类技能专门用于控制前端界面的审美、排版、留白、色彩和动效，彻底告别 AI 默认的“黑底紫色渐变”等千篇一律的模板风格。

*   **[taste-skill](./skills/ui-design/taste-skill/SKILL.md)**: 拒绝平庸前端页面（Anti-Slop）的视觉开发指南，提供全面的留白、排版以及用户心理契合原则。
*   **[brutalist-skill](./skills/ui-design/brutalist-skill/SKILL.md)**: 粗犷主义设计规范，强烈的视觉冲击力和独特的排版张力。
*   **[minimalist-skill](./skills/ui-design/minimalist-skill/SKILL.md)**: 极简主义设计指南，控制轻量级排版、Restrained 动效和极致的空间感。
*   **[gpt-tasteskill](./skills/ui-design/gpt-tasteskill/SKILL.md)**: 殿堂级 AIDA 布局、无缝 Bento 网格设计以及 GSAP（ScrollTrigger）高级滚动动效开发指南。
*   **[soft-skill](./skills/ui-design/soft-skill/SKILL.md)**: 柔和色彩、毛玻璃拟态（Glassmorphism）与天然温和的渐变搭配指引。
*   **[redesign-skill](./skills/ui-design/redesign-skill/SKILL.md)**: 针对既有项目的重构设计规范，指导如何在继承既有品牌资产的同时进行现代化重构。
*   **[stitch-skill](./skills/ui-design/stitch-skill/SKILL.md)**: Stitch 多组件协作开发标准，管理通用卡片、大块分区与流式布局。
*   **[brandkit](./skills/ui-design/brandkit/SKILL.md)**: 基础设计令牌（Design Tokens）与核心品牌色盘规范。

### 2. 🧪 测试与质量保障 (`skills/testing-qa/`)
这类技能确保 Agent 在写完代码后，能够运用业界顶尖的工具链自动运行测试，发现并修复潜在的 Bug。

*   **[webapp-testing](./skills/testing-qa/webapp-testing/SKILL.md)**: 针对 Web 应用的高保真自动化端到端（E2E）测试编写标准。
*   **[playwright-skill](./skills/testing-qa/playwright-skill/SKILL.md)**: Playwright 脚本生成与多平台浏览器交互调试专家级规范。
*   **[android_ui_verification](./skills/testing-qa/android_ui_verification/SKILL.md)**: 依托 Android 模拟器与 ADB 的端到端 UI 自动化测试运行及截屏比对指引。

### 3. 💻 核心开发与最佳实践 (`skills/development/`)
这类技能归纳了现代流行框架的标准写法与代码组织原则，避免 Agent 写出过时的 API 调用或凌乱的文件依赖。

*   **[image-to-code-skill](./skills/development/image-to-code-skill/SKILL.md)**: 根据 UI 截图生成精美、无瑕疵响应式代码的执行原则。
*   **[react-patterns](./skills/development/react-patterns/SKILL.md)**: 现代 React 开发模式规范，包含 Hooks 组件解耦、数据流向及状态机定义。
*   **[nextjs-best-practices](./skills/development/nextjs-best-practices/SKILL.md)**: Next.js App Router 的最佳实践（包括 Server Components 数据拉取、路由缓存及性能优化）。

---

## 🚀 如何安装与使用

### 第一步：克隆仓库
克隆本项目至你的本地工作目录：
```bash
git clone https://github.com/jiangshan-5/PersonSkills.git
```

### 第二步：导入到 Agent 的配置目录
Agent 会在它的全局配置文件夹中扫描所有的 `SKILL.md` 文件。你可以直接将你需要的技能子文件夹复制到 Agent 的全局 `skills` 目录下：

*   **Antigravity / Gemini IDE 用户**:
    *   **Windows 路径**: `C:\Users\<你的用户名>\.gemini\config\skills\`
    *   **MacOS / Linux 路径**: `~/.gemini/config/skills/`

**Windows 示例命令**（复制 `taste-skill` 到配置目录）：
```powershell
Copy-Item -Recurse -Force .\skills\ui-design\taste-skill C:\Users\$env:USERNAME\.gemini\config\skills\
```

### 第三步：在提示词中唤醒它
当你在与 AI Pair Programming 时，可以在需求中明确指定要使用的技能，例如：
> “使用 `design-taste-frontend` 的视觉规范，帮我重新设计并编写这个 Landing Page...”

或者
> “请调用 `webapp-testing` 写一个 E2E 测试脚本，覆盖用户登录与加入书架功能...”

Agent 将会自动加载并读取对应的 `SKILL.md`，提供符合顶尖水准的代码交付！
