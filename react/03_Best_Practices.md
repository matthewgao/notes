# React 最佳实践与避坑指南 (后端视角版)

后端开发者转 React，最容易在思维模式上栽跟头。以下是 3 个最常见的坑，以及如何用“后端思维”去理解和避免。

## 1. 绝对不要直接修改 State (Immutability)

**错误写法 (Mutation):**
```javascript
const [list, setList] = useState(['A', 'B']);

// ❌ 错误: 直接修改了内存中的对象
list.push('C'); 
// React 比较引用发现 list 还是原来那个 list (内存地址没变)，所以认为数据没变，不会刷新 UI。
setList(list); 
```

**后端类比:**
想象你的 State 是一个 **Value Object (值对象)** 或者 Java 中的 `String` / `BigInteger`。它们是不可变的。如果你想改变它，你必须创建一个**新的对象**。

**正确写法:**
```javascript
// ✅ 正确: 创建一个新数组 (Spread Operator)
const newList = [...list, 'C']; 
// React 发现 newList 和 list 的内存地址不同，触发重绘。
setList(newList);
```

> **最佳实践**: 始终假设 State 是只读的 (Read-only)。

## 2. 拥抱“组合”而非“继承”

在后端，我们习惯用继承 (`extends BaseController`) 来复用逻辑。
在 React 中，**几乎从不使用继承**。我们使用 **组合 (Composition)**。

**场景**: 你想做一个通用的“带边框的容器”。

**❌ 继承思维 (Don't do this):**
```javascript
class BaseCard extends Component { ... }
class UserCard extends BaseCard { ... } // 复杂且脆弱
```

**✅ 组合思维 (Do this):**
使用 `children` 属性，类似于后端的“装饰器模式”或者“模版方法”。

```jsx
// 容器组件
function Card({ children }) {
  return <div className="border-2 p-4 rounded">{children}</div>;
}

// 具体使用
function UserPage() {
  return (
    <Card>
      <h1>用户信息的具体内容</h1>
      <p>这里的内容被“注入”到了 Card 内部。</p>
    </Card>
  );
}
```

## 3. 逻辑复用：Custom Hooks (自定义 Hook)

如果你的组件里写了太多逻辑 (API 请求、数据转换)，它就变成了一个臃肿的 "God Class"。
你应该把逻辑抽离出来，就像后端把逻辑抽离到 Service 层一样。

**Custom Hook 就是你的 Service 类。**

**场景**: 多个页面都需要获取当前用户信息。

**不推荐**: 在每个组件里都写一遍 `useEffect` 和 `fetch`。

**推荐 (Custom Hook):**

```javascript
// hooks/useUser.js
function useUser(userId) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // 模拟 Service 调用
    fetchUser(userId).then(data => {
      setUser(data);
      setLoading(false);
    });
  }, [userId]);

  // 返回数据和状态，不涉及任何 UI
  return { user, loading };
}
```

**在组件中使用:**

```jsx
function UserProfile() {
  // 一行代码引入业务逻辑，组件只负责渲染
  const { user, loading } = useUser(123);

  if (loading) return <div>Loading...</div>;
  return <div>{user.name}</div>;
}
```

## 4. 依赖数组陷阱 (The Dependency Array)

在 `useEffect` 中，依赖数组 `[]` 决定了副作用什么时候执行。

**后端类比**: 这就像数据库触发器 (Trigger) 的 `WHEN` 条件。

- **漏写依赖**: 就像使用了过期的缓存。React 会使用旧闭包中的变量值。
- **对象作为依赖**:
  ```javascript
  // ❌ 危险: {} === {} 是 false。
  // 每次 Render 都会生成新的 params 对象，导致 useEffect 死循环不断执行。
  const params = { id: 1 };
  useEffect(() => { ... }, [params]); 
  ```
  **解决**: 确保依赖项是基本类型 (String, Number, Boolean)，或者使用 `useMemo` 缓存对象引用。

## 5. 什么时候该用全局状态 (Redux/Context)?

不要什么都往全局状态 (Redux) 里塞。

- **后端类比**: Redux 就像是 **Redis** 或 **全局单例**。
- **State**: 就像是 **局部变量**。

如果数据只在当前页面或当前组件树的一个小分支使用，用 `useState` 就够了。
只有当“用户登录信息”、“全局主题配置”、“购物车数据”这种跨越整个应用的数据，才适合放进全局状态。

