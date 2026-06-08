# CLAUDE.md - FastJetpack 项目指南

## 项目描述

FastJetpack 是一个 Android Jetpack 架构快速开发框架，核心目标是封装 Retrofit + Kotlin 协程，提供三种不同耦合度的网络请求方案，减少模板代码，加速开发。项目基于 Google 最新架构文档，推荐使用 dev 分支（Flow 流式编程方案）。

- 作者：ldlywt (wutao)
- 调试 API：wanandroid.com
- 包名：`com.aisier`（基础库）/ `com.fastjetpack`（App）

## 技术栈

| 类别 | 技术 |
|------|------|
| 语言 | Kotlin 1.7.10 |
| 构建 | Gradle 7.2.1, AGP 7.2.1 |
| SDK | compileSdk 32, minSdk 21 (库) / 29 (App), targetSdk 32 |
| 网络 | Retrofit 2.9.0 + OkHttp 4.1.0 + Gson 2.8.6 |
| 异步 | Kotlin Coroutines + Flow |
| 架构 | MVVM (ViewModel + LiveData/StateFlow) |
| UI | ViewBinding, Navigation 2.3.5, Material 1.6.1 |
| 生命周期 | Lifecycle 2.4.0 (ViewModel, LiveData, Runtime KTX) |
| 事件总线 | LiveEventBus 1.8.0 |
| 主题切换 | libraryTheme (自研 Colorful 库) |
| ViewBinding 增强 | ViewBindingKTX 2.0.2 |
| 内存泄漏检测 | LeakCanary 2.7 (debug) |

## 项目结构

```
FastJetpack/
├── app/                    # 应用主模块
│   └── src/main/java/com/aisier/
│       ├── App.kt          # Application，初始化主题配置
│       ├── bean/           # 数据实体类 (User, WxArticleBean)
│       ├── net/            # 网络层 (ApiService, RetrofitClient, Repository)
│       ├── ui/             # 界面层 (Activity, Fragment)
│       └── vm/             # ViewModel 层
├── base/                   # 基础架构模块（核心）
│   └── src/main/java/com/aisier/architecture/
│       ├── anno/           # 注解 (ActivityConfiguration, FragmentConfiguration)
│       ├── base/           # 基类 (BaseApp, BaseActivity, BaseFragment, BaseViewModel, IUiView)
│       ├── ktx/            # Kotlin 扩展函数 (Context, EditText, Resource, Thread, View)
│       └── util/           # 工具类 (FlowKtx 核心, SingleLiveEvent)
├── network/                # 网络封装模块
│   └── src/main/java/com/aisier/network/
│       ├── base/           # BaseRepository, BaseRetrofitClient
│       ├── entity/         # ApiResponse, HttpError
│       ├── ResultBuilder.kt    # 链式结果处理构建器
│       ├── StateLiveData.kt    # StateLiveData 类型别名与 observeState 扩展
│       └── Utils.kt            # Toast 等工具方法
├── libraryTheme/           # 主题切换模块 (Colorful)
│   └── src/main/java/com/ldlywt/colorful/
│       ├── ColorfulDelegate.kt # 主题代理
│       ├── ColorTheme.kt       # 主题初始化入口
│       ├── ColorThemeConfig.kt # 主题配置
│       ├── ThemeEditor.kt      # 主题编辑持久化
│       └── ThemeStyle.kt       # 主题样式枚举
├── basic.gradle            # 公共 library 模块配置
├── common.gradle           # 公共 Android 配置
├── config.gradle           # 依赖版本统一管理
└── settings.gradle         # 模块声明 (app, base, network, libraryTheme)
```

## 模块依赖关系

```
app --> base --> network
             --> libraryTheme
```

- `app` 依赖 `base`
- `base` 依赖 `network` 和 `libraryTheme`（均通过 `api` 传递依赖）
- `network` 依赖 Retrofit/OkHttp/Gson/LiveEventBus

## 构建说明

```bash
# 构建 Debug APK
./gradlew assembleDebug

# 构建 Release APK
./gradlew assembleRelease

# 清理构建
./gradlew clean
```

依赖版本统一在 `config.gradle` 中管理，各模块通过 `rootProject.ext.xxx` 引用。

## Jetpack 组件封装说明

### 1. 网络请求封装（三层架构）

**ApiResponse 统一响应体** (`network/entity/ApiResponse.kt`)
- `ApiResponse<T>`：基类，包含 data/errorCode/errorMsg，通过 `errorCode == 0` 判断成功
- `ApiSuccessResponse<T>`：成功响应
- `ApiEmptyResponse<T>`：数据为空响应
- `ApiFailedResponse<T>`：业务逻辑失败响应
- `ApiErrorResponse<T>`：异常响应

**BaseRepository** (`network/base/BaseRepository.kt`)
- 提供 `executeHttp()` 挂起函数，统一处理网络请求的成功/失败/异常
- 子类直接调用 `executeHttp { service.xxx() }` 即可

**BaseRetrofitClient** (`network/base/BaseRetrofitClient.kt`)
- 封装 OkHttpClient 和 Retrofit 创建逻辑
- 子类实现 `handleBuilder()` 添加自定义拦截器
- 提供 `getService()` 泛型方法获取 API 接口实例

### 2. 结果处理方式（三种模式）

**模式一：StateLiveData + observeState**（适合需要数据持久化的场景）
```kotlin
// ViewModel 中
val wxArticleLiveData = StateMutableLiveData<List<WxArticleBean>>()

// Fragment 中
mViewModel.wxArticleLiveData.observeState(this) {
    onSuccess = { /* 处理成功 */ }
    onFailed = { code, msg -> /* 处理失败 */ }
    onError = { e -> /* 处理异常 */ }
    onComplete = { /* 完成回调 */ }
}
```

**模式二：StateFlow + collectIn**（推荐，响应式）
```kotlin
// ViewModel 中
private val _uiState = MutableStateFlow<ApiResponse<T>>(ApiResponse())
val uiState: StateFlow<ApiResponse<T>> = _uiState.asStateFlow()

// Fragment 中
mViewModel.uiState.collectIn(this, Lifecycle.State.STARTED) {
    onSuccess = { /* 处理成功 */ }
    onFailed = { code, msg -> /* 处理失败 */ }
}
```

**模式三：launchWithLoadingAndCollect**（链式调用，无需声明 LiveData/Flow）
```kotlin
launchWithLoadingAndCollect({
    mViewModel.login("user", "pass")
}) {
    onSuccess = { /* 处理成功 */ }
    onFailed = { code, msg -> /* 处理失败 */ }
}
```

### 3. Flow 扩展工具 (`base/util/FlowKtx.kt`)

- `launchFlow()`：将挂起函数包装为 Flow，支持 onStart/onCompletion 回调
- `launchWithLoading()`：带 Loading 的简单请求，不返回数据
- `launchAndCollect()`：不带 Loading 的请求 + 结果收集
- `launchWithLoadingAndCollect()`：带 Loading 的请求 + 结果收集
- `collectIn()`：Flow 在指定生命周期状态下的安全收集

### 4. BaseActivity / BaseFragment

- 实现 `IUiView` 接口，提供 `showLoading()` / `dismissLoading()`
- BaseActivity 通过 LiveEventBus 监听全局 Toast 事件
- BaseFragment 支持 `@FragmentConfiguration` 注解配置
- 均使用 ViewBinding（通过 ViewBindingKTX 委托简化绑定）

### 5. BaseViewModel

- 继承自 `ViewModel()`，提供 `ViewEffect` sealed class（MVI 方向预留）
- 子类中通过 Repository 发起网络请求，操作 StateFlow/LiveData

### 6. 主题切换 (libraryTheme)

- 通过 `initColorful(app, config)` 在 Application 中初始化
- 支持暗色模式、半透明、自定义主题色
- 主题配置通过 SharedPreferences 持久化
- BaseActivity.onCreate 中自动应用主题

## 新增网络请求的标准流程

1. 在 `ApiService` 中定义接口方法，返回 `ApiResponse<T>`
2. 在 Repository 中继承 `BaseRepository`，调用 `executeHttp { service.xxx() }`
3. 在 ViewModel 中持有 Repository 实例，操作 StateFlow 或直接返回结果
4. 在 Fragment/Activity 中使用 `collectIn` 或 `launchWithLoadingAndCollect` 处理结果

## 注意事项

- dev 分支使用 Flow 替代 LiveData，符合 Google 最新架构推荐
- `network` 模块中的 `BaseRepository.executeHttp()` 包含 `delay(500)` 测试用延时，正式环境需移除
- `config.gradle` 中定义的依赖版本是项目基准，修改版本号需在此文件统一调整
- ViewBinding 已全局启用，不要使用 `findViewById`
