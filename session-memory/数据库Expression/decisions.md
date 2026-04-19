# Decisions

## 已确认结论

### 1. condition 语法统一为 expression
`WHERE` 中的条件改造方向已确认：
```text
condition = expression comp_op expression
```
不再继续维护 attr/value 四种组合分支。

### 2. ConditionSqlNode 必须改为表达式模型
确认使用：
```cpp
std::unique_ptr<Expression> left_expr;
CompOp comp;
std::unique_ptr<Expression> right_expr;
```

### 3. yacc 中不能实例化 Expression
`Expression` 是抽象类，不能写：
```cpp
make_unique<Expression>(...)
```
正确做法是：
- 直接接管 `$1/$3` 的所有权
- 或在后续阶段使用 `copy()` 复制具体表达式对象

### 4. ConditionSqlNode 不可拷贝，只能移动
因为内部有 `unique_ptr<Expression>`，所以：
- `emplace_back(*$1)` 错
- `emplace_back(std::move(*$1))` 对

### 5. Filter 不再自己做 attr/value 解析
`FilterStmt` 不应继续依赖：
- `left_is_attr/right_is_attr`
- `get_table_and_field`
- `FilterObj(field/value)`

确认改为：
- 直接持有左右表达式
- 统一复用 `ExpressionBinder`

### 6. Filter 的绑定环境应来自 BinderContext
`FilterStmt::create/create_filter_unit` 后续应接收 `BinderContext &`，而不是继续靠：
- `Db *db`
- `Table *default_table`
- `unordered_map<string, Table *> *tables`

### 7. 逻辑计划层也要保持 expression 模型
`LogicalPlanGenerator::create_plan(FilterStmt *, ...)` 不应再把过滤条件降级回：
- `FieldExpr`
- `ValueExpr`

而应直接从 `FilterUnit` 取左右表达式，使用 `copy()` 复制，再做类型对齐和 `ComparisonExpr` 构造。

### 8. 项目里的表达式复制接口是 copy()，不是 clone()
例如 `ArithmeticExpr` 已明确实现：
```cpp
unique_ptr<Expression> copy() const override
```
因此 Filter / 逻辑计划 侧应使用 `copy()`。

### 9. ArithmeticExpr 需要 const getter
为了让递归检查函数能访问子表达式，确认需要补：
```cpp
const unique_ptr<Expression> &left() const;
const unique_ptr<Expression> &right() const;
```

### 10. 存储层 ConditionFilter 目前先按兼容层处理
`condition_filter.cpp` 更像旧存储路径，不是当前主战场。
当前最稳策略：
- 暂不彻底 expression 化
- 先把 `DefaultConditionFilter::init(Table &, const ConditionSqlNode &)` 改成兼容层
- 只支持 `FieldExpr` / `ValueExpr`
- 复杂表达式先返回 `RC::UNSUPPORTED`
