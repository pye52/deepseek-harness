# WebCross 整合修改记录

本文集中记录本地开发分支为 WebCross 整合保留的源码调整及其原因。现有包说明、双语文档、翻译一致性记录和单元测试保持当前提交版本，整合说明不再分散写入这些文件。本文不替代上游使用手册，也不表示整合已完成发行验收。

## 整合目标与职责

DSH 后端运行在 WebCross 现有 User 主进程中，客户端挂载到工作台已有渲染进程，避免增加专门托管 DSH 的常驻进程。会话、模型调用和交互继续复用 DSH；宿主生命周期、IPC、凭据加密、界面裁剪和主题映射由 WebCross 的 `packages/harness` 承担。`abcp-dsh` 保持独立的 WebCross 浏览器插件。

当前 WebCross 只在开发环境启用该整合；打包版继续展示原占位面板。这项开关属于 WebCross，不写入 DSH 的通用启动逻辑。

## 源码调整与原因

### 宿主管理启动与退出

[profile-boot.ts](apps/cli/src/profile-boot.ts) 为 `runProfile` 增加可选的 `host` 参数，接收宿主提供的 `home`、`cwd`、`requestExit` 和 `prepare`。配置档仍通过现有启动流程装载；宿主可在配置档条目挂载前注入服务，WebCross 用它注入加密凭据适配。

独立应用会自行管理进程信号、全局代理、致命错误和退出；这些行为进入共享进程后会影响 WebCross。因此嵌入模式不安装这些进程级处理，将退出请求交给宿主，关闭配置档时仅释放 DSH 所持资源。未提供 `host` 的启动路径保留原有处理。

嵌入模式还使用 `PluginPackages` 已有的 Node 默认包查找方式，不启用进程级模块解析拦截。原因是原解析机制未能匹配当前 WebCross 定制 Electron 的运行时指纹；该限制不表示 DSH 只能使用官方 Electron，也与浏览器反指纹功能无关。宿主须提前准备可解析的固定插件集；WebCross 配置档关闭运行时插件安装和热重载。具体原生扩展的兼容性仍需单独验证。

### 数据目录由配置档提供

[app-boot/src/index.ts](packages/boot/app-boot/src/index.ts) 中的 `dshHomePath` 和 `dshPath` 优先使用配置档的 `home`，未提供配置档时使用原默认目录。`dshPath` 调用方显式传入 `dshHome` 时保留该目录。两种解析方式都无需修改全局环境变量；插件配置可以使用宿主指定的数据空间。

WebCross 为嵌入配置档提供独立的数据目录和工作目录，避免与独立安装的 DSH 默认目录混用。插件自有的目录配置仍由 WebCross 配置档负责。

### 客户端启动与全局样式分离

[embedded.ts](packages/client/web/src/embedded.ts) 新增 `./embedded` 导入入口，复用 `AppWebEntry` 和 `applyIndexInjections`，不自动导入独立应用的基础样式。[boot.ts](packages/client/web/src/boot.ts) 不再承担全局样式导入；[默认入口](packages/client/web/src/index.ts) 继续导入 `base.css`，保留独立应用的样式行为。[包导出](packages/client/web/package.json) 与[构建入口](packages/client/web/tsdown.config.ts) 同步包含新增入口。

原因是独立应用的基础样式会作用于 `html`、`body`、字体及窗口拖动区域，直接导入会影响 WebCross 整个工作台。嵌入入口共享客户端启动、失败处理和释放机制；文档样式、传输接线和页面组合由宿主负责。该入口并不自动将全部 DSH 样式隔离到局部容器。

## 取舍与后续维护

没有复制会话组件或完整嵌入独立应用外壳：复制会增加后续同步成本，完整外壳则不满足工作台的精简需求。当前复用 DSH 插件、配置档、插槽和客户端生命周期，仅补充宿主启动和样式入口。

没有为当前定制 Electron 扩展原解析器的运行时指纹适配：当前目标是装载预先准备的插件集，Node 默认查找足以满足该目标。若以后开放运行时插件管理，需要重新验证插件安装、解析、重载和释放流程。

WebCross 专属布局、主题、传输与运行环境开关继续留在 WebCross 仓库。后续同步上游时，重点检查宿主启动参数、数据路径服务和客户端导出是否仍可复用，避免将专属行为扩散到会话核心。

## 验证范围

[resolved-profile-boot.spec.ts](apps/cli/tests/resolved-profile-boot.spec.ts) 保持上游原有内容，覆盖共享配置档启动、运行时解析和关闭；不在 DSH 原有单元测试中增加 WebCross 嵌入场景。整合验证由 WebCross 仓库的 Harness 运行时测试及开发环境检查承担。

此前开发验证已在定制 Electron 44.9.3 / Node 24.22.0 中完成配置档启动和空会话渲染；工作台资源协议注册顺序的修复位于 WebCross 仓库。真实模型任务、人工接管、完整应用启动组合和跨平台发行仍不属于已完成验收。文档与单元测试的清理不改变功能源码，也不重新声明这些运行验证通过。
