+++
title = 'Unity 移动端热更新框架之HybridCLR+Yooasset'
date = 2026-07-18T13:51:11+08:00
tags = ["Game Design", "technology"]
categories = ["technology"]
slug = "HybridCLR + Yooasset: A Hot-Update Framework for Unity Mobile"
math = true
+++

# Unity 移动端热更新架构设计：HybridCLR + YooAsset 流程闭环与全自动引用计数实践

在 Unity 移动端项目开发中，**代码热更新（HybridCLR）** 与 **资源生命周期管理（YooAsset）** 的组合是一套常见的技术选型。但在实际的工程落地中，如何处理开机引导、断网容错、资源加载解耦及内存自动卸载，是决定系统健壮性的关键。

本文分享一套经过重构优化的移动端加载与资源管理方案。该方案主要解决了以下几个核心问题：
1. **启动加载流的彻底解耦与无感化设计**：首屏开机引导与资源包加载解耦，避免在 AOT 层硬编码包名列表，实现加载 UI 的完全热更 [1.1.4, 1.2.1]。
2. **重试状态防御**：解决在弱网断连、重试网络请求时，资源管理系统二次初始化引发异常导致闪退的问题 [1.4.2, 2.2.2]。
3. **内存全自动引用计数卸载**：实现业务层与资源句柄（Handle）的完全解耦，规避手动释放导致内存泄漏的隐患 [2.4.3]。
4. **极简打包工作流**：将 C# 编译出的 DLL 二进制文件视为 Unity 默认的 `TextAsset` 统一使用 SBP（AssetBundle 编译管线）打包，避免混合管线构建的复杂度 [1.1.8, 1.3.5]。

---

## 一、 启动加载管线的解耦设计：从“单包耦合”到“BIOS / OS”引导架构

### 1. 痛点：加载界面的“热更新悖论”
在传统设计中，进度条 UI 通常直接挂载在内置的 AOT 场景（`BootScene`）中，由 AOT 层的 `Launcher` 负责更新和下载所有的资源包。

这在实际真机运行中会暴露两个弊端：
* **首屏加载界面无法热更新**：由于进度条和加载 UI 是硬编码在 AOT 场景里的，游戏一旦发布上线，无法通过热更新去调整启动加载界面的视觉表现 [1.1.4, 1.2.1]。
* **新增包裹必须重打 APK**：若在 AOT 代码中写死了需要更新的资源包裹列表，后期项目体量变大、需要新增独立包裹（如活动包）时，由于 AOT 层代码无法热更，必须重新打包发布 APK 才能让客户端识别该新包。

### 2. 设计思路：BIOS（引导）与 OS（业务内核）模式
为此，该系统借鉴了计算机开机引导的 **“BIOS & 操作系统”** 模式：

#### 阶段 A：BIOS 阶段（Launcher 引导）
`Launcher` 作为 AOT 层的引导程序，**其唯一的职责是更新并加载最基础的“系统代码包”（`DefaultPackage`）** [1.2.1]。
* **用户体验无感化**：在这个阶段，界面上**不展示任何进度条**。玩家开机只会看到精美的静态背景图和 Logo（随主包打入），配合一个旋转的小菊花，提示“正在配置环境...” [3]。
* **极速通过**：由于 `DefaultPackage` 里只存放编译后的二进制 DLL，在正常网络下，这个无感更新过程会在 **0.8 秒**内悄悄完成 [1.1.4, 1.2.1]。

#### 阶段 B：OS 阶段（HotUpdateEntry 接管）
`DefaultPackage`（代码包）更新并载入完毕后，`Launcher` 通过反射唤醒热更新主入口 `HotUpdateEntry.StartGame()` [1.2.1]。
* **业务内核接手**：此时，**最新版本的 C# 代码已经开始在内存中运行** [1.2.1]。由它去向服务器拉取最新的多包配置文件（`package_config.json`），并动态决定需要初始化、更新和下载哪些大型资源大包（如 `MainPackage`、`MapPackage`） [1.1.2, 1.2.1]。
* **无缝 UI 接力**：
  在 Unity 场景中，我们将进度条 Slider 默认设为隐藏（Deactive） [3]。
  因为 `BootScene` 属于内置场景，根据 HybridCLR 规则，热更新程序集的脚本不能挂载在其物体上。
  因此，在 `HotUpdateEntry` 启动时，使用 `canvasGo.transform.Find` 接口**穿透查找**这些隐藏的进度条和文本组件并将其激活（*注：直接使用 `GameObject.Find` 寻找隐藏物体会返回 `null`，而 `transform.Find` 可以穿透寻找并控制隐藏状态下的子物体*） [3]。
  随后，热更层直接操控这些组件，让进度条优雅地滑出，并平滑显示真正资源大包的下载进度（`0% -> 100%`） [1, 3]。

---

### 3. 技术路线：放弃 RawFile，拥抱 SBP 单包方案
在早期设计中，为了加载 `HotUpdate.dll` 二进制文件，通常会使用 `RawFileBuildPipeline`（原生文件构建管线） [1.1.4, 1.1.8]。但这会带来一个分包困局：
* 原生文件包管线**无法打包 `.prefab`** 这种 Unity 引擎特有的预制体 [1.1.4, 1.2.3]。
* 这导致必须同时维护 `CodePackage` (RawFile) 和 `DefaultPackage` (SBP) 两个包，`Launcher` 必须在开机阶段去同时初始化和更新两个包，网络请求和加载开销均翻倍 [1.3.6]。

**破局方案**：
**将重命名后的 `.dll.bytes` 直接视为 Unity 的常规 `TextAsset`（文本资源）** [1.1.8]。
由于 `TextAsset` 完全可以被常规的 `ScriptableBuildPipeline` (SBP) 兼容打包成标准的 AssetBundle。
将代码 `.bytes` 一并打入统一的 `DefaultPackage` 中 [1.1.8]。在 `Launcher.cs` 中，调用 `_package.LoadAssetAsync<TextAsset>()` 加载并提取 `textAsset.bytes` [1.1.8]。这一改动使 `Launcher` 重新回归到**只需要静默更新唯一包裹**的极简形态，并规避了复杂的混合管线配置 [1.1.8, 1.3.5]。

#### 🛠️ AOT 引导层核心反射代码片：

```csharp
private IEnumerator LoadHotUpdateDLL()
{
    if (_package == null) yield break;

    string assetPath = $"Assets/HotUpdateDlls/{hotUpdateDllName}.bytes";
    
    // 【最简 SBP 方案】：直接将 DLL 作为常规 TextAsset 统一从 SBP AssetBundle 中异步载入
    var handle = _package.LoadAssetAsync<TextAsset>(assetPath);
    yield return handle;

    if (handle.Status != EOperationStatus.Succeeded)
    {
        ShowError("加载游戏程序失败。");
        yield break;
    }

    TextAsset textAsset = handle.AssetObject as TextAsset;
    if (textAsset != null)
    {
        byte[] dllBytes = textAsset.bytes;
        Type entryType = null;
#if !UNITY_EDITOR
        Assembly hotUpdateAssembly = Assembly.Load(dllBytes);
        entryType = hotUpdateAssembly.GetType("HotUpdateEntry");
#else
        Assembly hotUpdateAss = System.AppDomain.CurrentDomain.GetAssemblies().First(a => a.GetName().Name == "HotUpdate");
        entryType = hotUpdateAss.GetType("HotUpdateEntry");
#endif
        if (entryType != null)
        {
            MethodInfo method = entryType.GetMethod("StartGame", BindingFlags.Public | BindingFlags.Static);
            if (method != null)
            {
                // 反射调用：不需要向热更层传递任何 UI 参数，AOT 启动器与 UI 逻辑彻底解耦
                method.Invoke(null, new object[] { nextSceneName, playMode, cdnUrl });
            }
        }
    }
    handle.Release();
}
```

---

## 二、 防御性状态机设计：彻底解决弱网重试时的初始化冲突

### 1. 痛点：重试异常与状态冲突
在调试弱网环境下的重试逻辑时，极易遇到以下两个报错：
`InvalidOperationException: Resource package 'DefaultPackage' is already initialized.`
`InvalidOperationException: YooAssets is already initialized.`

**底层成因剖析**：
当网络不稳定导致下载失败时，发生错误的节点往往是 **步骤 4（下载补丁文件）**。而在此之前，**步骤 1（初始化 `InitializePackageAsync`）已经在首轮运行中成功完成了** [1.4.2]。
当玩家在异常弹窗上点击“重试”时，若没有做好状态防御，`Launcher` 会重新启动并再次执行 `YooAssets.Initialize()` 和 `InitializePackageAsync()` [1.4.2, 2.2.2]。这会触犯 YooAsset 的底层安全机制，直接抛出异常。

### 2. 重构设计：双重状态防御（Defensive Programming）
为了给移动端断网重连和手动重试提供容错率，系统进行了状态拦截保护：
1. **全局驱动器拦截**：在最外层，先判定 `YooAssets.IsInitialized`。如果已经初始化完成，直接跳过 `YooAssets.Initialize()` [2.2.2]。
2. **安全实例持有**：先通过 `TryGetPackage` 确保 AOT 层的 `_package` 成员变量成功持有了引用（防止在后续状态判断中由于 `_package` 为 `null` 抛出空引用异常）。
3. **包裹状态拦截**：通过 `_package.InitializeStatus` 校验。如果其已经被置为 `EOperationStatus.Succeeded`，**直接安全地 `yield break` 退出初始化方法** [1.4.2, 1.4.5]。

#### 🛠️ 初始化阶段防御性状态机逻辑片：

```csharp
private IEnumerator InitYooAsset()
{
    // 防御 1：使用 IsInitialized 拦截保护全局驱动器，防止重复初始化报错 [2.2.2]
    if (!YooAssets.IsInitialized)
    {
        YooAssets.Initialize();
    }

    // 防御 2：确保 _package 变量不为空，防止重试时发生空引用异常崩溃
    YooAssets.TryGetPackage(packageName, out _package);
    _package ??= YooAssets.CreatePackage(packageName);

    // 防御 3：如果该包此前已初始化成功，直接跳过，防止底层抛出已初始化异常
    if (_package.InitializeStatus == EOperationStatus.Succeeded)
    {
        Debug.Log($"资源包 [{packageName}] 此前已完成初始化，重试时直接安全跳过。");
        yield break;
    }

    InitializePackageOptions initParameters = null;
    // ... 根据模式 (PlayMode) 组装参数 ...

    var initOperation = _package.InitializePackageAsync(initParameters);
    yield return initOperation;

    if (initOperation.Status != EOperationStatus.Succeeded)
    {
        ShowError($"初始化失败: {initOperation.Error}");
        yield break;
    }
}
```

---

## 三、 内存生命周期的“无感卸载”：全自动引用计数与多维生命周期托管

### 1. 痛点：手动 Release() 的繁琐与内存溢出风险
YooAsset 底层是引用计数（Reference Counting）机制 [2.4.3]。如果要求业务层在写 UI 或角色时，必须手动持有并维护 `AssetHandle`，并在对象销毁或切场景时调用 `handle.Release()`，会在工程协作中引入巨大隐患：
* **托管内存与物理内存的不对等**：
  在 Unity 中，AssetBundle 的物理数据驻留在 C++ 底层内存中，而 C# 层的 `AssetHandle` 仅仅是一个极轻量的引用包装 [2.4.3]。
  若开发人员忘记调用 `Release()`，即使 C# 层的垃圾回收（GC）被触发，由于 C++ 层的引用计数不为零，底层的 AssetBundle **永远不会被释放** [2.4.3]。随着贴图和模型堆积，手机就会因物理内存（RAM）爆满直接被系统强杀（闪退）。
* **业务逻辑强耦合**：UI 脚本、战斗脚本不得不大量引入 `YooAsset` 的命名空间和 `AssetHandle` 类型，违背了单一职责原则。

---

### 2. 演进与最优解：无感生命周期托管
为了实现“**写业务时无需关心资源释放，生命周期全自动托管**”，我们构建了一套多维度的自动内存清理机制。

同时，方案彻底抛弃了协程（Coroutine）和隐藏的 `CoroutineRunner` GameObject 挂载器 [1.2.6]。由于 YooAsset 3.x 底层原生支持了 **`async/await`** 异步任务驱动 [1.3.1]，我们直接采用原生的 Task 流。

* **原理剖析**：
  在 Unity 中，启动一个 Coroutine 协程会在底层（Heap）分配一个 `IEnumerator` 状态机对象，频繁调用会产生微额的 GC 堆分配。
  利用 C# 原生的 `async/await`，当资源句柄直接重写并支持了 `GetAwaiter()` 接口后，可以**直接 await 句柄对象** [1.3.1]。这不仅使得代码书写逻辑像同步一样直观，更做到了**完全不需要创建任何临时 GameObject 挂载器、纯 CPU 驱动的无残留异步加载** [1.2.6]！

#### A. `ResKeeper` 的 GameObject 挂载自毁（防泄露基石）
编写一个独立的轻量组件 `ResKeeper` [1.1.2]。
当通过 `ResourceManager` 异步/同步加载资源时，传入一个 `owner` (GameObject)；底层会自动为该物体挂载（或复用）一个 `ResKeeper`，并将资源句柄注入其内部的 `_boundHandles` 中 [1.1.2]。
当这个物体被 `Destroy` 销毁时，Unity 底层生命周期会**自动触发其 `OnDestroy`**。
在 `OnDestroy` 中循环执行 `handle.Release()`，物体的销毁与底包资源的卸载彻底在底层绑定，实现了全自动自毁 [1.1.2, 1.2.1]。

#### B. `InstantiateAsync` 的克隆体绑定（预制体实例化最优解）
针对最常用的预制体产生（如子弹、特效、UI 面板），设计了 `InstantiateAsync` 接口：
```csharp
GameObject instance = Instantiate(prefab);
BindLifecycle(instance, handle);
```
当在业务层调用 `Destroy(bulletGo)` 时，挂载在 bulletGo 上的 `ResKeeper` 会随之销毁，并**自动减少底层 AssetBundle 的引用计数**。这从底层上消除了高频生成/销毁带来的内存泄漏 [1.1.2, 1.2.1, 2.4.3]。

#### C. `SafeAsset<T>` 作用域清理（短生命周期最佳搭档）
针对配置表、单次音效或临时 UI 切图，频繁去挂载 `ResKeeper` 组件会显得过于繁重。设计了 `SafeAsset<T>` 结构体，并实现 C# 的 **`IDisposable`** 接口。
业务层只需采用 `using` 语法：
```csharp
using (var safeConfig = ResourceManager.Instance.LoadSafe<TextAsset>("Assets/Res/Skill.bytes"))
{
    // 在这里解析配置
} // 离开大括号瞬间，Dispose 被调用，句柄自动 Release，零 GC、零残留！
```

#### D. 无主资源的“场景级自动大扫除”（终极兜底）
当加载资源时传参 `owner == null`，`ResourceManager` 在底层会自动获取 `SceneManager.GetActiveScene().name`，并将其塞入场景对应的句柄容器中。
当旧场景被卸载（场景切换）时，`ResourceManager` 监听的 `OnSceneUnloaded` 触发，对属于该场景的所有无主资源句柄执行全局的 `Release` 集中大清扫。

---

### 3. 热更新入口业务内核：`HotUpdateEntry.cs`

本脚本放于热更新程序集（`HotUpdate.asmdef`）下，作为游戏内核，在接管首屏 UI 表现、从本地代码包（`DefaultPackage`）中读取包配置后，实现多包裹的异步 Task 加载与场景跳转。

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.Networking;
using UnityEngine.UI;
using TMPro;
using YooAsset;

public static class HotUpdateEntry
{
    private static PackageConfig _packageConfig; // 存储从服务器拉取到的包名配置

    private static void LoadSaveDataAndConfigs()
    {
        Debug.Log("[HotUpdate] 正在反序列化玩家存档数据...");
    }

    /// <summary>
    /// 【主入口】：通过 async 异步启动，无缝接管 AOT 场景物体
    /// </summary>
    public static async void StartGame(string nextSceneName, EPlayMode playMode, string cdnUrl)
    {
        Debug.Log("[HotUpdate] 业务内核已通过 async/await 异步启动，开始接管 UI 表现。");

        // 1. 接管 Canvas，在场景中安全定位那些在第一阶段被隐藏的进度条和状态文本
        GameObject canvasGo = GameObject.Find("Canvas");
        Slider progressSlider = null;
        TMP_Text progressText = null;

        if (canvasGo != null)
        {
            Transform sliderTrans = canvasGo.transform.Find("ProgressSlider");
            if (sliderTrans != null)
            {
                progressSlider = sliderTrans.GetComponent<Slider>();
                progressSlider.gameObject.SetActive(true); // 唤醒并显示
                progressSlider.value = 0f;
            }

            Transform textTrans = canvasGo.transform.Find("ProgressText");
            if (textTrans != null)
            {
                progressText = textTrans.GetComponent<TMP_Text>();
                progressText.gameObject.SetActive(true); // 唤醒并显示
                progressText.text = "正在准备更新资源... (0%)";
            }
        }

        // 2. 执行游戏数据、大厅、配置等业务初始化
        LoadSaveDataAndConfigs();

        // 3. 依靠 Task 任务流驱动，无需创建任何 MonoBehaviour 挂载器 [1.2.6]
        try
        {
            await LoadAllResourcePackagesAndEnterGameAsync(nextSceneName, playMode, cdnUrl, progressSlider, progressText);
        }
        catch (Exception ex)
        {
            Debug.LogError($"[HotUpdate] 异步热更新流程执行发生严重异常: {ex}");
        }
    }

    private static async Task LoadAllResourcePackagesAndEnterGameAsync(string sceneToLoad, EPlayMode playMode, string cdnUrl, Slider slider, TMP_Text text)
    {
        // -------------------------------------------------------------
        // 第一步：作为本地 TextAsset 资源，直接在更新完的代码包中读取配置文件 [1.1.8]
        // -------------------------------------------------------------
        var defaultPackage = YooAssets.GetPackage("DefaultPackage");
        string configPath = "Assets/Configs/package_config.json.bytes"; 
        
        var configHandle = defaultPackage.LoadAssetAsync<TextAsset>(configPath);
        await configHandle; // 异步等待 [1.3.1]

        if (configHandle.Status == EOperationStatus.Succeeded)
        {
            TextAsset textAsset = configHandle.AssetObject as TextAsset;
            if (textAsset != null)
            {
                _packageConfig = JsonUtility.FromJson<PackageConfig>(textAsset.text);
                Debug.Log($"[HotUpdate] 成功从本地代码包中读取最新多包配置: {textAsset.text}");
            }
        }
        else
        {
            // 兜底方案：如果本地配置读取失败，尝试外网 Web URL 异步拉取 (符合实际生产环境标准) [1.2.6]
            Debug.LogWarning("[HotUpdate] 本地配置读取失败，尝试启动外网 Web URL 动态拉取配置...");
            await FetchPackageConfigFromServerAsync();
        }
        configHandle.Release();

        if (_packageConfig == null || _packageConfig.package_names == null || _packageConfig.package_names.Count == 0)
        {
            Debug.LogError("[HotUpdate] 获取资源包清单配置失败，热更新流程被迫中断。");
            return;
        }

        // -------------------------------------------------------------
        // 第二步：平滑分配进度条，依次下载其余大型美术资源包
        // -------------------------------------------------------------
        List<string> pendingPackages = new List<string>();
        foreach (var name in _packageConfig.package_names)
        {
            if (!string.Equals(name, "DefaultPackage", StringComparison.OrdinalIgnoreCase))
            {
                pendingPackages.Add(name); // 采用 Foreach 避免 LINQ 产生的微额 GC 堆分配
            }
        }

        int totalNewPackages = pendingPackages.Count;
        if (totalNewPackages > 0)
        {
            for (int i = 0; i < totalNewPackages; i++)
            {
                string pkgName = pendingPackages[i];
                float startProgress = (float)i / totalNewPackages;
                float endProgress = (float)(i + 1) / totalNewPackages;

                // 异步等待大包更新
                await PackageManager.InitializeAndDownloadPackageAsync(pkgName, playMode, cdnUrl, slider, text, startProgress, endProgress);
            }
        }

        // -------------------------------------------------------------
        // 第三步：异步载入主场景并跳转 (载入完毕后，旧的 BootScene 连同进度条 UI 会被 Unity 自动销毁卸载)
        // -------------------------------------------------------------
        string mainPkgName = pendingPackages.Count > 0 ? pendingPackages[0] : "DefaultPackage";
        var mainPackage = YooAssets.GetPackage(mainPkgName);
        
        var sceneHandle = mainPackage.LoadSceneAsync(sceneToLoad, UnityEngine.SceneManagement.LoadSceneMode.Single);
        await sceneHandle;

        sceneHandle.Release();
    }

    private static async Task FetchPackageConfigFromServerAsync()
    {
        string url = "https://your_server.com/package_config.json"; // 移动端生产环境建议采用安全的 HTTPS 连接
        using (UnityWebRequest www = UnityWebRequest.Get(url))
        {
            var operation = www.SendWebRequest();
            while (!operation.isDone)
            {
                await Task.Yield(); // 零 GC 堆分配的异步分帧等待 [1.2.6]
            }

            if (www.result != UnityWebRequest.Result.Success)
            {
                Debug.LogError($"[HotUpdate] 请求多包配置文件失败: {www.error}");
                _packageConfig = new PackageConfig { package_names = new List<string> { "MainPackage" } };
            }
            else
            {
                string json = www.downloadHandler.text;
                _packageConfig = JsonUtility.FromJson<PackageConfig>(json);
            }
        }
    }

    [System.Serializable]
    public class PackageConfig { public List<string> package_names; }
}
```

---

#### 🛠️ 配合托管的资源加载管线：`ResourceManager.cs` 核心方法

在业务开发中，我们封装了这套带有 **“生命周期自动托管”** 的资源助手。它支持**异步加载任务**、**克隆体自毁绑定**、**using 作用域托管** 以及 **场景级全自动大扫除**：

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.SceneManagement;
using YooAsset;

public class ResourceManager : MonoBehaviour
{
    public static ResourceManager Instance { get; private set; }
    private string _defaultPackageName = "MainPackage";
    private readonly Dictionary<string, List<AssetHandle>> _sceneHandles = new(4);

    // 【1. 经典同步加载】：替代 Unity 的 Resources.Load
    public T Load<T>(string address, GameObject owner = null) where T : UnityEngine.Object
    {
        var package = YooAssets.GetPackage(_defaultPackageName);
        AssetHandle handle = package.LoadAssetSync<T>(address);

        if (owner != null)
        {
            BindLifecycle(owner, handle); // 绑定到 GameObject，物体销毁时自动 Release [1.1.2, 1.2.1]
        }
        else
        {
            BindToCurrentScene(handle); // 无主资源自动绑定到活跃场景，切场景时大扫除清理
        }

        return handle.AssetObject as T;
    }

    // 【2. 异步实例化接口】：异步加载预制体，并将句柄绑定在克隆体上。
    // 当调用 Destroy(克隆体) 时，它的 AB 引用计数会在底层自动释放，从而规避内存泄漏。
    public async Task<GameObject> InstantiateAsync(string address, Transform parent = null)
    {
        var package = YooAssets.GetPackage(_defaultPackageName);
        AssetHandle handle = package.LoadAssetAsync<GameObject>(address);

        await handle; // 异步挂起等待载入，不阻塞主线程 [1.2.1, 1.3.1]

        if (handle.Status != EOperationStatus.Succeeded)
        {
            Debug.LogError($"[ResourceManager] 异步加载预制体失败: {address}");
            handle.Release();
            return null;
        }

        GameObject prefab = handle.AssetObject as GameObject;
        GameObject instance = Instantiate(prefab, parent);

        BindLifecycle(instance, handle); // 绑定到生成的克隆体本身！
        return instance;
    }

    // 【3. using 作用域加载】：最适合解析配置表、播放音效等短生命周期资源。离开 using 块立即卸载！
    public SafeAsset<T> LoadSafe<T>(string address) where T : UnityEngine.Object
    {
        var package = YooAssets.GetPackage(_defaultPackageName);
        AssetHandle handle = package.LoadAssetSync<T>(address);
        return new SafeAsset<T>(handle);
    }

    private void BindLifecycle(GameObject target, AssetHandle handle)
    {
        if (target == null || handle == null) return;
        if (!target.TryGetComponent<ResKeeper>(out var keeper))
        {
            keeper = target.AddComponent<ResKeeper>();
        }
        keeper.BindHandle(handle);
    }

    private void BindToCurrentScene(AssetHandle handle)
    {
        string activeSceneName = SceneManager.GetActiveScene().name;
        if (!_sceneHandles.TryGetValue(activeSceneName, out var list))
        {
            list = new List<AssetHandle>(16); // 预分配容量，避免移动端频繁扩容导致 GC
            _sceneHandles[activeSceneName] = list;
        }
        list.Add(handle);
    }

    private void OnSceneUnloaded(Scene scene)
    {
        // 场景卸载时，自动对该场景关联的所有无主句柄执行全局的 Release 清理
        if (_sceneHandles.TryGetValue(scene.name, out var list))
        {
            foreach (var handle in list)
            {
                if (handle != null) handle.Release();
            }
            list.Clear();
            _sceneHandles.Remove(scene.name);
        }
    }
}
```

---

## 💡 附录：自动化配套与工程规范（附加阅读）

### 1. 独立的资源生命周期守护组件：`ResKeeper.cs`
将此脚本独立存放。当挂载的物体销毁时，它的 `OnDestroy` 机制会在底层自动削减对应的引用计数。

```csharp
using System.Collections.Generic;
using UnityEngine;
using YooAsset;

[DisallowMultipleComponent]
public class ResKeeper : MonoBehaviour
{
    private readonly List<AssetHandle> _boundHandles = new();

    public void BindHandle(AssetHandle handle)
    {
        if (handle == null) return;
        _boundHandles.Add(handle);
    }

    private void OnDestroy()
    {
        for (int i = 0; i < _boundHandles.Count; i++)
        {
            var handle = _boundHandles[i];
            if (handle != null && handle.IsValid)
            {
                handle.Release(); // 自动释放 YooAsset 引用计数
            }
        }
        _boundHandles.Clear();
    }
}
```

### 2. 作用域安全托管辅助类：`SafeAsset.cs`
```csharp
using System;
using YooAsset;

public sealed class SafeAsset<T> : IDisposable where T : UnityEngine.Object
{
    private AssetHandle _handle;
    public T Asset => _handle?.AssetObject as T;

    public SafeAsset(AssetHandle handle) { _handle = handle; }

    public void Dispose()
    {
        if (_handle != null)
        {
            _handle.Release();
            _handle = null;
        }
    }
}
```

---

### 3. 本地提效：AOT 补充元数据与热更 DLL 自动收集覆盖脚本
该编辑器自动化脚本可以一键在本地提取生成的 AOT 程序集和热更代码，避免手动操作的低效。请将其放置在项目的 `Editor` 文件夹下运行：

```csharp
#if UNITY_EDITOR
using System;
using System.Collections.Generic;
using System.IO;
using System.Reflection;
using UnityEditor;
using UnityEngine;

public static class HybridCLRAOTDllCopier
{
    private static IReadOnlyList<string> GetPatchedAOTAssemblyList()
    {
        Type t = Type.GetType("AOTGenericReferences");
        if (t != null) return GetListFromType(t);

        string[] targetAssemblies = { "Assembly-CSharp", "Assembly-CSharp-firstpass" };
        foreach (var assName in targetAssemblies)
        {
            try
            {
                var assembly = Assembly.Load(assName);
                if (assembly != null)
                {
                    foreach (var type in assembly.GetTypes())
                    {
                        if (type.Name == "AOTGenericReferences") return GetListFromType(type);
                    }
                }
            }
            catch { }
        }
        return null;
    }

    private static IReadOnlyList<string> GetListFromType(Type type)
    {
        var field = type.GetField("PatchedAOTAssemblyList", BindingFlags.Public | BindingFlags.Static);
        return field != null ? field.GetValue(null) as IReadOnlyList<string> : null;
    }

    private static void CopyDlls(string target)
    {
        string projectRoot = Directory.GetCurrentDirectory();
        var assemblyList = GetPatchedAOTAssemblyList();
        bool hasAOTList = (assemblyList != null && assemblyList.Count > 0);
        int aotCopiedCount = 0;

        if (hasAOTList)
        {
            string baseAOTSourceDir = Path.Combine(projectRoot, "HybridCLRData", "AssembliesPostIl2CppStrip");
            string aotSourceDir = Path.Combine(baseAOTSourceDir, target);

            if (target == "Windows" && !Directory.Exists(aotSourceDir))
            {
                if (Directory.Exists(Path.Combine(baseAOTSourceDir, "StandaloneWindows64")))
                    aotSourceDir = Path.Combine(baseAOTSourceDir, "StandaloneWindows64");
            }

            if (Directory.Exists(aotSourceDir))
            {
                string aotDestDir = Path.Combine(projectRoot, "Assets", "AotDlls");
                if (!Directory.Exists(aotDestDir)) Directory.CreateDirectory(aotDestDir);

                foreach (var aotDll in assemblyList)
                {
                    string dllFileName = aotDll;
                    if (!dllFileName.EndsWith(".dll", StringComparison.OrdinalIgnoreCase)) dllFileName += ".dll";

                    string sourceFilePath = Path.Combine(aotSourceDir, dllFileName);
                    string destFilePath = Path.Combine(aotDestDir, dllFileName + ".bytes");

                    if (File.Exists(sourceFilePath))
                    {
                        File.Copy(sourceFilePath, destFilePath, true);
                        aotCopiedCount++;
                    }
                }
            }
        }

        string baseHotUpdateSourceDir = Path.Combine(projectRoot, "HybridCLRData", "HotUpdateDlls");
        string hotUpdateSourceDir = Path.Combine(baseHotUpdateSourceDir, target);

        bool hotUpdateCopied = false;
        string hotUpdateSourceFile = Path.Combine(hotUpdateSourceDir, "HotUpdate.dll");

        if (Directory.Exists(hotUpdateSourceDir) && File.Exists(hotUpdateSourceFile))
        {
            string hotUpdateDestDir = Path.Combine(projectRoot, "Assets", "HotUpdateDlls");
            if (!Directory.Exists(hotUpdateDestDir)) Directory.CreateDirectory(hotUpdateDestDir);

            File.Copy(hotUpdateSourceFile, Path.Combine(hotUpdateDestDir, "HotUpdate.dll.bytes"), true);
            hotUpdateCopied = true;
        }

        AssetDatabase.Refresh();
        EditorUtility.DisplayDialog("Dll 收集器结果", "AOT元数据与热更代码自动同步重命名完成！", "确定");
    }

    [MenuItem("HybridCLR/CopyDllsToAssets/Android")]
    public static void CopyAndroidDlls() { CopyDlls("Android"); }

    [MenuItem("HybridCLR/CopyDllsToAssets/iOS")]
    public static void CopyIOSDlls() { CopyDlls("iOS"); }

    [MenuItem("HybridCLR/CopyDllsToAssets/Windows")]
    public static void CopyWindowsDlls() { CopyDlls("Windows"); }
}
#endif
```

---

### 4. 团队开发 Git 过滤规则配置（`.gitignore`）
```gitignore
# Unity 官方标准排除项
/[Ll]ibrary/
/[Tt]emp/
/[Oo]bj/
/[Bb]uild/
/[Bb]uilds/
/[Ll]ogs/
/[Uu]ser[Ss]ettings/

# 常用 IDE 排除项
.vs/
*.csproj
*.sln
.idea/
.vscode/
.DS_Store

# HybridCLR 中间程序集与桥接代码 (本地一键可重建)
/HybridCLRData/
Assets/HybridCLRData/

# YooAsset 构建出的庞大资源包及清单，切勿上传！
/Bundles/
/yoo/

# 自动拷贝生成的 .bytes 排除，交由开发人员在本地通过 Copier 一键导入
Assets/AotDlls/*.bytes
Assets/AotDlls/*.bytes.meta
Assets/HotUpdateDlls/*.bytes
Assets/HotUpdateDlls/*.bytes.meta
```