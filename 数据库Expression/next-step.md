# Next Step

## 下次接着做什么
按这个顺序继续：

### Step 1. 先把 parser 这层彻底编译通过
重点检查并修正：
- `condition` 是否直接接管 `$1/$3` 的所有权
- `condition_list` 是否统一改成 `std::move(*$1)`
- `ConditionSqlNode` 是否只保留 `left_expr/right_expr/comp`

目标：先解决 `yacc_sql.cpp.o` 编译失败。

### Step 2. 补齐表达式类接口
至少先完成：
- `ArithmeticExpr::left()/right()` 的 const 版本
- 确认 `copy()` 在相关表达式子类中可正常工作

目标：让 filter 里的递归检查函数能编译通过。

### Step 3. 改 FilterUnit
把旧的 `FilterObj(field/value)` 模型替换掉，改为：
- `unique_ptr<Expression> left_`
- `unique_ptr<Expression> right_`
- `CompOp comp_`

目标：让 Filter 的内部表示和 `ConditionSqlNode` 保持一致。

### Step 4. 改 FilterStmt::create/create_filter_unit
调整为复用：
- `BinderContext`
- `ExpressionBinder`

重点：
- 不再依赖 `left_is_attr/right_is_attr`
- 不再依赖旧的 `get_table_and_field` 分支逻辑
- 左右表达式绑定后都必须只得到一个 bound expression

### Step 5. 先补两个最小辅助函数
优先写：
1. `check_expression_valid_for_filter`
2. `check_comparable`

先做最小版本：
- 允许字段 / 值 / 算术表达式
- 禁止 `*`
- 禁止聚合表达式
- 允许同类型比较以及 int/float 混合比较

### Step 6. 检查执行阶段旧代码残留
重点排查：
- `FilterObj`
- `is_attr`
- `init_attr/init_value`
- 基于 field/value 二选一的比较逻辑

目标：确认后续执行阶段也能走 expression 模型。

## 下次开始时最优先看的文件
1. `yacc_sql.y`
2. `parse_defs.h`
3. `filter_stmt.h`
4. `filter_stmt.cpp`
5. `select_stmt.cpp`
6. `expression.h` / `ArithmeticExpr` 定义

## 下次开始时最先问自己的 3 个问题
1. parser 是否已经不再拷贝 `ConditionSqlNode`？
2. Filter 是否已经完全摆脱 `attr/value` 旧模型？
3. 执行阶段是否仍有旧字段访问逻辑没有迁移？
