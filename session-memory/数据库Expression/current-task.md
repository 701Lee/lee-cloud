# Current Task

## 当前正在做的事
把 MiniOB 中 `WHERE` 条件处理链路，从旧的 `attr/value` 模型，改造成 `expression vs expression` 模型，并把这套模型继续向逻辑计划层推进。

## 当前正在改的关键位置
1. `yacc_sql.y`
   - `condition` 已改为 `expression comp_op expression`
   - `condition_list` 需要使用 `std::move(*$1)` 放入 `vector<ConditionSqlNode>`

2. `ConditionSqlNode`
   - 已从旧字段模型迁移为：
     - `unique_ptr<Expression> left_expr`
     - `CompOp comp`
     - `unique_ptr<Expression> right_expr`

3. `FilterUnit`
   - 从 `FilterObj(field/value)` 改为直接保存：
     - `unique_ptr<Expression> left_`
     - `unique_ptr<Expression> right_`

4. `FilterStmt::create/create_filter_unit`
   - 已切向使用 `BinderContext + ExpressionBinder`
   - 不再做旧的 attr/value 分支判断

5. `LogicalPlanGenerator::create_plan(FilterStmt *, ...)`
   - 正在从旧逻辑：`FilterObj -> FieldExpr/ValueExpr`
   - 改为直接：`filter_unit->left()->copy()` / `filter_unit->right()->copy()`

## 当前刚暴露出的编译问题
- `condition_filter.cpp` 还在访问旧的 `ConditionSqlNode` 字段
- `select_stmt.cpp` / `update_stmt.cpp` 仍在调用旧版 `FilterStmt::create`
- `parse_defs.h` 一处触发了 `vector<ConditionSqlNode>` 的拷贝

## 当前阶段目标
优先让主链路重新编译通过：
- parser
- FilterStmt
- Select/Update/Delete Stmt
- LogicalPlanGenerator

再决定是否继续深入存储层的旧 `ConditionFilter` 路径。