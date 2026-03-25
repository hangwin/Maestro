# Maestro Web UI 自动化测试实现分析

基于仓库中的实现文件（如 `maestro-client/src/main/java/maestro/Maestro.kt`, `maestro-client/src/main/java/maestro/drivers/CdpWebDriver.kt`, `maestro-client/src/main/resources/maestro-web.js`, `maestro-web/src/main/kotlin/maestro/web/cdp/CdpClient.kt`, `maestro-web/src/main/kotlin/maestro/web/selenium/ChromeSeleniumFactory.kt`），汇总出 Maestro Web 端 UI 自动化的工作方式、依赖以及登录态处理策略。

## 1. Web 端 UI 自动化的实现原理
- **驱动链路**：CLI 选择 `chromium` 或 `platform: web` 时，`Maestro.web(...)` 会构建 `CdpWebDriver` 并在启动时创建带有指定窗口大小/无头模式的 ChromeDriver (`CdpWebDriver#createSeleniumDriver`)，随后保持同一浏览器会话贯穿整次运行。
- **DOM 采集与元素查询**：启动或每次取层级时，通过 CDP 将 `maestro-web.js` 注入页面；脚本从 `document.body` 递归构建节点树（携带 `text`/`bounds`/`resource-id` 等属性），支持 iframe 同源穿透、`<option>` 这类“合成”节点、Flutter Web 识别与 `queryCss` 选择器检索。
- **交互执行**：用户态动作（`tap`/`swipe`/`inputText`/`pressKey` 等）通过 Selenium 的指针动作或 JS（如 `tapOnSyntheticElement`）在当前 Chrome 窗口执行；滚动支持标准页面与 Flutter Web 动画滚动；截图走 CDP 的 `Page.captureScreenshot`，录屏使用 `WebScreenRecorder` + `JcodecVideoEncoder`。
- **窗口与设备能力**：驱动会监听新 window handle 并切换焦点；提供地理位置模拟 (`Emulation.setGeolocationOverride`)、CSS 查询、屏幕静态检测等能力，整体对外暴露为 `Capability.FAST_HIERARCHY` 的 Web 设备。

## 2. 仅做 Web 自动化是否需要 Java
- 需要。Maestro CLI/驱动基于 JVM，`Maestro.web` 启动前会校验 Java 版本（要求 11+）。即便只跑浏览器测试，也必须安装符合版本的 JRE/JDK 来运行 CLI 和驱动。
- 还需本机可用的 Chrome/Chromium 及由 Selenium 管理的 ChromeDriver（在 `CdpWebDriver` 和 `ChromeSeleniumFactory` 中直接构建 `ChromeDriver`，使用 Selenium 自带的驱动管理）。除浏览器与 Java 外，无需额外的前端框架依赖。

## 3. Web 自动化的登录态处理
- **默认保持**：单次 Maestro 会话内复用同一个 ChromeDriver 实例，浏览器的 cookies/localStorage/sessionStorage 会一直保留，因而完成一次登录后，后续步骤仍处于已登录状态。`launchApp` 仅加载 URL，不主动清理数据。
- **显式清除**：流程里使用 `clearState`（或 `launchApp: { clearState: true }`）会触发 `ClearStateCommand`，进而调用 `CdpWebDriver.clearAppState`。该方法解析目标域名后，通过 CDP 的 `Storage.clearDataForOrigin` 清除 cookies、本地存储和缓存，同时重置权限，从而让登录态回到初始状态。
- **可选注入**：`launchArguments` 会在脚本注入时写入全局作用域（`executeJS` 中循环设置），可借此传递 token/环境配置来脚本化地复现登录态；若要跨多次运行保留登录，需要外部复用浏览器用户数据目录或避免 `clearState`，否则新会话会得到干净环境。
