# React 核心概念 (后端视角版)

作为一名后端开发者，你其实已经掌握了 React 最核心的心智模型。React 并不是什么魔法，它的本质就是一个**状态机**。

## 1. 核心公式: UI = f(State)

在后端开发中，我们习惯于编写**纯函数 (Pure Function)**：输入确定，输出就确定，没有副作用。

React 的组件 (Component) 本质上就是这样的函数：

- **输入**: State (内部状态) + Props (外部参数)
- **输出**: UI (通常是 HTML/DOM)

```java
// 后端类比 (Java伪代码)
public class UserCard {
    // 这里是 State
    private boolean isOnline;

    // render 就像是一个 pure function
    public View render(User user) {
        // user 是 Props (外部传进来的)
        if (this.isOnline) {
            return new View("<div>" + user.getName() + " (Online)</div>");
        } else {
            return new View("<div>" + user.getName() + " (Offline)</div>");
        }
    }
}
```

在 React (JavaScript) 中，这长这样：

```jsx
function UserCard({ user }) { // Props 作为参数传入
  const [isOnline, setIsOnline] = useState(false); // State 定义在函数内部

  if (isOnline) {
    return <div>{user.name} (Online)</div>;
  } else {
    return <div>{user.name} (Offline)</div>;
  }
}
```

**关键点**：你不需要手动去修改 DOM (比如 `document.getElementById('name').innerText = ...`)。你只需要修改 **State**，React 会自动重新调用这个函数，计算出新的 UI。

## 2. Props vs State

对于后端来说，区分这两个概念非常简单：

| 概念 | 后端类比 | 特性 |
| :--- | :--- | :--- |
| **Props** | **函数参数 (Arguments)** 或 **DTO** | **只读**。父组件传给子组件的数据。子组件无权修改 Props，就像你调用一个函数，函数内部不应该修改传入的参数引用。 |
| **State** | **类的实例变量 (Instance Variables)** | **可变**。组件自己维护的数据。当它改变时，触发组件的“重新渲染” (Re-render)。 |

## 3. 渲染机制: Virtual DOM 与 数据库事务

你可能会问：“每次 State 变了都重新运行函数生成 HTML，岂不是性能很差？”

这里 React 引入了 **Virtual DOM**。

### 核心流程：
1.  **Render 阶段 (计算)**:
    - 就像你在内存中构建一个巨大的 JSON 对象来描述当前的 UI 结构。
    - React 比较“旧的虚拟树”和“新的虚拟树”的差异 (Diff 算法)。
    - **类比**: 这就像是数据库事务中的 **Write Ahead Log (WAL)** 或者在内存中计算变更。

2.  **Commit 阶段 (提交)**:
    - React 将计算出的**差异 (Diff)** 应用到真实的浏览器 DOM 上。
    - **类比**: 这就是数据库的 **Commit** 操作。只提交真正改变了的数据，而不是重写整个表。

**结论**: 你可以放心地写“全量生成 UI”的代码，React 会自动帮你优化成“最小量更新 DOM”。

## 4. Hooks: 函数式组件的“外挂”

在以前 (Class Component 时代)，我们用类来写组件。现在推荐用函数 (Function Component)。
但是函数执行完就销毁了，怎么保存状态？怎么处理生命周期？

答案是 **Hooks**。Hooks 就像是给纯函数挂载了外部的“能力”。

### `useState`: 状态持久化
让函数组件拥有“记忆”。

```javascript
// 每次组件重绘，userCount 都会被保留，不会被重置为 0
const [userCount, setUserCount] = useState(0);
```

**后端类比**: 就像是 Spring Bean 的 Scope。虽然函数每次都运行，但 `useState` 帮你把数据存在了 React 的底层上下文 (Context) 中，类似 `ThreadLocal` 或者 Session 作用域。

### `useEffect`: 副作用处理 (Side Effects)
函数应该是纯的，但我们需要发 API 请求、监听 WebSocket、操作 LocalStorage。这些都是“副作用”。

`useEffect` 告诉 React：“在渲染完 UI **之后**，执行这段逻辑。”

```javascript
useEffect(() => {
  // 1. 逻辑体: 组件挂载(Mount)或依赖更新时执行
  const socket = connectToSocket();

  // 2. 清理函数 (Optional): 组件卸载(Unmount)时执行
  return () => {
    socket.disconnect(); // 类似于 finally 块或析构函数
  };
}, [serverId]); // 3. 依赖数组: 只有 serverId 变了，才重新执行
```

**后端类比**:
- 依赖数组 `[]` 空数组: 相当于构造函数 `Constructor` (只执行一次)。
- 依赖数组 `[var]` 有值: 相当于 `Observer` 模式，监听 `var` 的变化。
- 返回的清理函数: 相当于 `PreDestroy` 或 `defer` (Go)，用于资源释放。

