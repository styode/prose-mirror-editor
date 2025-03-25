# ProseMirror 编辑器

这是一个基于 Vue 3 和 ProseMirror 的富文本编辑器项目。

## 功能特点

- 文本格式化（粗体、斜体、下划线）
- 段落样式（标题、段落）
- 列表功能（有序列表、无序列表）
- 快捷输入（如 # 生成标题，* 生成无序列表）

## 技术栈

- Vue 3
- ProseMirror
- Vite

## 推荐开发环境

[VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (确保禁用 Vetur)。

## 项目设置

```sh
npm install
```

### 开发模式

```sh
npm run dev
```

### 生产构建

```sh
npm run build
```

### 代码格式化与检查

```sh
npm run lint
npm run format
```

## 使用说明

1. 工具栏上的按钮可用于格式化文本
2. 下拉菜单可选择段落类型或标题级别
3. 支持快捷输入：
   - 输入 `#` 后空格创建一级标题
   - 输入 `##` 后空格创建二级标题
   - 输入 `###` 后空格创建三级标题
   - 输入 `*` 后空格创建无序列表
   - 输入 `1.` 后空格创建有序列表
