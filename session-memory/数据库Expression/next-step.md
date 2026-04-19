# Next Step

## 下次继续时的推荐顺序

### Step 1. 先改 `select_stmt.cpp` / `update_stmt.cpp`
把旧调用：
```cpp
FilterStmt::create(db, table, &table_map, ..., filter_stmt)
```
改成：
```cpp
BinderContext binder_context;
binder_context.add_table(table);
FilterStmt::create(binder_context, conditions.data(), condition_num, filter_stmt)
```

`select` 场景则复用已构造好的 `binder_context`。

### Step 2. 改 `condition_filter.cpp` 为兼容层
不要立刻彻底 expression 化。
先实现：
- 从 `condition.left_expr/right_expr` 读取表达式
- 若是 `FieldExpr` / `ValueExpr`，转成旧 `ConDesc`
- 否则返回 `RC::UNSUPPORTED`

### Step 3. 排查 `parse_defs.h` 第 142 行附近
重点查清：
- 谁在拷贝 `vector<ConditionSqlNode>`
- 哪个 SQL Node 被按值传递
- 哪个默认拷贝构造被触发

### Step 4. 改逻辑计划层
把 `LogicalPlanGenerator::create_plan(FilterStmt *, ...)` 中的旧逻辑：
- `FilterObj`
- `is_attr`
- 手工恢复 `FieldExpr/ValueExpr`

改为：
- 直接 `filter_unit->left()->copy()`
- 直接 `filter_unit->right()->copy()`
- 再做 cast 与 `ComparisonExpr`

### Step 5. 编译后继续看新暴露的问题
本轮目标不是一次全改完，而是：
- 先把主链路编译继续往前推进
- 每次清掉一批旧模型残留

## 下次开始时优先看的文件
1. `src/observer/sql/stmt/select_stmt.cpp`
2. `src/observer/sql/stmt/update_stmt.cpp`
3. `src/observer/storage/common/condition_filter.cpp`
4. `src/observer/sql/parser/parse_defs.h`
5. `src/observer/sql/optimizer/logical_plan_generator.cpp`

## 下次开始时先问自己的问题
1. `FilterStmt::create` 的所有调用点是否都已切到 `BinderContext`？
2. `ConditionSqlNode` 是否还在任何地方被拷贝？
3. 存储层旧过滤器是否只作为兼容层存在，而不是继续污染主链路？
