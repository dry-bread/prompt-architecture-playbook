# prompt-architecture-playbook

A curated collection of high-quality prompts for frontend and full-stack engineering, with a focus on large-scale systems, project structure, and code architecture. These prompts are designed to guide AI in real-world engineering tasks, not just code generation.

## 📁 目录结构

```
prompt-architecture-playbook/
└── instructions/
    └── react-frontend-architecture.instructions.md   # React 前端架构规范
```

## 🚀 如何使用

### 方式一：GitHub Copilot Instructions（推荐）

1. 在你的项目根目录创建 `.github/copilot-instructions.md` 文件
2. 复制所需 instruction 的内容到该文件
3. GitHub Copilot 将自动应用这些规范

### 方式二：直接引用

在你的项目中创建 `.github/instructions/` 目录，将所需的 `.instructions.md` 文件复制进去。

### 方式三：作为 Git Submodule

```bash
git submodule add https://github.com/dry-bread/prompt-architecture-playbook.git .github/prompt-playbook
```

然后在 `.github/copilot-instructions.md` 中引用：

```markdown
See [React Frontend Architecture](.github/prompt-playbook/instructions/react-frontend-architecture.instructions.md) for coding standards.
```

## 📚 Instructions 列表

| 文件 | 描述 | 适用范围 |
|------|------|----------|
| [react-frontend-architecture.instructions.md](instructions/react-frontend-architecture.instructions.md) | React 前端代码结构设计规范，包含 MVVM 架构、RxJS 状态管理、组件职责分离等 | `**/*.tsx`, `**/*.ts` |

## 🎯 设计理念

- **可组合性**：每个 instruction 都是独立的，可以根据项目需要组合使用
- **实战导向**：所有规范都来自真实的大型项目实践
- **AI 友好**：专门针对 AI 辅助编程优化，提供清晰的结构和示例

## 🤝 贡献

欢迎提交 PR 添加新的 instructions！请确保：

1. 遵循现有的文件命名规范：`{topic}.instructions.md`
2. 包含 YAML front matter 指定 `applyTo` 范围
3. 提供清晰的示例代码
4. 更新本 README 的 Instructions 列表

## 📄 License

MIT
