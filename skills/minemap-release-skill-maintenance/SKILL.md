---
name: minemap-release-skill-maintenance
description: 从 MineMap master 发布源码审计并更新技能库。用于生成新版本 skills、版本同步、覆盖和误用审查；不用于普通业务地图开发。
metadata:
  minemap-version: 4.28.2
---

# MineMap Release Skill Maintenance

## 基线和流程

1. 检查工作区与目标分支，保护用户改动；授权更新时 fetch 并快进 master，分叉先分析，不自动 reset。
2. 有 .codegraph 时先查询 CodeGraph；无法使用时按工具反馈使用源码搜索，不自动索引。
3. 与 `minemap-skills/manifest.json` 的 sourceCommit 比较公开导出、Map、schema、实现和 demo；首次迁移从文档基线 tag 对照，涵盖中间小版本。
4. 依据独立用户任务、入口和生命周期补专题；枚举/几何不必一项一个 skill，可完善已有专题。
5. 审查推荐代码的导出、签名、参数单位/默认值、异步返回、初始化/清理及 demo 私有字段。注释和实现冲突时限定到实际运行路径。
6. 入口保留任务决策与约束，长参数说明移入 references，补按需链接；不机械复制整个源码。
7. 更新 README 当前基线、版本增量和主题索引，再运行同步与校验。

在仓库根目录执行：

```sh
node minemap-skills/scripts/sync-skills.cjs
node minemap-skills/scripts/sync-skills.cjs --check
node minemap-skills/scripts/validate-skills.cjs
node minemap-skills/scripts/inspect-skills.cjs
git diff --check
```

## 校验边界

- metadata.minemap-version 是当前适用版本，历史“某版本新增”描述应保留；不全局替换旧版本叙述。
- sync 生成版本元数据、列表和 manifest，不自动生成正确 API 内容。源码修改后必须完成实际审查再运行。
- validate 验证结构、链接、JavaScript 语法和索引；inspect 的未匹配名称/未提及导出是人工复核线索，否定示例和内部类可能误报。
- 有源码无 demo 的公开能力可记录，注明未联调；不能因缺 demo 漏掉真实入口。
- 静态检查不等于浏览器运行通过。有真实资源才联调，区分语法、源码核对和行为验证。
- 脚本用仓库现有 Node 依赖 js-yaml/@babel/parser。纯文档更新不执行上传/发布命令。
- 最后核对分支/diff 并汇报结果，提交、推送或安装到个人目录按用户任务范围处理，不由脚本隐式执行。
- [脚本](../../scripts/)、[README](../../README.md)、[生成教程](../../SKILL-AUTHORING-TUTORIAL.md)。
