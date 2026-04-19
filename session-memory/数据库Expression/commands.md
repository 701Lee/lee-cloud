# Commands

## 关键命令 / 操作

### 1. 编译项目
在 MiniOB 项目根目录执行：
```bash
cmake -B build
cmake --build build -j
```
或：
```bash
cd build
make -j
```

### 2. 重点看编译失败位置
当前高频失败文件：
- `yacc_sql.y`
- `condition_filter.cpp`
- `select_stmt.cpp`
- `update_stmt.cpp`
- `logical_plan_generator.cpp`
- `parse_defs.h`

### 3. 全局搜索旧条件模型残留
```bash
grep -R "left_is_attr" -n src
grep -R "right_is_attr" -n src
grep -R "left_attr" -n src
grep -R "right_attr" -n src
grep -R "left_value" -n src
grep -R "right_value" -n src
```

### 4. 搜索 Filter 旧接口残留
```bash
grep -R "FilterObj" -n src
grep -R "init_attr" -n src
grep -R "init_value" -n src
grep -R "is_attr" -n src
```

### 5. 搜索 Binder 相关参考实现
```bash
grep -R "ExpressionBinder" -n src
grep -R "BinderContext" -n src
grep -R "bind_expression" -n src
```

### 6. 搜索表达式复制接口
确认统一使用 `copy()`：
```bash
grep -R "copy() const override" -n src
```

### 7. 搜索可能触发 vector<ConditionSqlNode> 拷贝的位置
```bash
grep -R "vector<ConditionSqlNode>" -n src
grep -R "SelectSqlNode" -n src
grep -R "UpdateSqlNode" -n src
grep -R "DeleteSqlNode" -n src
```

### 8. 当前高频注意事项
- 不要写 `make_unique<Expression>(...)`
- `ConditionSqlNode` 放进 `vector` 时要 `std::move`
- Filter / 逻辑计划不要再手工恢复旧 field/value 模型
- 复制表达式时优先用 `copy()`
- `ArithmeticExpr` 等需要递归访问孩子的类，要补 const getter
