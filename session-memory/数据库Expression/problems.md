# Problems

## 还没解决的问题

### 1. `condition_filter.cpp` 仍直接依赖旧 ConditionSqlNode 字段
当前明确报错：
- `left_is_attr`
- `left_attr`
- `left_value`
- `right_is_attr`
- `right_attr`
- `right_value`

但 `ConditionSqlNode` 已改为：
- `left_expr`
- `right_expr`
- `comp`

需要把 `DefaultConditionFilter::init(Table &, const ConditionSqlNode &)` 改成兼容层。

### 2. `select_stmt.cpp` / `update_stmt.cpp` 仍调用旧版 `FilterStmt::create`
当前新签名已变成：
```cpp
FilterStmt::create(BinderContext &, const ConditionSqlNode *, int, FilterStmt *&)
```

但调用处仍传：
- `db`
- `table`
- `table_map`

需要改为传 `BinderContext`。

### 3. `parse_defs.h` 中存在 `vector<ConditionSqlNode>` 的拷贝
报错已经暴露：
- `ConditionSqlNode` 含有 `unique_ptr`
- 因此不可拷贝，只能移动
- 当前某处还触发了 `std::vector<ConditionSqlNode>` 的拷贝构造

重点要查：
- 哪个 SQL Node 结构体含有 `vector<ConditionSqlNode>`
- 哪个地方按值传递或整体复制了它

### 4. 逻辑计划层旧模型残留还需继续清理
`LogicalPlanGenerator::create_plan(FilterStmt *, ...)` 旧代码仍依赖：
- `FilterObj`
- `is_attr`
- `FieldExpr/ValueExpr` 的重新构造

需要改为直接消费 `FilterUnit` 持有的表达式。

### 5. 存储层是否要彻底 expression 化，当前还没定
目前倾向：
- 先不把 `ConditionFilter` 做成完整 expression 执行器
- 先兼容简单 `FieldExpr` / `ValueExpr`
- 待主链路跑通后，再决定是否彻底改存储过滤路径

### 6. 下游执行阶段是否仍残留旧 attr/value 逻辑，尚未完全排查
仍需继续搜索：
- `FilterObj`
- `is_attr`
- `init_attr/init_value`
- 基于 field/value 二选一的比较逻辑
