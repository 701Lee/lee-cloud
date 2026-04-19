# Commands

## 关键操作

### 1. 编译项目
在 MiniOB 项目根目录执行：
```bash
cmake -B build
cmake --build build -j
```
或在已有构建目录中直接：
```bash
cd build
make -j
```

### 2. 重点检查 yacc 编译错误
当前最常见失败点是：
- `yacc_sql.y`
- `yacc_sql.cpp.o`

编译失败时优先看：
- `condition` 规则
- `condition_list` 规则
- `ConditionSqlNode` 的拷贝/移动相关报错

### 3. 全局搜索旧条件模型残留
重点搜索这些旧字段/旧逻辑：
```bash
grep -R "left_is_attr" -n .
grep -R "right_is_attr" -n .
grep -R "left_attr" -n .
grep -R "right_attr" -n .
grep -R "left_value" -n .
grep -R "right_value" -n .
grep -R "FilterObj" -n .
```

### 4. 搜索 Filter 旧执行接口残留
```bash
grep -R "init_attr" -n .
grep -R "init_value" -n .
grep -R "is_attr" -n src
```

### 5. 搜索 Binder 相关参考实现
为了复用 `ExpressionBinder`，重点搜索：
```bash
grep -R "ExpressionBinder" -n src
grep -R "BinderContext" -n src
grep -R "bind_expression" -n src
```

### 6. 搜索表达式复制接口
确认项目里统一使用 `copy()`：
```bash
grep -R "copy() const override" -n src
```

### 7. 搜索需要补 const getter 的表达式类
```bash
grep -R "unique_ptr<Expression> &left()" -n src
grep -R "unique_ptr<Expression> &right()" -n src
```

### 8. 当前最关键的代码修改点
- `yacc_sql.y`
- `parse_defs.h / parse_defs.cpp`
- `filter_stmt.h / filter_stmt.cpp`
- `select_stmt.cpp`
- `expression.h / expression.cpp`（以及相关表达式子类）

### 9. 当前高频注意事项
- 不要写 `make_unique<Expression>(...)`
- `ConditionSqlNode` 放进 `vector` 时要 `std::move`
- Filter 里不要再手工维护 attr/value 分支
- Filter 复制表达式时优先用 `copy()`
