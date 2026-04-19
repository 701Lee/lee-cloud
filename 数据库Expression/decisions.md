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

原因：这些信息已经在 `SelectStmt::create` 中被整理进 `BinderContext`。

### 7. create_filter_unit 的职责已确认
它应做：
1. 复制 condition 左右表达式
2. 使用 `ExpressionBinder` 绑定
3. 检查 filter 场景表达式是否合法
4. 检查左右表达式是否可比较
5. 构造 `FilterUnit`

### 8. 项目里的表达式复制接口是 copy()，不是 clone()
例如 `ArithmeticExpr` 已明确实现：
```cpp
unique_ptr<Expression> copy() const override
```
因此 Filter 侧应使用 `copy()`。

### 9. ArithmeticExpr 需要 const getter
为了让 `check_expression_valid_for_filter(const Expression &expr)` 能递归访问子表达式，确认需要补：
```cpp
const unique_ptr<Expression> &left() const;
const unique_ptr<Expression> &right() const;
```

### 10. WHERE 复用 expression，但语义不等于 SELECT
已确认：
- `WHERE` 中要禁止 `StarExpr`
- `WHERE` 中暂时禁止聚合表达式
- 算术表达式可以递归支持
