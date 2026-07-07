# BDD / Gherkin：验收标准写法

## 是什么

BDD（Behavior-Driven Development）用自然语言描述系统行为。Gherkin 是 BDD 的标准语法，用 Given-When-Then 结构写验收标准。

**在 SOP Design 中的位置：** Requirements Spec 的验收标准 (Acceptance Criteria) 部分。

---

## Gherkin 语法

```gherkin
Scenario: [场景名称]
  Given [前置条件]
  When [触发动作]
  Then [预期结果]
```

### 三要素

| 关键字 | 含义 | 写什么 |
|--------|------|--------|
| **Given** | 前置条件 | 系统当前状态、已有数据、用户已登录等 |
| **When** | 触发动作 | 用户点击按钮、调用 API、定时任务触发等 |
| **Then** | 预期结果 | 页面显示什么、数据库变化、返回值、副作用 |

### 扩展关键字

```gherkin
Scenario: 用户下单
  Given 用户已登录
  And 购物车中有 2 件商品         # And = 追加 Given
  And 用户默认地址为"北京市"       # 可多个 And
  When 用户点击"提交订单"
  And 选择微信支付                 # And = 追加 When
  Then 订单状态为"待支付"
  And 购物车被清空                 # And = 追加 Then
  But 库存数量不变                 # But = 反面的 Then（强调不应该发生）
```

---

## 前端 React 场景示例

```gherkin
Scenario: 搜索框输入后显示结果
  Given 用户已打开搜索页面
  And 系统中有 3 条包含"React"的文章
  When 用户在搜索框输入"React"
  And 等待 300ms 防抖
  Then 搜索结果列表显示 3 条文章
  And 每条结果高亮匹配的关键词"React"

Scenario: 搜索无结果时显示空状态
  Given 用户已打开搜索页面
  When 用户输入"xyznonexistent"
  And 等待 300ms 防抖
  Then 页面显示"未找到相关结果"
  And 显示建议："请尝试其他关键词"

Scenario: 搜索失败时显示错误提示
  Given 搜索 API 返回 500 错误
  When 用户输入任意关键词
  Then 页面显示错误提示"搜索服务暂不可用"
  And 不显示空状态或加载状态
  And 保留用户已输入的关键词
```

---

## 后端 Python/Go 场景示例

```gherkin
Scenario: 创建资源成功
  Given 用户已通过认证
  And 请求体包含有效的 name 和 description
  When 用户发送 POST /api/items
  Then 响应状态码为 201
  And 响应体包含 id, name, description, created_at
  And 数据库中新增一条对应记录

Scenario: 创建资源失败——缺少必填字段
  Given 用户已通过认证
  When 用户发送 POST /api/items，请求体不包含 name
  Then 响应状态码为 422
  And 响应体包含 error 字段，说明"name 为必填项"

Scenario: 未认证访问返回 401
  Given 用户未携带有效 token
  When 用户发送 GET /api/items
  Then 响应状态码为 401
```

---

## 最佳实践

### ✅ Do

```gherkin
# 具体、可验证
Scenario: 用户修改密码
  Given 用户已登录
  And 用户当前密码为"oldPass123"
  When 用户在修改密码页输入旧密码"oldPass123"
  And 输入新密码"newPass456"
  And 点击"确认修改"
  Then 页面提示"密码修改成功"
  And 用户被重定向到登录页
  And 用户可以用新密码"newPass456"登录
  But 用户不能用旧密码"oldPass123"登录
```

### ❌ Don't

```gherkin
# 太抽象、不可验证
Scenario: 用户可以修改密码
  Given 用户已登录
  When 用户修改密码
  Then 密码被修改

# 包含了实现细节
Scenario: 用户修改密码
  When 用户调用 PUT /api/user/password
  And 请求体中包含 oldPassword 和 newPassword 字段
  Then API 返回 200
  And user 表的 password_hash 字段已更新
```

**规则：** Gherkin 描述**行为**，不描述**实现**。不要写 API 路径、数据库字段名、组件名。

---

## 与 Harness 的衔接

当测试开关开启时，Gherkin 场景可直接转化为自动化测试用例：

| Gherkin | 测试框架映射 |
|---------|-------------|
| Given | 测试 setup：插入测试数据、mock 认证 |
| When | 触发动作：render 组件并交互 / 发送 HTTP 请求 |
| Then | 断言：expect / assert |

### Python (Behave / pytest-bdd)

```python
# features/password.feature 中的 Scenario 自动映射到：
@given("用户已登录")
def step_user_logged_in(context):
    context.token = create_test_token("test_user")

@when("用户在修改密码页输入旧密码")
def step_enter_old_password(context):
    context.response = client.put(
        "/api/user/password",
        json={"old": "oldPass123", "new": "newPass456"},
        headers={"Authorization": f"Bearer {context.token}"}
    )

@then("页面提示'密码修改成功'")
def step_success_message(context):
    assert context.response.status_code == 200
    assert context.response.json()["message"] == "密码修改成功"
```

### React (React Testing Library + Vitest/Jest)

```typescript
// Given
test('搜索框输入后显示结果', async () => {
  render(<SearchPage />)
  mockSearchAPI.mockResolvedValue({ items: mockResults })

  // When
  await user.type(screen.getByRole('searchbox'), 'React')
  await waitFor(() => screen.getByRole('list'))

  // Then
  expect(screen.getAllByRole('listitem')).toHaveLength(3)
  expect(screen.getByText('React')).toHaveClass('highlight')
})
```

---

## 验收标准质量自检

写完后逐条检查：

- [ ] 每个 Given 是可重现的状态？（不是"系统正常"这种废话）
- [ ] 每个 When 是具体的动作？（不是"用户使用搜索功能"）
- [ ] 每个 Then 是可验证的结果？（不是"用户体验良好"）
- [ ] 是否避免了实现细节？（没有 API 路径、表名、组件内部状态）
- [ ] 是否覆盖了正常流程 + 边界 + 异常？（不只写 happy path）
