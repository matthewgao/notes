# React 路由管理 (后端视角版)

在传统后端开发 (Spring MVC / Django) 中，路由是由后端服务器控制的。用户访问 `/about`，服务器匹配 `@GetMapping("/about")`，然后渲染 `about.html` 返回给浏览器。

在 React 单页应用 (SPA) 中，路由逻辑被**移交到了前端 (浏览器)**。
服务器只负责返回同一个 `index.html`，而“页面跳转”实际上是 JavaScript 在**动态替换 DOM 节点**，并修改浏览器地址栏的 URL (利用 History API)，而**不触发真正的页面刷新**。

目前 React 生态中最主流的路由库是 **React Router (v6)**。

## 1. 核心概念映射

| 概念 | 后端类比 (Spring MVC) | React Router 组件 | 作用 |
| :--- | :--- | :--- | :--- |
| **Router** | `DispatcherServlet` | `<BrowserRouter>` | 监听 URL 变化，分发请求。通常包裹在 App 最外层。 |
| **Route Config** | `@RequestMapping` / `Controller` | `<Routes>` & `<Route>` | 定义 "URL 路径" 与 "组件" 的对应关系。 |
| **Link** | `<a href="...">` (但不会刷新) | `<Link to="...">` | 声明式导航。点击后 URL 变了，但页面不白屏。 |
| **Navigate** | `response.sendRedirect(...)` | `useNavigate()` Hook | 编程式导航。在 JS 代码中手动触发跳转 (如登录成功后)。 |
| **Params** | `@PathVariable` | `useParams()` Hook | 获取 URL 中的动态参数 (如 `/user/:id`)。 |

## 2. 基本配置 (声明路由表)

通常在 `App.jsx` 或 `main.jsx` 中定义。

```jsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import Home from './pages/Home';
import About from './pages/About';
import UserDetail from './pages/UserDetail';

function App() {
  return (
    // 1. 启用路由功能 (DispatcherServlet 启动)
    <BrowserRouter>
      {/* 2. 定义路由表 (Controller Mappings) */}
      <Routes>
        {/* 当 URL 是 / 时，渲染 Home 组件 */}
        <Route path="/" element={<Home />} />
        
        {/* 当 URL 是 /about 时，渲染 About 组件 */}
        <Route path="/about" element={<About />} />
        
        {/* 动态路由: :id 是占位符，类似 @PathVariable("id") */}
        <Route path="/user/:id" element={<UserDetail />} />
        
        {/* 404 处理: * 匹配所有剩余路径 */}
        <Route path="*" element={<div>404 Not Found</div>} />
      </Routes>
    </BrowserRouter>
  );
}
```

## 3. 页面跳转 (Navigation)

### A. 声明式跳转 (点击链接)
**不要使用 `<a>` 标签！** `<a>` 标签会导致浏览器向服务器重新发起请求，页面会刷新，React 状态会丢失。

**✅ 使用 `<Link>` 或 `<NavLink>`:**

```jsx
import { Link } from 'react-router-dom';

function NavBar() {
  return (
    <nav>
      {/* 渲染出来就是 <a href="/about">...</a>，但拦截了点击事件 */}
      <Link to="/about">关于我们</Link>
      
      <Link to="/user/123">查看用户 123</Link>
    </nav>
  );
}
```

### B. 编程式跳转 (JS 代码控制)
**场景**: 用户点击“登录”按钮，API 请求成功后，自动跳到首页。
**后端类比**: `response.sendRedirect("/dashboard")`

使用 `useNavigate` Hook。

```jsx
import { useNavigate } from 'react-router-dom';

function LoginPage() {
  const navigate = useNavigate(); // 获取导航对象

  const handleLogin = async () => {
    // 1. 调用后端 API
    await loginApi(); 
    
    // 2. 跳转到 Dashboard
    // navigate('/dashboard'); // 简单跳转
    navigate('/dashboard', { replace: true }); // 替换当前历史记录 (禁止后退)
  };

  return <button onClick={handleLogin}>登录</button>;
}
```

## 4. 获取参数 (URL Params & Query)

### A. 路径参数 (`/user/:id`)
**后端类比**: `@GetMapping("/user/{id}")` + `@PathVariable String id`

```jsx
// 路由定义: <Route path="/user/:id" element={<UserDetail />} />

import { useParams } from 'react-router-dom';

function UserDetail() {
  // 自动解析 URL 中的参数
  const { id } = useParams(); 
  
  return <h1>正在查看用户 ID: {id}</h1>;
}
```

### B. 查询参数 (`/search?q=react`)
**后端类比**: `@RequestParam("q") String query`

使用 `useSearchParams` Hook (用法类似 `useState`)。

```jsx
import { useSearchParams } from 'react-router-dom';

function SearchPage() {
  const [searchParams, setSearchParams] = useSearchParams();
  
  // 获取参数 ?q=...
  const query = searchParams.get('q'); 

  return (
    <div>
      <p>搜索关键词: {query}</p>
      {/* 修改查询参数，页面 URL 会自动变更为 ?q=vue */}
      <button onClick={() => setSearchParams({ q: 'vue' })}>
        搜 Vue
      </button>
    </div>
  );
}
```

## 5. 嵌套路由 (Nested Routes)

这是后端路由通常没有的概念。React 允许路由**嵌套**，就像 UI 布局嵌套一样。
比如 `/admin/users` 和 `/admin/settings`，它们共享同一个侧边栏和顶部导航，只有中间内容区在变。

```jsx
// App.jsx
<Route path="/admin" element={<AdminLayout />}>
  {/* 子路由路径是相对的 */}
  <Route path="users" element={<UserList />} />
  <Route path="settings" element={<Settings />} />
</Route>
```

```jsx
// AdminLayout.jsx
import { Outlet } from 'react-router-dom';

function AdminLayout() {
  return (
    <div className="admin-panel">
      <Sidebar /> {/* 侧边栏常驻 */}
      
      <main>
        {/* Outlet 相当于 "子路由的占位符" */}
        {/* 类似 JSP 的 <jsp:include> 插槽 */}
        <Outlet /> 
      </main>
    </div>
  );
}
```

