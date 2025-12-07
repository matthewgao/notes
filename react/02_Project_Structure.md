# React 工程结构解析 (后端视角版)

打开一个标准的 React 工程 (通常由 Vite 或 Create-React-App 创建)，你会看到很多文件。别慌，我们用后端 MVC/分层架构的视角来拆解它。

## 1. 顶层文件映射

| 文件/目录 | 后端类比 | 作用说明 |
| :--- | :--- | :--- |
| `package.json` | `pom.xml` (Maven) / `go.mod` (Go) | 依赖管理、启动脚本定义。 |
| `vite.config.js` | `application.yml` / `webpack.config` | 构建工具配置。控制编译、代理 (Proxy) 等行为。 |
| `index.html` | **唯一的 View 模板** | 整个单页应用 (SPA) 的宿主。里面通常只有一个空的 `<div id="root"></div>`。 |
| `src/main.jsx` | **`public static void main(String[] args)`** | **程序入口**。它的唯一作用就是找到 ID 为 `root` 的 div，把 React 应用“挂载”上去。 |
| `src/App.jsx` | **Root Controller** | 根组件。通常用于配置路由 (Router)、全局 Layout、全局 Provider (Context)。 |

## 2. `src` 目录结构建议

虽然 React 没有强制规定结构，但为了可维护性，通常推荐以下分层 (类似 MVC)：

```text
src/
├── assets/           # 静态资源 (Images, Global CSS)
├── components/       # 【通用 UI 组件库】 (Button, Modal, Card)
│   └── ...           # 类比: 通用的 View 模版片段，不包含复杂业务逻辑
├── pages/            # 【页面级组件】 (Login, Dashboard)
│   └── ...           # 类比: Controller 的各种 Action，聚合多个组件
├── hooks/            # 【业务逻辑层】 (useAuth, useProductList)
│   └── ...           # 类比: Service 层 / Business Logic。负责数据获取、状态处理。
├── services/         # 【数据访问层】 (api.js, axios配置)
│   └── ...           # 类比: DAO / Repository 层。只负责发 HTTP 请求，不处理业务。
├── utils/            # 【工具类】 (formatDate, validators)
│   └── ...           # 类比: Util 类。
└── store/            # 【全局状态管理】 (Redux/Zustand)
    └── ...           # 类比: 内存数据库 / Redis。存放跨页面共享的数据。
```

## 3. 数据流向 (Data Flow)

在后端，数据流通常是：`Controller -> Service -> DAO -> DB`。

在 React 中，数据流是 **单向** 的 (Top-Down)：

1.  **Parent (Page)**:
    - 拥有数据 (State)。
    - 将数据作为 **Props** 传递给 Child。
    - 将“修改数据的函数”作为 **Callback** 传递给 Child。

2.  **Child (Component)**:
    - 接收 Props 用于展示。
    - 发生交互时 (Click)，调用父组件传下来的 Callback。

```jsx
// 类比: Parent 就像 Controller
function Dashboard() {
    const [user, setUser] = useState(null); // Controller 持有数据

    const handleLogin = (newUser) => {
        setUser(newUser); // Controller 定义如何修改数据
    }

    // 传递数据(user) 和 行为(onLogin) 给子组件
    return <LoginForm user={user} onLogin={handleLogin} />;
}

// 类比: Child 就像 View
function LoginForm({ user, onLogin }) {
    // 触发 Action
    return <button onClick={() => onLogin({name: 'Admin'})}>Login</button>;
}
```

## 4. 构建与编译 (Build Process)

后端代码需要编译成 Bytecode 或 Binary。前端代码也不例外。

- **源码**: `.jsx`, `.ts`, `.scss` (浏览器看不懂)
- **编译器 (Vite/Webpack)**: 负责“翻译”和“打包”。
- **产物 (`dist/` 目录)**: `.html`, `.js`, `.css` (浏览器能看懂的原生代码)。

当你运行 `npm run build` 时，就是在执行编译过程。最终生成的 `dist` 文件夹，你可以把它扔到 Nginx、S3 或者任何静态文件服务器上，**它不需要 Node.js 环境即可运行** (因为它已经是纯静态文件了)。

