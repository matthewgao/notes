# 编译与转译：React 如何支持 JS/TS/JSX/TSX (后端视角版)

你可能会好奇，浏览器明明只认识 `HTML`, `CSS` 和 `JavaScript`，它根本不懂什么 `TypeScript`, `JSX`, `TSX`。那么 React 是怎么跑起来的？

这背后其实发生了一场“欺骗”—— 或者说，一场精密的**编译 (Transpilation)** 过程。

## 1. 核心流程：从源码到产物

这与后端的编译过程非常相似：

| 阶段 | 后端 (Java) | 前端 (React) |
| :--- | :--- | :--- |
| **源码** | `.java` | `.tsx` / `.jsx` / `.ts` |
| **编译器** | `javac` | **Babel** / **SWC** / **TypeScript Compiler (tsc)** |
| **中间产物** | `.class` (Bytecode) | (无直接对应，通常在内存中) |
| **运行环境** | JVM | **浏览器 (JS 引擎)** |
| **最终产物** | 机器码 | **ES5 / ES6 JavaScript** (浏览器能看懂的原生 JS) |

### 关键点：浏览器只运行“原生 JS”

不管你写的是 `.jsx` 还是 `.tsx`，在浏览器真正执行前，它们全都被转换成了普通的 `.js` 文件。

- **JSX/TSX 中的标签** (`<div>`) 被转成了 `React.createElement('div')`。
- **TypeScript 的类型注解** (`: string`, `interface`) 被**直接剔除**（擦除）。

## 2. 工具链解密

### A. 构建工具 (Build Tool): Vite / Webpack
**类比**: Maven / Gradle

Vite 或 Webpack 是**管家**。它们负责协调整个编译流程：
1. 扫描你的项目文件。
2. 发现一个 `.tsx` 文件。
3. 把它丢给 **转译器 (Transpiler)** 去处理。
4. 将处理好的 JS 代码打包 (Bundle) 成一个或几个大文件 (`bundle.js`)。
5. 启动一个本地服务器 (Dev Server) 让你预览。

### B. 转译器 (Transpiler): Babel / SWC
**类比**: `javac` 的前端部分 (词法分析、语法分析、生成)

这是真正干活的“翻译官”。

1.  **处理 JSX**:
    ```jsx
    // 源码
    const element = <h1>Hello</h1>;
    ```
    ⬇️ **翻译后**
    ```javascript
    // 产物
    const element = React.createElement("h1", null, "Hello");
    ```

2.  **处理 TypeScript**:
    Babel (或 SWC) 处理 TS 的方式非常简单粗暴：**它不检查类型，只负责把类型定义删掉**。
    
    ```tsx
    // 源码
    const add = (a: number, b: number): number => a + b;
    ```
    ⬇️ **翻译后** (类型被擦除)
    ```javascript
    // 产物
    const add = (a, b) => a + b;
    ```

    > **思考**: 那谁来检查类型错误？
    > 通常是 IDE (VS Code) 的插件，或者你在命令行运行 `tsc` 命令时进行检查。构建工具通常只负责“翻译”，以便让代码跑起来。

## 3. 为什么能“同时支持”？

因为对于构建工具 (Vite) 来说，配置非常灵活：

- 如果遇到 `.js` 文件 -> 直接打包。
- 如果遇到 `.jsx` 文件 -> 启用 "JSX 插件" 进行翻译。
- 如果遇到 `.ts` 文件 -> 启用 "TS 插件" 剔除类型。
- 如果遇到 `.tsx` 文件 -> 启用 "TS 插件" + "JSX 插件"。

这就像 Maven 可以同时编译 Java 和 Kotlin 代码一样，只要配置了相应的 Compiler Plugin。

## 4. 总结

React 并不是原生支持这些语法，而是依赖强大的**工程化工具链**。

1.  **你在写**: 高级语法 (JSX, TSX, TS)。
2.  **工具链 (Vite/Babel)**: 实时监视文件，一旦保存，立即**编译**。
3.  **浏览器**: 接收到的是编译后的、朴实无华的**原生 JavaScript**。

所以，JSX 和 TSX 本质上是**“编译时语法糖”**。它们只存在于你的源代码中，不存在于用户的浏览器里。

