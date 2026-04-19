# Current Task

## 当前正在做的事
把 MiniOB 中 `WHERE` 条件处理链路，从旧的 `attr/value` 模型，改造成 `expression vs expression` 模型。

## 正在改的关键位置
1. `yacc_sql.y`
   - `condition` 改为 `expression comp_op expression`
   - `condition_list` 改为用 `std::move` 放入 `vector<ConditionSqlNode>`

2. `ConditionSqlNode`
   - 删除旧字段：
     - `left_is_attr/right_is_attr`
     - `left_attr/right_attr`
     - `left_value/right_value`
   - 改为：
     - `unique_ptr<Expression> left_expr`
     - `CompOp comp`
     - `unique_ptr<Expression> right_expr`

3. `FilterUnit`
   - 从保存 `FilterObj` 改为直接保存：
     - `unique_ptr<Expression> left_`
     - `unique_ptr<Expression> right_`

4. `FilterStmt::create/create_filter_unit`
   - 去掉旧的 attr/value 分支判断
   - 改为使用 `BinderContext + ExpressionBinder` 绑定左右表达式

5. `ArithmeticExpr`
   - 已确认项目使用 `copy()`，不是 `clone()`
   - 需要补 `const` 版本的 `left()/right()` getter

## 当前阶段的重点
不是继续扩展新功能，而是先把：
- yacc 能编译通过
- Filter 能编译通过
- 绑定链路接通

优先保证主链路能跑通。