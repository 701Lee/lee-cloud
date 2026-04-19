# Recent Update

## 最近一次压缩总结
本轮对话继续把 MiniOB 的条件模型从旧的 `attr/value` 体系推进到 `expression vs expression` 体系，并把问题定位从 parser/Filter 扩展到逻辑计划与存储兼容层。

### 本轮新增确认
1. 逻辑计划层不能再依赖 `FilterObj`
   - `LogicalPlanGenerator::create_plan(FilterStmt *, ...)` 旧代码仍从 `FilterObj` 构造 `FieldExpr/ValueExpr`
   - 新模型下应直接从 `FilterUnit` 拿左右表达式，并使用 `copy()` 复制后构造 `ComparisonExpr`

2. `DeleteStmt::create` 也要切换到 `BinderContext`
   - 单表 `DELETE` 场景只需：
   ```cpp
   BinderContext binder_context;
   binder_context.add_table(table);
   ```
   然后调用新版 `FilterStmt::create(...)`

3. `ConditionFilter` 更像存储层旧路径
   - 当前不把它当主战场
   - 先当兼容层处理更稳
   - `DefaultConditionFilter::init(Table &, const ConditionSqlNode &)` 可先只支持：
     - `FieldExpr`
     - `ValueExpr`
   - 复杂表达式先返回 `RC::UNSUPPORTED`

### 本轮最新编译错误总结
根据最新构建日志，主要暴露出 4 类问题：

#### A. `condition_filter.cpp` 仍使用旧字段
直接报错访问：
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

#### B. `select_stmt.cpp` / `update_stmt.cpp` 仍调用旧版 `FilterStmt::create`
当前新签名已是：
```cpp
FilterStmt::create(BinderContext &, const ConditionSqlNode *, int, FilterStmt *&)
```
但调用点仍传 6 个参数。

#### C. `parse_defs.h` 一处发生 `vector<ConditionSqlNode>` 的拷贝
说明某个含有 `vector<ConditionSqlNode>` 的 SQL Node 仍在被：
- 值传递
- 默认拷贝构造
- 或整体复制

而 `ConditionSqlNode` 因持有 `unique_ptr<Expression>` 不可拷贝。

#### D. 主链路上下游仍残留旧 attr/value 模型
说明目前只改通了中间一段，仍需继续清理：
- stmt 调用层
- logical plan
- storage condition filter 兼容层

### 当前推荐的最短推进路径
1. 先改 `select_stmt.cpp` / `update_stmt.cpp`
2. 再把 `condition_filter.cpp` 改成兼容层
3. 再去 `parse_defs.h` 第 142 行附近查清谁在拷贝 `vector<ConditionSqlNode>`
4. 然后继续推进逻辑计划层的 expression 化

## 当前状态一句话
项目已经从 parser/filter 层进入“清理上下游旧模型残留”的阶段，最新优先级是：先修 `stmt + storage 兼容层 + parse_defs 拷贝问题`，再继续推进逻辑计划。