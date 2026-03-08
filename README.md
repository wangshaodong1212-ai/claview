# Claude History Manager

> 本地私有化的 Claude Code 全局控制台 - 管理你的 Mac 电脑上的 Claude Code 本地提问记录

## 项目简介

这是一个本地 B/S 架构项目，用于管理和查看 Claude Code 的本地对话历史记录。通过直观的 Web 界面，你可以浏览所有项目、查看会话列表、阅读对话详情，并查看 Claude 在后台执行的系统命令和工具调用。

## 技术栈

### 后端
- Node.js + Express
- CORS 支持

### 前端
- Vue 3 + TypeScript
- Vite
- Element Plus
- Axios
- Marked (Markdown 渲染)
- Highlight.js (代码高亮)

## 快速开始

### 一键启动（推荐）

```bash
# macOS / Linux
./start.sh

# 或使用 npm
npm run start
```

服务启动后，打开浏览器访问: http://localhost:5173/

### 手动启动

如果需要分别启动前后端：

```bash
# 1. 安装依赖
npm run install:all

# 2. 启动后端服务器 (端口 3333)
npm run start:server

# 3. 启动前端开发服务器 (端口 5173)
npm run start:client
```

### 可用命令

```bash
npm run start          # 安装依赖并一键启动
npm run start:dev       # 启动前后端（不安装依赖）
npm run start:server    # 仅启动后端
npm run start:client    # 仅启动前端
npm run install:all     # 安装前后端依赖
npm run install:server  # 仅安装后端依赖
npm run install:client  # 仅安装前端依赖
```

## 功能特性

- 📁 **项目管理**: 浏览所有使用 Claude Code 的项目
- 💬 **会话历史**: 查看每个项目的所有会话记录
- 📝 **对话详情**: 类似微信聊天界面的对话流展示
- 🎨 **Markdown 渲染**: 完整支持 Markdown 格式，包括代码高亮
- 🔧 **工具调用**: 可折叠查看 Claude 在后台执行的系统命令和工具调用
- 🌙 **深色风格**: 简洁高效的后台管理系统 UI

## API 接口

### 获取项目列表
```
GET /api/projects
```

### 获取会话列表
```
GET /api/sessions?path=<项目路径>
```

### 获取会话详情
```
GET /api/session-detail?filePath=<会话文件路径>
```

## 项目结构

```
claude-history-manager/
├── server/                    # 后端服务
│   ├── index.js              # 主服务文件
│   └── package.json
├── client/                    # 前端应用
│   ├── src/
│   │   ├── main.ts
│   │   ├── App.vue
│   │   └── components/
│   │       └── MarkdownRenderer.vue
│   └── package.json
├── PROJECT_PLAN.md           # 项目规划文档
├── PROGRESS.md              # 开发进度文档
└── README.md                # 本文档
```

## 使用说明

1. **选择项目**: 在左侧项目列表中点击要查看的项目
2. **查看会话**: 项目选中后，下方会显示该项目的所有会话
3. **阅读对话**: 点击会话即可查看完整的对话历史
4. **工具调用**: 如果 Claude 执行了系统命令或工具调用，会在消息下方显示可折叠的工具调用信息

## 开发说明

本项目使用 Claude Code + GLM 4.7 AI 编码助手开发，采用"分步提示词策略"分四个阶段完成：

1. **阶段一**: 搭建项目骨架与后端基础 API
2. **阶段二**: 攻克核心难点——解析底层对话数据
3. **阶段三**: 初始化前端与核心 UI 布局
4. **阶段四**: 前后端联调与细节打磨

详细的开发进度请参阅 [PROGRESS.md](PROGRESS.md)。

## 许可证

MIT
