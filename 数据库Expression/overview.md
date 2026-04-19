# Overview

## 项目背景
当前项目在 MiniOB 上做一次表达式体系升级，核心目标是把 `WHERE` 条件从旧的 `attr/value` 二分模型，统一升级为：

```text
expression comp_op expression
```

这样 `SELECT`、`WHERE`、`GROUP BY` 都能围绕同一套 `Expression` 体系工作。

## 为什么要改
旧版 `condition` 只支持：
- `rel_attr comp_op value`
- `value comp_op value`
- `rel_attr comp_op rel_attr`
- `value comp_op rel_attr`

问题是：
- `WHERE` 不能自然复用 `expression`
- 不方便支持 `a + 1 > b * 2` 这类表达式比较
- Filter 后续代码大量依赖 `left_is_attr/right_is_attr`，扩展成本高

## 总体改造方向
- parser：把 `condition` 改为 `expression comp_op expression`
- parse node：`ConditionSqlNode` 改为保存左右 `Expression`
- binder：继续统一使用 `ExpressionBinder`
- filter：不再保存 field/value 二选一，而是直接保存左右表达式
- executor：后续改为“分别计算左右表达式，再比较”

## 当前已经明确的原则
- 不在 Filter 中继续维护旧的 attr/value 特判模型
- `SELECT` / `GROUP BY` / `WHERE` 都应尽量复用同一套表达式绑定逻辑
- `WHERE` 虽然复用 `expression`，但语义上仍需要额外限制，例如禁止 `*` 和聚合表达式
