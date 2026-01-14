---
title: 雙版本 UXML 實現 Unity 版本相容性
last_updated: 2026-01-14
owner: @jialongzhang
status: accepted
related_adr: []
---

# ADR-0001: 雙版本 UXML 實現 Unity 版本相容性

## 狀態

**Accepted**

## 背景

MCP For Unity 專案需要同時支援 Unity 2020.3 LTS 和 Unity 2021+ 版本。

### 問題

1. **UI Toolkit 差異**：
   - Unity 2021+ 提供 `DropdownField` 元件
   - Unity 2020 只有 `PopupField<T>`（功能等價但類型不同）

2. **UXML 限制**：
   - UXML 檔案**不支援條件編譯**（`#if` 等預處理器指令）
   - 無法在同一個 UXML 中針對不同 Unity 版本使用不同元件

3. **維護需求**：
   - 本專案是 upstream（CoplayDev/unity-mcp）的 fork
   - 需要定期從 upstream 同步更新
   - 希望盡量保持 upstream 原版程式碼完整

## 決策

採用**雙版本 UXML + C# 條件編譯載入**方案：

### 檔案結構

```
MCPForUnity/Editor/Windows/
├── Components/ClientConfig/
│   ├── McpClientConfigSection.uxml       # 原版（2021+）
│   └── McpClientConfigSection_2020.uxml  # Unity 2020 專用
├── EditorPrefs/
│   ├── EditorPrefsWindow.uxml            # 原版（2021+）
│   ├── EditorPrefsWindow_2020.uxml       # Unity 2020 專用
│   ├── EditorPrefItem.uxml               # 原版（2021+）
│   └── EditorPrefItem_2020.uxml          # Unity 2020 專用
```

### UXML 差異

| 版本 | 原版（2021+） | _2020 版本 |
|-----|--------------|-----------|
| Dropdown 元件 | `<ui:DropdownField name="xxx">` | `<ui:VisualElement name="xxx-container">` |
| 其他內容 | 完全相同 | 完全相同 |

### C# 載入邏輯

```csharp
// UXML 載入
#if UNITY_2021_1_OR_NEWER
    var tree = AssetDatabase.LoadAssetAtPath<VisualTreeAsset>("path/to/Component.uxml");
#else
    var tree = AssetDatabase.LoadAssetAtPath<VisualTreeAsset>("path/to/Component_2020.uxml");
#endif

// Dropdown 處理
#if UNITY_2021_1_OR_NEWER
    var dropdown = root.Q<DropdownField>("name");
    dropdown.choices = choices;
#else
    var container = root.Q<VisualElement>("name-container");
    var dropdown = new PopupField<string>("", choices, 0);
    container.Add(dropdown);
#endif
```

## 考慮過的替代方案

### 方案 A：單一 UXML + 完全動態創建

所有版本都使用 VisualElement 容器，C# 中動態創建 dropdown。

**優點**：
- 只需維護一份 UXML

**缺點**：
- Unity 2021+ 無法使用 upstream 原版 UXML
- 每次從 upstream 同步都需要手動修改 UXML
- 失去 UXML 中直接定義 DropdownField 的便利性

### 方案 B：完全分叉維護

完全獨立維護 Unity 2020 版本，不再同步 upstream。

**優點**：
- 完全控制程式碼

**缺點**：
- 無法獲得 upstream 的新功能和修復
- 長期維護成本高

## 後果

### 正面

1. **upstream 同步簡化**：
   - 原版 UXML 可直接從 upstream 覆蓋
   - 只需維護 `_2020.uxml` 差異版本

2. **版本行為一致**：
   - 兩個版本視覺外觀相同
   - 功能完全等價

3. **程式碼清晰**：
   - 條件編譯區塊明確標示版本差異
   - 易於理解和維護

### 負面

1. **檔案數量增加**：
   - 每個有 DropdownField 的 UXML 需要額外的 `_2020` 版本

2. **同步時需注意**：
   - 當 upstream 新增含 DropdownField 的 UXML 時，需建立對應 `_2020` 版本

## 影響的檔案

### 新增
- `McpClientConfigSection_2020.uxml`
- `EditorPrefItem_2020.uxml`
- `EditorPrefsWindow_2020.uxml`

### 修改
- `MCPForUnityEditorWindow.cs` - UXML 載入條件編譯
- `McpClientConfigSection.cs` - dropdown 處理條件編譯
- `EditorPrefsWindow.cs` - UXML 載入 + dropdown 處理條件編譯
