# 数据库Expression / Session Memory Index

## 读取约定
以后当用户提到“数据库Expression”或当前 MiniOB 表达式改造项目时，优先读取本目录下以下文件：
1. `overview.md`
2. `current-task.md`
3. `decisions.md`
4. `problems.md`
5. `commands.md`
6. `next-step.md`
7. `recent-update.md`

## 文件说明
- `overview.md`：项目背景与总体目标
- `current-task.md`：当前正在推进的主任务
- `decisions.md`：已经确认的技术结论
- `problems.md`：当前未解决的问题与报错线索
- `commands.md`：关键命令、搜索点与操作方式
- `next-step.md`：下次继续时的推荐顺序
- `recent-update.md`：最近一次对话压缩总结与最新上下文

## 当前状态
- parser / Filter / 逻辑计划 正在从 `attr/value` 条件模型迁移到 `expression vs expression` 模型
- `ConditionSqlNode` 已改为保存左右表达式
- `FilterStmt::create` 已切向 `BinderContext`
- 当前新增暴露的问题主要在：
  - `condition_filter.cpp` 仍使用旧字段
  - `select_stmt.cpp` / `update_stmt.cpp` 仍调用旧版 `FilterStmt::create`
  - `parse_defs.h` 一处发生 `vector<ConditionSqlNode>` 拷贝
