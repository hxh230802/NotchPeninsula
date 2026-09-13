# NotchPeninsula 插件制作指南

本文档教你如何为 NotchPeninsula（NPS）编写一个插件。NPS 是一个「刘海/灵动岛」风格的桌面挂件，其本体是一个**平台**，所有可见功能（时间、硬件监控、媒体控制、课表等）都由**插件**提供。

读完本文你可以：

- 新建一个插件工程并让主程序加载它；
- 在刘海栏里显示自定义内容（组件）；
- 添加右键展开的详情页；
- 添加设置页（声明式控件 + 自定义 UI）；
- 弹出插件自己的窗口；
- 使用主机的设置持久化、定时刷新、提醒等服务。

---

## 目录

1. [架构总览](#1-架构总览)
2. [插件工程结构](#2-插件工程结构)
3. [插件入口 `INotchPlugin`](#3-插件入口-inotchplugin)
4. [组件 `IWidget`](#4-组件-iwidget)
5. [详情页 `IDetailPage`](#5-详情页-idetailpage)
6. [设置页](#6-设置页)
7. [插件窗口 `IPluginWindow`](#7-插件窗口-ipluginwindow)
8. [主机服务 `IPluginHost`](#8-主机服务-ipluginhost)
9. [部署插件](#9-部署插件)
10. [完整最小示例](#10-完整最小示例)
11. [线程模型与注意事项](#11-线程模型与注意事项)

---

## 1. 架构总览

```
NotchPeninsula.exe（平台）
 ├─ Plugin/PluginApi.cs      ← 插件契约（所有接口定义在这里）
 ├─ Plugin/PluginLoader.cs   ← 从 plugins/ 目录发现并加载插件 DLL
 ├─ Plugin/PluginHost.cs     ← 插件宿主：注册中心 + 服务
 └─ plugins/                 ← 插件目录（每个子目录一个插件）
     ├─ HelloPlugin/
     │   ├─ HelloPlugin.dll
     │   └─ plugin.json
     ├─ SystemPlugins/
     └─ ThuCourse/
```

**核心思想**：插件是独立的 .NET 类库（DLL），实现 `INotchPlugin` 接口。主程序启动时扫描 `plugins/` 目录，用 `AssemblyLoadContext` 加载每个插件的 DLL，反射找到 `INotchPlugin` 实现，调用 `Initialize(host)` 完成注册。

插件的**依赖解析**：插件自己带的 DLL（如课表插件带的 `ThuInfoLib.dll`）从插件目录加载；与主程序共享的（如 `SkiaSharp.dll`、`NotchPeninsula.dll`）回退到主程序，避免类型重复加载。

---

## 2. 插件工程结构

一个插件就是一个 class library 工程，目录建议放在 `SamplePlugins/<插件名>/`。

### 2.1 csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0-windows10.0.19041.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <OutputType>Library</OutputType>
  </PropertyGroup>
  <ItemGroup>
    <!-- 引用主程序，获得 INotchPlugin / IWidget 等插件 API 类型 -->
    <ProjectReference Include="..\..\NotchPeninsula.csproj" />
  </ItemGroup>
</Project>
```

> 说明：
> - `TargetFramework` 必须与主程序一致（`net10.0-windows10.0.19041.0`）。
> - 引用 `NotchPeninsula.csproj` 后，插件即可使用 `NotchPeninsula.Plugins` 命名空间下的所有接口，以及 `SkiaSharp`。
> - 若插件有**私有依赖**（别的 NuGet 包或自己的库），默认 `dotnet build` 会把它们复制到插件输出目录，部署时一起带上即可。

### 2.2 plugin.json（插件清单）

放在插件源码根目录，内容：

```json
{ "id": "hello", "name": "Hello 演示插件", "dll": "HelloPlugin.dll" }
```

| 字段 | 含义 |
|---|---|
| `id` | 插件唯一 ID（用于设置隔离前缀，见 §8） |
| `name` | 显示名（元数据） |
| `dll` | 插件 DLL 文件名（`PluginLoader` 据此加载） |

> 注意：`id` 必须与 `INotchPlugin.Id` 一致，否则设置会串。

---

## 3. 插件入口 `INotchPlugin`

```csharp
public interface INotchPlugin
{
    string Id { get; }          // 唯一 ID
    string DisplayName { get; } // 显示名
    string Version { get; }     // 版本号
    void Initialize(IPluginHost host); // 入口，在这里注册组件/设置页
}
```

示例：

```csharp
using NotchPeninsula.Plugins;

public sealed class MyPlugin : INotchPlugin
{
    public string Id => "myplugin";
    public string DisplayName => "我的插件";
    public string Version => "1.0.0";

    public void Initialize(IPluginHost host)
    {
        host.RegisterWidget(new MyWidget(host));
        host.RegisterSettingsPage(new MySettingsPage(host));
    }
}
```

`Initialize` 里可以注册多个组件、多个设置页、辅助组件。

---

## 4. 组件 `IWidget`

组件是刘海栏里**横排显示**的一块内容。每个组件有宽度（`MeasureWidth`）、绘制（`Draw`）、命中检测（`HitTest`）和点击回调。

```csharp
public interface IWidget
{
    string Id { get; }
    string DisplayName { get; }
    IDetailPage? DetailPage { get; }   // null = 无详情页（右键不会展开）

    float MeasureWidth(float availableHeight);  // 期望宽度（逻辑像素）
    void Draw(SKCanvas canvas, SKRect rect, WidgetFrame frame);

    WidgetHit HitTest(float x, float y, SKRect rect); // 左键命中
    void OnLeftClick(string? action, float x, float y);
    void OnRightClick();                              // 右键（默认展开详情页）
    void OnActivate(IPluginHost host);
    void OnDeactivate();
}
```

### 4.1 最小组件

```csharp
public sealed class MyWidget : IWidget
{
    private static readonly SKPaint _paint = new()
    {
        Color = SKColors.White, TextSize = 13f, IsAntialias = true,
        Typeface = SKTypeface.FromFamilyName("Microsoft YaHei UI",
            SKFontStyleWeight.Bold, SKFontStyleWidth.Normal, SKFontStyleSlant.Upright)
    };

    public string Id => "my.widget";
    public string DisplayName => "我的组件";
    public IDetailPage? DetailPage => null;

    public float MeasureWidth(float availableHeight)
        => _paint.MeasureText("你好") + 32f;

    public void Draw(SKCanvas canvas, SKRect rect, WidgetFrame frame)
    {
        _paint.Color = frame.Theme.TextColor.WithAlpha(frame.Alpha);
        canvas.DrawText("你好", rect.Left + 16f, rect.MidY + 5f, _paint);
    }

    public WidgetHit HitTest(float x, float y, SKRect rect) => WidgetHit.None;
    public void OnLeftClick(string? action, float x, float y) { }
    public void OnRightClick() { }
    public void OnActivate(IPluginHost host) { }
    public void OnDeactivate() { }
}
```

### 4.2 关键点

- **坐标**：`Draw` 的 `rect` 是组件所在区域（逻辑坐标），`rect.Left`/`rect.MidY` 是定位基准。坐标以 `rect` 为准，不要假设绝对位置。
- **主题色**：用 `frame.Theme.TextColor`（主文字）、`frame.Theme.SubTextColor`（次要文字），并乘上 `frame.Alpha`（淡入/叠化透明度），这样组件自动适配深浅色主题。
- **宽度**：`MeasureWidth` 返回组件想要的宽度（逻辑像素），主程序据此布局。文本用 `_paint.MeasureText` 量出来加 padding。
- **命中**：`HitTest` 返回 `WidgetHit(action)` 表示命中某个可点击区域（`action` 是自定义字符串），`WidgetHit.None` 表示不命中。命中后主程序调用 `OnLeftClick(action, x, y)`。

### 4.3 带点击的组件

```csharp
public WidgetHit HitTest(float x, float y, SKRect rect)
{
    // 假设按钮在 rect 右侧 30px 宽
    if (x >= rect.Right - 30f && x <= rect.Right) return new WidgetHit("toggle");
    return WidgetHit.None;
}

public void OnLeftClick(string? action, float x, float y)
{
    if (action == "toggle") { /* 切换状态 */ }
}
```

---

## 5. 详情页 `IDetailPage`

右键组件（`OnRightClick`）时展开的详情页。比组件更高、更宽，可容纳更多内容。

```csharp
public interface IDetailPage
{
    float MeasureWidth();
    float MeasureHeight();
    void Draw(SKCanvas canvas, SKRect rect, WidgetFrame frame);
    WidgetHit HitTest(float x, float y, SKRect rect);
    void OnAction(string? action, float x, float y);
}
```

示例：

```csharp
public sealed class MyDetailPage : IDetailPage
{
    public float MeasureWidth() => 340f;
    public float MeasureHeight() => 130f;

    public void Draw(SKCanvas canvas, SKRect rect, WidgetFrame frame)
    {
        var p = new SKPaint { Color = frame.Theme.TextColor.WithAlpha(frame.Alpha),
            TextSize = 14f, IsAntialias = true,
            Typeface = SKTypeface.FromFamilyName("Microsoft YaHei UI", SKFontStyleWeight.Bold) };
        canvas.DrawText("详情页标题", rect.Left + 20f, rect.Top + 28f, p);
    }

    public WidgetHit HitTest(float x, float y, SKRect rect) => WidgetHit.None;
    public void OnAction(string? action, float x, float y) { }
}
```

> 提示：详情页高度 `MeasureHeight` 会被主程序用来自动撑高窗口（上限 `MAX_WINDOW_HEIGHT`）。要做成**可交互的详情页**（比如日期选择器），参考 `SamplePlugins/ThuCourse` 的 `CourseDetailPage`：用 `HitTest` 返回 `WidgetHit("day:3")`，`OnAction` 里解析动作更新状态。

---

## 6. 设置页

插件设置显示在主程序设置窗口的「插件管理 / 插件设置」页里。有**声明式控件**和**自定义 UI** 两种，可同时使用。

### 6.1 声明式控件 `ISettingsPage`

适合开关、下拉、数字等简单设置：

```csharp
public sealed class MySettingsPage : ISettingsPage
{
    public string Title => "我的插件";

    public IReadOnlyList<SettingControl> Controls { get; } = new SettingControl[]
    {
        new ToggleSetting("Enable", "启用", true),                       // 开关
        new ChoiceSetting("Mode", "模式", new[] { "自动", "手动" }, 0),  // 下拉
        new NumberSetting("Interval", "刷新间隔(秒)", 1f, 60f, 1f, 5f), // 数字（步进）
    };
}
```

三种控件：

| 控件 | 参数 | 说明 |
|---|---|---|
| `ToggleSetting(key, label, default)` | 开关 | 值存 "1"/"0" |
| `ChoiceSetting(key, label, options[], defaultIndex)` | 下拉 | 值存选中索引（字符串） |
| `NumberSetting(key, label, min, max, step, default)` | 步进 | 值存浮点数字符串 |

> 主程序会渲染这些控件，并在用户改动时调用 `host.SetSetting(key, value)` 并触发 `SettingsChanged` 事件。

### 6.2 自定义 UI `ICustomSettingsPage`

适合需要自己绘制的复杂设置（比如按钮、自定义布局）：

```csharp
public interface ICustomSettingsPage
{
    float MeasureHeight();
    void Draw(SKCanvas canvas, SKRect rect, RenderTheme theme);
    void OnMouseDown(float x, float y);
    void OnMouseMove(float x, float y);
    void OnMouseUp(float x, float y);
}
```

一个设置页可以**同时实现** `ISettingsPage` 和 `ICustomSettingsPage`：声明式控件先渲染，自定义 UI 渲染在其下方。

```csharp
public sealed class MySettingsPage : ISettingsPage, ICustomSettingsPage
{
    private bool _hover;
    public string Title => "我的插件";
    public IReadOnlyList<SettingControl> Controls { get; } = Array.Empty<SettingControl>();

    public float MeasureHeight() => 40f;

    public void Draw(SKCanvas canvas, SKRect rect, RenderTheme theme)
    {
        var btn = new SKRect(rect.Left, rect.Top, rect.Left + 100, rect.Top + 30);
        var p = new SKPaint { Color = _hover ? new SKColor(0,140,240) : new SKColor(0,120,212), IsAntialias = true };
        canvas.DrawRoundRect(btn, 6, 6, p);
        var t = new SKPaint { Color = SKColors.White, TextSize = 13f, IsAntialias = true,
            Typeface = SKTypeface.FromFamilyName("Microsoft YaHei UI") };
        canvas.DrawText("点我", btn.Left + 30, btn.Top + 20, t);
    }

    public void OnMouseDown(float x, float y)
    {
        if (x >= 0 && x <= 100 && y >= 0 && y <= 30) { /* 点击处理 */ }
    }
    public void OnMouseMove(float x, float y) => _hover = x >= 0 && x <= 100 && y >= 0 && y <= 30;
    public void OnMouseUp(float x, float y) { }
}
```

### 6.3 设置读写与即时生效

```csharp
// 读（带默认值）
bool enabled = host.GetSetting("Enable", "1") == "1";

// 写（会触发 SettingsChanged）
host.SetSetting("Enable", enabled ? "0" : "1");

// 订阅变更，改动后立即生效
host.SettingsChanged += () =>
{
    enabled = host.GetSetting("Enable", "1") == "1";
    // 更新状态...
};
```

> 设置的 key 会自动加前缀 `Plugin.<id>.`，例如 `Plugin.myplugin.Enable`，因此不同插件之间互不干扰。**不要**直接读注册表。

---

## 7. 插件窗口 `IPluginWindow`

插件可以弹出自己的独立窗口（基于 Win32 分层窗口 + SkiaSharp 绘制），用于登录、表单等场景。

```csharp
public interface IPluginWindow
{
    void SetDraw(Action<SKCanvas, int, int>? draw);  // 绘制回调 (canvas, 宽, 高)
    void SetMouse(Action<float,float>? down, Action<float,float>? move, Action<float,float>? up);
    void SetKey(Action<char>? key);                  // 键盘字符输入
    void RequestRedraw();
    void Close();
}
```

示例：

```csharp
var win = host.CreateWindow("登录", 320, 220);

win.SetDraw((canvas, w, h) =>
{
    // 画内容（背景是透明的，圆角+边框由框架统一提供）
    var p = new SKPaint { Color = SKColors.White, TextSize = 14f, IsAntialias = true,
        Typeface = SKTypeface.FromFamilyName("Microsoft YaHei UI") };
    canvas.DrawText("请输入", 24, 44, p);
});

win.SetMouse((x, y) =>
{
    // 鼠标按下，x/y 是逻辑坐标
}, (x, y) => { /* 移动 */ }, (x, y) => { /* 抬起 */ });

win.SetKey(c =>
{
    // 键盘输入字符（含退格 '\b'）
});

// 需要重绘时：
win.RequestRedraw();

// 关闭：
win.Close();
```

**窗口自带的能力**（无需插件处理）：

- **居中显示**（DPI 感知）。
- **右上角圆形关闭按钮** + **Esc 键**关闭。
- **置顶**（阻止下方交互）。
- **圆角 + 边框**外观。

> 注意：窗口背景由框架绘制（深色圆角 + 边框），插件的 `SetDraw` 里**不要** `canvas.Clear(深色)`，直接画内容即可（否则会把框架的背景抹掉）。需要透明就画在透明背景上。

---

## 8. 主机服务 `IPluginHost`

`IPluginHost` 是插件与主程序交互的核心。常用成员：

```csharp
public interface IPluginHost
{
    // 注册
    void RegisterWidget(IWidget widget);
    void RegisterSecondaryWidget(ISecondaryWidget widget);
    void RegisterSettingsPage(ISettingsPage page);

    // 主题快照
    RenderTheme CurrentTheme { get; }

    // 提醒
    void PostReminder(ReminderData reminder);

    // 设置
    string GetSetting(string key, string fallback);
    void SetSetting(string key, string value);
    event Action? SettingsChanged;

    // 定时刷新
    IDisposable ScheduleRefresh(TimeSpan interval, Action callback);

    // 窗口
    IPluginWindow CreateWindow(string title, int width, int height);

    // 交互（预留）
    void RequestRedraw();
    void OpenDetailPage(string widgetId);
    void CloseDetailPage();
}
```

### 8.1 定时刷新

```csharp
host.ScheduleRefresh(TimeSpan.FromMinutes(5), () =>
{
    // 在后台线程执行，更新自己的数据
    FetchLatestData();
});
```

返回的 `IDisposable` 可用来停止刷新（`Dispose()`）。

### 8.2 提醒

```csharp
host.PostReminder(new ReminderData
{
    Title    = "课表提醒",
    Body     = "下一节课 10 分钟后开始",
    Duration = TimeSpan.FromSeconds(5),        // 展示时长，默认 4 秒
    IconPath = @"C:\path\to\icon.png",         // 自定义图标路径（可选）
    OnClick  = () => { /* 点击提醒时触发 */ }, // 点击回调（可选）
});
```

`PostReminder` 复用主程序的 Toast 通知机制，弹出一次性提醒。`ReminderData` 共五个字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `Title` | string | 标题 |
| `Body` | string | 正文 |
| `Duration` | TimeSpan | 展示时长，默认 4 秒 |
| `IconPath` | string? | 自定义图标路径；缺省用默认图标 |
| `OnClick` | Action? | 点击提醒时触发的回调；缺省点击只关闭提醒 |

### 8.3 主题

`RenderTheme` 提供当前帧的颜色快照：

```csharp
public readonly record struct RenderTheme(
    SKColor TextColor,     // 主文字色
    SKColor SubTextColor,  // 次要文字色
    SKColor BackgroundColor,
    float GlobalDpi,
    float NotchBottomRadius);
```

组件里一般直接用 `WidgetFrame.Theme`（见 §4），不必单独取 `CurrentTheme`。

---

## 9. 部署插件

插件编译出的 DLL（及其私有依赖）放到主程序运行目录的 `plugins/<插件名>/` 下：

```
NotchPeninsula.exe
plugins/
  └─ MyPlugin/
      ├─ MyPlugin.dll
      └─ plugin.json
```

- `plugin.json` 里的 `dll` 字段指向插件 DLL 文件名。
- 插件私有依赖（非共享的）也放同一目录。
- 重启主程序即可加载。

> 主程序已内置「随主工程编译并自动部署插件」的 MSBuild Target（见 `NotchPeninsula.csproj` 的 `BuildAndDeployPlugins`），`SamplePlugins/` 下的插件工程会自动编译并把产物部署到 `plugins/`，无需手工拷贝。

---

## 10. 完整最小示例

一个显示「已运行」文本、带一个开关设置、可右键展开详情页的完整插件：

```csharp
using NotchPeninsula.Plugins;
using SkiaSharp;

namespace MyPlugin;

public sealed class MyPlugin : INotchPlugin
{
    public string Id => "myplugin";
    public string DisplayName => "我的插件";
    public string Version => "1.0.0";

    public void Initialize(IPluginHost host)
    {
        host.RegisterWidget(new MyWidget(host));
        host.RegisterSettingsPage(new MySettingsPage());
    }
}

public sealed class MySettingsPage : ISettingsPage
{
    public string Title => "我的插件";
    public IReadOnlyList<SettingControl> Controls { get; } = new SettingControl[]
    {
        new ToggleSetting("Enabled", "启用", true),
    };
}

public sealed class MyWidget : IWidget
{
    private readonly IPluginHost _host;
    private volatile bool _enabled = true;

    public MyWidget(IPluginHost host)
    {
        _host = host;
        _enabled = host.GetSetting("Enabled", "1") == "1";
        host.SettingsChanged += () =>
            _enabled = host.GetSetting("Enabled", "1") == "1";
    }

    public string Id => "my.text";
    public string DisplayName => "我的组件";
    public IDetailPage? DetailPage { get; } = new MyDetailPage();

    private static readonly SKPaint _paint = new()
    {
        Color = SKColors.White, TextSize = 13f, IsAntialias = true,
        Typeface = SKTypeface.FromFamilyName("Microsoft YaHei UI", SKFontStyleWeight.Bold)
    };

    public float MeasureWidth(float h) => _paint.MeasureText("已运行") + 32f;

    public void Draw(SKCanvas canvas, SKRect rect, WidgetFrame frame)
    {
        _paint.Color = frame.Theme.TextColor.WithAlpha(frame.Alpha);
        canvas.DrawText(_enabled ? "已运行" : "已停止", rect.Left + 16f, rect.MidY + 5f, _paint);
    }

    public WidgetHit HitTest(float x, float y, SKRect rect) => WidgetHit.None;
    public void OnLeftClick(string? a, float x, float y) { }
    public void OnRightClick() { }
    public void OnActivate(IPluginHost host) { }
    public void OnDeactivate() { }
}

public sealed class MyDetailPage : IDetailPage
{
    public float MeasureWidth() => 320f;
    public float MeasureHeight() => 130f;

    public void Draw(SKCanvas canvas, SKRect rect, WidgetFrame frame)
    {
        var p = new SKPaint { Color = frame.Theme.TextColor.WithAlpha(frame.Alpha),
            TextSize = 14f, IsAntialias = true,
            Typeface = SKTypeface.FromFamilyName("Microsoft YaHei UI", SKFontStyleWeight.Bold) };
        canvas.DrawText("我的详情页", rect.Left + 20f, rect.Top + 28f, p);
    }

    public WidgetHit HitTest(float x, float y, SKRect rect) => WidgetHit.None;
    public void OnAction(string? a, float x, float y) { }
}
```

对应的 `plugin.json`：

```json
{ "id": "myplugin", "name": "我的插件", "dll": "MyPlugin.dll" }
```

---

## 11. 线程模型与注意事项

### 11.1 线程模型

- **`MeasureWidth` / `Draw`**：在主程序**渲染线程**被每帧调用（约 60 FPS）。读取的插件内部状态必须线程安全（`volatile` / `lock` / `Interlocked`）。
- **`ScheduleRefresh` 回调**：在**后台线程**执行。更新数据时注意与渲染线程的同步（一般 `volatile` 字段足够）。
- **`HitTest` / `OnLeftClick` / `OnAction`**：在主程序 UI 线程（消息循环）调用。

### 11.2 注意事项

1. **不要依赖 `Renderer` / `NotchWindow` 的私有状态**：插件应自包含，只依赖公共契约（`WidgetFrame`、canvas、`IPluginHost`）。这是平台化的核心约定。
2. **主题色**：始终用 `frame.Theme.TextColor` / `SubTextColor`，并乘 `frame.Alpha`，不要写死颜色。
3. **SKPaint 复用**：把 `SKPaint` 做成 `static readonly` 字段，避免每帧分配（性能）。
4. **设置前缀**：设置 key 会被自动加 `Plugin.<id>.` 前缀，读写都用 `host.GetSetting/SetSetting`，不要直接碰注册表。
5. **插件窗口背景**：`SetDraw` 里不要 `canvas.Clear(深色)`，框架已提供圆角深色背景 + 边框，直接画内容。
6. **组件 ID 唯一**：不同组件的 `Id` 要唯一（组件顺序/启停按 ID 持久化）。
7. **私有依赖**：插件的私有 NuGet 包/库要随插件部署到插件目录；共享的（SkiaSharp、NotchPeninsula 等）不要重复打包，避免类型重复加载。

---

## 参考

- 契约定义：`Plugin/PluginApi.cs`
- 加载逻辑：`Plugin/PluginLoader.cs`
- 宿主实现：`Plugin/PluginHost.cs`
- 示例插件：`SamplePlugins/HelloPlugin`（最简）、`SamplePlugins/SystemPlugins`（媒体/时钟/硬件）、`SamplePlugins/ThuCourse`（课表：登录窗口 + 日期选择详情页）
