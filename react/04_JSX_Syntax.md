# JSX 语法详解 (后端视角版)

React 代码里那些看起来像 HTML 但又混在 JavaScript 里的东西，叫 **JSX (JavaScript XML)**。

对于后端开发者来说，最直观的理解是：**JSX 就像是更强大的 JSP / Thymeleaf / Freemarker 模版，但它是直接写在 Java/Go 代码里的。**

## 1. JSX 的本质：它不是字符串

在 JSP 中，HTML 只是单纯的字符串拼接。
但在 React 中，JSX **最终会被编译成普通的 JavaScript 函数调用**。

```jsx
// 开发者写的 JSX
const element = <h1 className="greeting">Hello, world!</h1>;
```

**编译后的代码 (Babel 转译):**
```javascript
// 就像是构建一个 DOM 节点对象
const element = React.createElement(
  'h1',
  { className: 'greeting' },
  'Hello, world!'
);
```

> **后端类比**: 这就像是你在用 Java 的 `StringBuilder` 或者 DOM API (`DocumentBuilder`) 在内存中构建 XML 对象，而不是简单的 String Concat。

## 2. 核心规则：只能有一个根节点

函数只能返回一个值。既然 JSX 编译后是函数调用，那么它也必须返回**一个**对象。

**❌ 错误:**
```jsx
return (
  <h1>标题</h1>
  <p>内容</p>
);
// 编译报错：相当于 return obj1, obj2; 这是不合法的 JS 语法。
```

**✅ 正确:**
用一个 `div` 或者 React Fragment (`<>...</>`) 包裹起来。

```jsx
return (
  <> 
    <h1>标题</h1>
    <p>内容</p>
  </>
);
// Fragment 就像是一个“虚拟容器”，它在渲染到浏览器时不会产生额外的 <div> 标签。
```

## 3. 插值表达式：`{}` 就是你的 `<%= %>`

在 JSX 中，只要遇到大括号 `{ ... }`，就意味着**“切换回 JavaScript 模式”**。你可以在里面写任何**有返回值**的 JS 表达式。

| 功能 | JSX 写法 | 后端模版类比 (JSP/Thymeleaf) |
| :--- | :--- | :--- |
| **变量输出** | `<h1>{user.name}</h1>` | `${user.name}` |
| **方法调用** | `<p>{formatDate(new Date())}</p>` | `${#dates.format(date)}` |
| **运算** | `<div>Score: {a + b}</div>` | `${a + b}` |

> **注意**: 不能在 `{}` 里写 `if` 语句或 `for` 循环（因为它们是语句，没有返回值）。你需要用三元运算符或 `map`。

## 4. 样式处理：`className` 与 `style`

因为 JSX 本质是 JS，而 `class` 是 JS 的保留关键字（定义类用的），所以：

1.  **CSS 类名**: 使用 `className` 而不是 `class`。
    ```jsx
    <div className="container">...</div>
    ```

2.  **内联样式**: `style` 接受一个**对象 (Map)**，而不是字符串。属性名要用**驼峰命名 (camelCase)**。
    ```jsx
    // CSS: background-color: red; margin-top: 10px;
    // JSX:
    <div style={{ backgroundColor: 'red', marginTop: '10px' }}>
      Error!
    </div>
    ```
    > **注意**:这里有两个大括号。外层 `{}` 代表进入 JS 模式，内层 `{}` 代表这是一个 JS 对象。

## 5. 条件渲染：告别 `if-else`

在 JSX 内部（return 语句里），你不能写 `if`。我们常用以下两种方式：

### A. 三元运算符 (Ternary) - 类似 `if-else`
```jsx
<div>
  {isLoggedIn ? (
    <UserPanel />
  ) : (
    <LoginButton />
  )}
</div>
```

### B. 逻辑与运算符 (&&) - 类似 `if` (无 else)
这是 JS 的一个特性：如果前面为 `true`，则返回后面的值；如果为 `false`，则直接忽略。

```jsx
<div>
  {hasError && <p className="error">出错了！</p>}
</div>
```

## 6. 列表渲染：`map` 就是 `foreach`

不要试图在 JSX 里写 `for` 循环。React 习惯用数组的函数式方法 `map`。

**后端类比**: 就像 Java Stream API 的 `.map()`.

```jsx
const users = [{id: 1, name: 'Alice'}, {id: 2, name: 'Bob'}];

return (
  <ul>
    {/* Java: users.stream().map(user -> <li>...</li>).collect(Collectors.toList()) */}
    {users.map(user => (
      <li key={user.id}>
        {user.name}
      </li>
    ))}
  </ul>
);
```

> **关于 `key`**: 你必须给循环生成的每个元素一个唯一的 `key` 属性（通常是数据库 ID）。
> **原因**: React 需要这个 key 来优化 Diff 算法。如果列表顺序变了，React 通过 ID 就能复用 DOM 节点，而不是销毁重建。这就像数据库的主键。

## 7. 闭合标签

在 HTML 中，有些标签可以不闭合（如 `<input>`, `<br>`）。
在 JSX 中，**所有标签必须闭合**。

- **❌ 错误**: `<input type="text">`
- **✅ 正确**: `<input type="text" />` (自闭合) 或 `<div></div>`

