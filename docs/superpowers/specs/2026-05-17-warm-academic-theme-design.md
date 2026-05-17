# Warm Academic Theme Design

## Summary

将博客从 Quartz 默认风格改为暖调极简学术风。核心特征：奶油白底色、琥珀/棕褐强调色、Georgia 衬线标题、专注阅读体验的克制设计。

## Color Tokens

### Light Mode

| Token           | Color                      | Usage                      |
| --------------- | -------------------------- | -------------------------- |
| `light`         | `#fdfaf3`                  | 页面背景，奶油白           |
| `lightgray`     | `#efe5d5`                  | 代码背景、分割线、标签背景 |
| `gray`          | `#b8a88a`                  | 次要文字、边框             |
| `darkgray`      | `#5c4a3a`                  | 正文文字                   |
| `dark`          | `#3d2e1e`                  | 标题、强调文字             |
| `secondary`     | `#c17817`                  | 链接、强调色               |
| `tertiary`      | `#8b6914`                  | hover 链接、次要强调       |
| `highlight`     | `rgba(193, 120, 23, 0.12)` | 内部链接高亮背景           |
| `textHighlight` | `#f5d78a88`                | 文字高亮标记               |

### Dark Mode

保持 Quartz 默认暗色模式不变，仅在亮色模式中应用暖调主题。

## Typography

- **标题**: Georgia (serif) — 学术经典衬线字体
- **正文**: Source Sans Pro (sans-serif) — 保持高可读性
- **代码**: IBM Plex Mono (monospace) — 保持当前设置

## Scope

- 修改 `quartz.config.ts` 中的 `theme.colors.lightMode` 和 `theme.typography`
- 不修改布局结构 (`quartz.layout.ts`)
- 不修改 emitter/transformer 插件
- 不修改暗色模式配色
