# Testing Trophy & Pyramid：测试分层 + 分层策略

## 测试分层策略

### 核心原则：按层决策，不一刀切

不同层级的测试有不同的 ROI 和稳定性，应该分层对待，而不是统一开关。

```
Static Analysis  → 始终开启（零维护成本，捕获30%+ bug）
Unit Tests       → 默认开启（纯函数/工具函数，稳定可靠）
Integration      → 关键行为开启（用户可见行为、API 契约）
E2E              → 默认关闭（脆弱、维护成本高，仅核心流程）
```

### 为什么 E2E 默认关闭

**E2E 测试仍存在固有问题：**
- AI 生成的 E2E 容易写出过度耦合的 brittle tests
- 维护成本随数量非线性增长
- 运行慢、调试难、"改一行挂十个"

### 开启场景

**单测/集成测试关闭场景（满足任一）：**

| 条件 | 示例 |
|------|------|
| 纯原型/一次性代码 | 快速验证想法的 throwaway 代码 |
| 人明确要求跳过 | "这个 Task 不用写测试" |

**E2E 开启场景（满足任一）：**

| 条件 | 示例 |
|------|------|
| 核心支付/计费流程 | 下单→支付→回调 全链路 |
| 关键用户路径回归 | 登录→搜索→查看详情 主流程 |
| 人明确要求开启 | "这个流程写 E2E" |

### 开启方式

在 Implementation Spec 的 Task 中声明测试策略：

```markdown
### Task 5: 价格计算引擎
- **test: unit**（默认，无需显式声明）
- **原因**：纯计算逻辑，单测即可覆盖

### Task 3: 搜索功能端到端
- **test: e2e**
- **原因**：搜索是核心用户路径，涉及前后端联动
- **测试范围**：搜索输入 → API 调用 → 结果展示
```

---

## React 前端：Testing Trophy

Kent C. Dodds 的 Testing Trophy 是 React 社区事实标准：

```
        ┌─────────┐
        │   E2E    │  ← 极少，仅核心流程（Playwright）
        │  (少量)   │
        ├─────────┤
        │Integration│ ← 重点！大部分测试在这里
        │  (大量)   │   React Testing Library
        ├─────────┤
        │   Unit   │  ← 纯函数、工具函数、Hooks
        │  (适量)   │   Vitest / Jest
        ├─────────┤
        │  Static  │  ← 基底，TypeScript + ESLint
        │ Analysis │   永远运行，不视为"测试开关"控制
        └─────────┘
```

### 各层详解

#### Static Analysis（始终开启，不受测试开关控制）

```typescript
// TypeScript 严格模式
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true
  }
}
```

```bash
# ESLint 检查 + Prettier 格式化
npx eslint src/ --ext .ts,.tsx
npx prettier --check src/
```

**收益**：零维护成本，捕获 30%+ 的 bug（类型错误、未使用变量、错误拼写）。

#### Unit Tests（纯函数/工具函数）

```typescript
// utils/price.ts
export function calculateDiscount(price: number, rate: number): number {
  return Math.round(price * (1 - rate) * 100) / 100
}

// utils/price.test.ts
test('calculateDiscount applies rate correctly', () => {
  expect(calculateDiscount(100, 0.2)).toBe(80)
})
test('calculateDiscount rounds to 2 decimals', () => {
  expect(calculateDiscount(100, 0.333)).toBe(66.7)
})
```

#### Integration Tests（重点层）

```typescript
// components/SearchPage.test.tsx
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'

test('user can search and see results', async () => {
  // Given: mock API
  mockSearchAPI.mockResolvedValue({ items: [{ id: 1, title: 'React Guide' }] })

  // When: user types and waits for debounce
  render(<SearchPage />)
  await userEvent.type(screen.getByRole('searchbox'), 'React')
  await waitFor(() => expect(screen.getByRole('list')).toBeInTheDocument())

  // Then: results are displayed
  expect(screen.getByText('React Guide')).toBeInTheDocument()
})
```

**原则：** 测试用户行为，不测试实现细节。不测 state、不测 props 传递、不测内部方法调用。

#### E2E Tests（极少）

```typescript
// e2e/search.spec.ts (Playwright)
test('search flow from homepage to result', async ({ page }) => {
  await page.goto('/')
  await page.fill('[role=searchbox]', 'React')
  await page.waitForSelector('[role=list]')
  await expect(page.locator('[role=listitem]')).toHaveCount(3)
})
```

**原则：** 只测"用户完成一个核心任务"的完整流程。登录→搜索→下单。不测边缘情况。

---

## Python/Go 后端：Testing Pyramid

```
        ┌───────┐
        │  E2E   │  ← 极少
        ├─────────┤
        │Integration│ ← 适量
        ├─────────┤
        │   Unit   │  ← 重点！大部分测试在这里
        └─────────┘
```

### Python (pytest)

```python
# Unit: tests/test_price_calculator.py
def test_discount_applied_correctly():
    result = calculate_discount(price=100, rate=0.2)
    assert result == 80.0

def test_negative_price_raises_error():
    with pytest.raises(ValueError, match="price must be positive"):
        calculate_discount(price=-10, rate=0.2)

# Integration: tests/test_items_api.py
async def test_create_item_success(async_client):
    response = await async_client.post(
        "/api/items",
        json={"name": "Test", "description": "A test item"},
        headers={"Authorization": f"Bearer {test_token}"}
    )
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Test"
    assert "id" in data

# 验证数据库写入
async def test_create_item_persists_to_db(async_client, db_session):
    response = await async_client.post("/api/items", json={...})
    item_id = response.json()["id"]
    db_item = await db_session.get(Item, item_id)
    assert db_item is not None
```

### Go (testing + testify)

```go
// Unit: price_calculator_test.go
func TestCalculateDiscount(t *testing.T) {
    result := CalculateDiscount(100, 0.2)
    assert.Equal(t, 80.0, result)
}

func TestNegativePriceReturnsError(t *testing.T) {
    _, err := CalculateDiscount(-10, 0.2)
    require.Error(t, err)
    assert.Contains(t, err.Error(), "price must be positive")
}

// Integration: items_handler_test.go
func TestCreateItem_Success(t *testing.T) {
    router := setupTestRouter()
    body := `{"name":"Test","description":"A test item"}`
    
    req := httptest.NewRequest("POST", "/api/items", strings.NewReader(body))
    req.Header.Set("Authorization", "Bearer "+testToken)
    req.Header.Set("Content-Type", "application/json")
    
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)
    
    assert.Equal(t, 201, w.Code)
    var resp map[string]interface{}
    json.Unmarshal(w.Body.Bytes(), &resp)
    assert.Equal(t, "Test", resp["name"])
}
```

---

## 测试分层决策流程

```
开始 Task 实现
    │
    ▼
是不是一次性/原型代码？
    │
    ├── 是 → 跳过所有测试
    │
    └── 否 → 静态分析（始终运行）
              │
              ▼
         涉及纯函数/工具函数？
              │
              ├── 是 → 写单元测试（默认）
              │
              └── 否 → 跳过单测
              │
              ▼
         涉及用户可见行为/API 契约？
              │
              ├── 是 → 写集成测试（默认）
              │
              └── 否 → 跳过集成测试
              │
              ▼
         Task 声明了 test: e2e？
              │
              ├── 是 → 写 E2E 测试
              │
              └── 否 → 跳过 E2E
```

---

## 测试运行命令参考

| 场景 | 命令 |
|------|------|
| React Unit/Integration | `npx vitest --run` |
| React E2E | `npx playwright test` |
| TypeScript 检查 | `npx tsc --noEmit` |
| ESLint | `npx eslint src/` |
| Python Unit | `pytest tests/ -v` |
| Python 类型检查 | `mypy src/` |
| Go Unit | `go test ./... -v` |
| Go 静态分析 | `golangci-lint run` |

---

## 常见错误

| 错误 | 正确做法 |
|------|----------|
| "每个组件都要写测试" | 分层策略：单测默认开，E2E 默认关，不一刀切 |
| 测试组件内部 state | 测试渲染结果和用户可见行为 |
| 一个测试覆盖多个行为 | 一个 test case 只验证一件事 |
| E2E 测边缘情况 | E2E 只测核心流程，边缘情况在 Unit/Integration 层覆盖 |
| mock 一切依赖 | 只 mock 外部 API 和副作用，不 mock 自己的模块 |
| 原型代码也写测试 | 一次性/原型代码明确跳过，不浪费时间 |
