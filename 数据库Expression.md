# 数据库Expression

## 1. 项目目标
- 当前改造目标：将 `WHERE` 中原本按 `attr/value` 区分的条件模型，统一升级为 `expression comp_op expression`。
- 目标收益：
  - 让 `SELECT` 与 `WHERE` 共用表达式体系。
  - 支持更复杂的过滤条件，如 `a + 1 > b * 2`。
  - 避免 condition 分支继续膨胀。

---

## 2. yacc / 语法层改造

### 2.1 原始问题
旧版 `condition` 只支持：
- `rel_attr comp_op value`
- `value comp_op value`
- `rel_attr comp_op rel_attr`
- `value comp_op rel_attr`

这会导致 `WHERE` 无法直接复用 `expression`。

### 2.2 新方向
将 `condition` 改为：
```yacc
condition:
    expression comp_op expression
```

### 2.3 正确写法
`ConditionSqlNode` 内部改为持有：
- `unique_ptr<Expression> left_expr`
- `CompOp comp`
- `unique_ptr<Expression> right_expr`

`condition` 规则中不要写：
- `make_unique<Expression>(*$1)`
- `make_unique<Expression>(*$3)`

因为 `Expression` 是抽象类，不能直接实例化。

应改为直接接管 parser 生成出的具体表达式对象：
```yacc
condition:
    expression comp_op expression
    {
      $$ = new ConditionSqlNode;
      $$->left_expr = std::unique_ptr<Expression>($1);
      $$->comp = $2;
      $$->right_expr = std::unique_ptr<Expression>($3);
    }
    ;
```

### 2.4 condition_list 的关键修改
由于 `ConditionSqlNode` 内部含有 `unique_ptr`，所以不可拷贝，只可移动。
旧写法：
```yacc
$$->emplace_back(*$1);
```
会报错。

应改为：
```yacc
$$->emplace_back(std::move(*$1));
```

推荐写法：
```yacc
condition_list:
    /* empty */
    {
      $$ = nullptr;
    }
    | condition
    {
      $$ = new vector<ConditionSqlNode>;
      $$->emplace_back(std::move(*$1));
      delete $1;
    }
    | condition AND condition_list
    {
      $$ = $3;
      $$->emplace_back(std::move(*$1));
      delete $1;
    }
    ;
```

---

## 3. ConditionSqlNode 改造思想

### 3.1 旧模型
- `left_is_attr`
- `left_attr`
- `left_value`
- `right_is_attr`
- `right_attr`
- `right_value`

### 3.2 新模型
统一改成：
```cpp
struct ConditionSqlNode {
  std::unique_ptr<Expression> left_expr;
  CompOp comp;
  std::unique_ptr<Expression> right_expr;
};
```

### 3.3 后续影响
凡是依赖旧字段的地方都要改，重点搜索：
- `left_is_attr`
- `right_is_attr`
- `left_attr`
- `right_attr`
- `left_value`
- `right_value`

---

## 4. FilterStmt / FilterUnit 改造

### 4.1 旧设计问题
旧版：
- `FilterObj` 只支持 `field/value` 二选一。
- `FilterUnit` 保存 `FilterObj left_ / right_`。

这与新的 `ConditionSqlNode(expression, expression)` 不匹配。

### 4.2 新设计方向
删掉 `FilterObj` 这一层，`FilterUnit` 直接持有表达式：
```cpp
class FilterUnit {
public:
  void set_comp(CompOp comp) { comp_ = comp; }
  void set_left(std::unique_ptr<Expression> expr) { left_ = std::move(expr); }
  void set_right(std::unique_ptr<Expression> expr) { right_ = std::move(expr); }

  CompOp comp() const { return comp_; }
  const Expression *left() const { return left_.get(); }
  const Expression *right() const { return right_.get(); }

private:
  CompOp comp_ = NO_OP;
  std::unique_ptr<Expression> left_;
  std::unique_ptr<Expression> right_;
};
```

### 4.3 create_filter_unit 的新职责
旧逻辑：
- 判断左边是不是 attr
- 判断右边是不是 value
- 手工绑定字段

新逻辑：
1. 从 `ConditionSqlNode` 取左右表达式。
2. 调 `ExpressionBinder` 绑定左右表达式。
3. 校验过滤场景下表达式是否合法。
4. 校验左右表达式能否比较。
5. 构造 `FilterUnit(left_expr, comp, right_expr)`。

### 4.4 create_filter_unit 的推荐形态
不再依赖：
- `Db *db`
- `Table *default_table`
- `unordered_map<string, Table *> *tables`

而是直接接收：
```cpp
RC FilterStmt::create_filter_unit(
    BinderContext &binder_context,
    const ConditionSqlNode &condition,
    FilterUnit *&filter_unit)
```

原因：Filter 不再自己查表查字段，而是复用统一的绑定器。

---

## 5. SelectStmt 与 FilterStmt 的职责划分

### 5.1 SelectStmt::create 的职责
- 查找并校验 `FROM` 中的表。
- 将表加入 `BinderContext`。
- 绑定 `SELECT` 列表表达式。
- 绑定 `GROUP BY` 表达式。
- 调用 `FilterStmt::create(...)` 构造 `WHERE` 过滤条件。

### 5.2 FilterStmt::create 的职责
- 遍历 conditions。
- 逐个调用 `create_filter_unit(...)`。
- 收集为 `filter_units_`。

### 5.3 关键结论
高层职责没有重叠；真正要避免的是：
- `SelectStmt` 用 `ExpressionBinder`
- `FilterStmt` 还保留一套手工字段绑定逻辑

正确做法是：
- `SELECT`
- `GROUP BY`
- `WHERE`

三者统一走 `BinderContext + ExpressionBinder`。

---

## 6. ExpressionBinder 的定位
项目中已有：
```cpp
class ExpressionBinder
```
负责把 SQL 解析出的文本表达式绑定成具体数据库对象。

它已经支持绑定：
- star
- unbound field
- field
- value
- cast
- comparison
- conjunction
- arithmetic
- aggregate

### 使用策略
Filter 改造后不要自己重新实现绑定逻辑，应直接复用：
```cpp
ExpressionBinder binder(binder_context);
binder.bind_expression(expr, bound_expressions);
```

### 额外约束
在 filter 中，左右两边最终都必须只绑定出 **1 个表达式**。
所以要检查：
```cpp
left_bound_exprs.size() == 1
right_bound_exprs.size() == 1
```
否则像 `*` 展开成多个表达式，就不合法。

---

## 7. Filter 场景下必须补的配套函数

### 7.1 check_expression_valid_for_filter
职责：检查某个表达式是否允许出现在 `WHERE` / filter 中。

第一版建议规则：
- 允许：
  - `VALUE`
  - `FIELD`
  - `ARITHMETIC`
- 禁止：
  - `STAR`
  - `AGGREGATE`

并且对算术表达式递归检查其左右孩子。

### 7.2 check_comparable
职责：检查左右表达式结果类型是否可比较。

第一版建议：
- 同类型允许。
- `INTS` 与 `FLOATS` 互相允许。
- 其他先返回错误。

---

## 8. ArithmeticExpr 相关适配

### 8.1 已确认现状
项目里的 `ArithmeticExpr`：
- 子表达式保存在 `unique_ptr<Expression> left_ / right_`
- 复制接口不是 `clone()`，而是：
```cpp
unique_ptr<Expression> copy() const override
```

### 8.2 对 Filter 代码的影响
因此在 `create_filter_unit` 中，不应写：
```cpp
condition.left_expr->clone()
```
而应写：
```cpp
condition.left_expr->copy()
```

### 8.3 const 访问接口问题
已有接口：
```cpp
unique_ptr<Expression> &left()
unique_ptr<Expression> &right()
```

这会导致在 `const ArithmeticExpr &` 上无法调用。

应补充：
```cpp
const unique_ptr<Expression> &left() const { return left_; }
const unique_ptr<Expression> &right() const { return right_; }
```

这样 `check_expression_valid_for_filter(const Expression &expr)` 才能递归访问子表达式。

---

## 9. 当前典型编译问题与结论

### 9.1 不能实例化 Expression
错误来源：
```cpp
make_unique<Expression>(...)
```

结论：
- `Expression` 是抽象类，不能直接创建。
- 应直接接管 `$1/$3` 指向的具体派生类对象，或通过 `copy()` 复制。

### 9.2 ConditionSqlNode 不可拷贝
错误来源：
- `vector::emplace_back(*condition_ptr)`

结论：
- `ConditionSqlNode` 内含 `unique_ptr`，所以不可拷贝。
- 必须使用 `std::move(*condition_ptr)`。

### 9.3 const 对象调用非 const getter
错误来源：
- `const ArithmeticExpr` 调用 `right()`

结论：
- 给 getter 增加 const 版本。

---

## 10. 推荐修改顺序
1. 修改 `ConditionSqlNode` 为 expression 版本。
2. 修改 yacc 中的 `condition` 与 `condition_list`，改为接管表达式所有权 + move 条件对象。
3. 修改 `FilterUnit`，直接持有左右表达式。
4. 修改 `FilterStmt::create/create_filter_unit`，改为接收 `BinderContext`。
5. 在 filter 中统一复用 `ExpressionBinder`。
6. 补 `check_expression_valid_for_filter`。
7. 补 `check_comparable`。
8. 给 `ArithmeticExpr` 的 `left/right` 增加 const 版本。
9. 使用 `copy()` 代替 `clone()`。
10. 最后检查执行阶段是否仍有旧字段访问逻辑。

---

## 11. 待继续排查的点
- `FilterStmt::~FilterStmt()` 是否正确释放 `filter_units_`。
- 执行阶段是否仍有：
  - `left().is_attr`
  - `right().value`
  - `init_attr/init_value`
- `FilterUnit` 改造后，比较执行逻辑是否已改为：
  - 先分别计算左右表达式值
  - 再按 `comp` 比较
- `Expression` 基类及各子类的 `copy()` 是否完整实现。
- `check_comparable` 中是否需要更完整的类型规则。

---

## 12. 本次讨论的核心结论
这次改造的本质不是简单替换字段名，而是把整条链路从：
```text
attr/value 条件模型
```
升级为：
```text
expression vs expression 条件模型
```

关键原则：
- parser 负责构造表达式树。
- binder 负责绑定表达式树。
- filter 负责校验和组织表达式比较。
- 后续执行阶段负责计算左右表达式并比较。

整个系统应围绕 `Expression` 体系统一，而不是在 filter 中继续维持旧的 attr/value 特判模型。
