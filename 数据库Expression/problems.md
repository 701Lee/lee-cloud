# Problems

## 还没解决的问题

### 1. yacc 编译链路是否已完全打通
虽然已经明确：
- `condition` 不能 `make_unique<Expression>`
- `condition_list` 必须 `std::move`

但还需要实际确认：
- `ConditionSqlNode` 在 parser / stmt 之间的所有权流转是否完整
- `%destructor` 与 `unique_ptr` 交接后是否没有重复释放问题

### 2. FilterUnit 改造后的下游执行逻辑还没完全检查
旧执行逻辑大概率仍依赖：
- `left().is_attr`
- `right().value`
- `FilterObj`

这些地方如果还存在，就说明执行阶段还没完成 expression 化。

### 3. check_expression_valid_for_filter 还没真正落地
已经确定要写，但还没完全实现并验证。主要待确认：
- 项目里的 `ExprType` 枚举名称
- 各表达式子类的访问接口
- 对哪些表达式类型返回 `RC::SUCCESS`
- 对哪些表达式类型返回错误

### 4. check_comparable 还没真正落地
目前只有思路，没有完整实现。需要确认：
- `Expression::value_type()` 是否足以用于比较检查
- 是否要支持更多类型组合
- `< > <= >= = !=` 是否都能共用同一套类型规则

### 5. FilterStmt::create / create_filter_unit 的签名改动还没彻底收口
目前方向已定为传 `BinderContext &`，但还需要统一以下调用链：
- 头文件声明
- cpp 实现
- `SelectStmt::create` 调用处
- 其他可能引用旧接口的位置

### 6. ExpressionBinder 在 filter 场景的使用细节还需最终确认
虽然方向已明确，但仍需确认：
- `BinderContext` 是否已包含 filter 绑定所需全部上下文
- `bind_expression` 绑定一个 filter 表达式后是否一定只产生一个表达式
- 如果出现多个表达式，返回什么错误码更合适

### 7. ArithmeticExpr 之外的其他表达式子类是否也需要 const getter / copy 适配
当前只明确检查了 `ArithmeticExpr`，但还要确认：
- 其他会递归访问子表达式的类
- 是否同样缺少 const 访问接口

### 8. 执行阶段的“表达式求值再比较”逻辑还没有系统排查
最终目标应是：
```text
对当前 tuple 分别计算 left_expr / right_expr
再按 comp 比较
```
但目前还没完整确认是否所有旧路径都已经或准备切到这套模型。
