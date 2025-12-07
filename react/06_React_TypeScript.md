# React 与 TypeScript 的关系 (后端视角版)

## 1. 它们是什么关系？

对于后端开发者（尤其是习惯了 Java/Go/C# 的人）来说，JavaScript (JS) 最让人抓狂的地方就是**弱类型**和**运行时报错**。

- **JavaScript**: 就像 **Python** 或 **PHP**。变量没类型，传参传错了不知道，拼写错了直到运行那一行才崩。
- **TypeScript (TS)**: 就像 **Java** 或 **Go**。它给 JS 加上了强类型系统和编译时检查。

**React 和 TypeScript 是绝配。**
现在几乎所有的企业级 React 项目**默认都会使用 TypeScript**。

> **比喻**: 
> - 写 React + JS 就像在用 `Map<String, Object>` 到处传数据，虽灵活但很容易空指针。
> - 写 React + TS 就像在用 `UserDTO`，编译器会告诉你哪个字段必须填，哪个字段可能是 null。

## 2. 为什么后端更应该用 TS 写 React？

1.  **代码提示 (IntelliSense)**:
    在 JS 里，你输入 `props.`，IDE 不知道后面有什么。
    在 TS 里，你输入 `props.`，IDE 会直接弹窗提示 `.userId`、`.userName`，就像在写 Java 类一样爽。

2.  **重构信心**:
    你想修改一个 Props 的名字。
    - **JS**: 全局搜索字符串替换，祈祷别漏改，改错了运行时才会挂。
    - **TS**: 右键 -> Rename，IDE 自动把所有引用的地方都改了，如果有遗漏编译直接报错。

3.  **自文档化**:
    不需要看文档就知道这个组件需要传什么参数，哪个是必填的。

## 3. 核心用法对比

### A. 定义组件 Props (类似 DTO)

**JavaScript:**
```jsx
// ❌ 参数 user 是什么结构？不知道。必须看代码内部或者猜。
function UserCard({ user, onEdit }) { ... }
```

**TypeScript:**
```tsx
// ✅ 定义一个 Interface，就像定义一个 Java Class / POJO
interface User {
  id: number;
  name: string;
  email?: string; // ? 表示可选字段 (Optional)
}

// 定义 Props 接口
interface UserCardProps {
  user: User;
  onEdit: (id: number) => void; // 定义回调函数的签名
}

// 泛型写法: React.FC<Props>
function UserCard({ user, onEdit }: UserCardProps) {
  return (
    <div onClick={() => onEdit(user.id)}>
      {user.name}
    </div>
  );
}
```
如果你在父组件里少传了 `user`，或者 `onEdit` 传了个字符串，**TS 编译器会直接标红报错**，根本跑不起来。

### B. Hooks 的泛型 (Generics)

**useState:**
```tsx
// TS 通常能自动推断类型，但有时需要显式指定
const [count, setCount] = useState(0); // 推断为 number

// 显式指定泛型 (类似 List<User>)
const [user, setUser] = useState<User | null>(null); 
```

**useRef:**
```tsx
// 绑定 DOM 元素
const inputRef = useRef<HTMLInputElement>(null);

// 使用时，IDE 会提示 HTMLInputElement 所有的属性 (value, focus 等)
inputRef.current?.focus();
```

### C. 事件处理 (Event Handling)

在 JS 里，`event` 对象经常也是个黑盒。TS 提供了准确的类型。

```tsx
// ChangeEvent 是 React 提供的泛型接口
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  // IDE 知道 e.target.value 是 string
  console.log(e.target.value); 
};

return <input onChange={handleChange} />;
```

## 4. 常见类型定义速查表

| TS 类型 | 后端类比 | 场景 |
| :--- | :--- | :--- |
| `string` / `number` / `boolean` | `String` / `Integer` / `Boolean` | 基础类型 |
| `string[]` 或 `Array<string>` | `List<String>` | 数组 |
| `React.ReactNode` | `Object` (但在 React 上下文中) | **最常用的类型**。表示任何可以被渲染的东西 (组件、字符串、数字、HTML标签)。常用于 `children` 属性。 |
| `React.CSSProperties` | `Map<String, String>` | 用于 `style={{...}}` 属性的类型。 |
| `() => void` | `Runnable` / `void func()` | 不带参数、无返回值的回调函数。 |
| `(id: number) => void` | `Consumer<Integer>` | 带参数的回调。 |

## 5. 如何开始？

如果你使用 Vite 创建项目，直接选择 TypeScript 模版：

```bash
npm create vite@latest my-app -- --template react-ts
```

文件后缀名会有变化：
- `.js` -> `.ts` (纯逻辑文件)
- `.jsx` -> `.tsx` (包含 JSX 组件的文件)

**总结**：作为后端开发，**强烈建议**直接上手 TypeScript。它会大大减少你的“挫败感”，让你感觉像是在写强类型的后端代码一样舒适。

