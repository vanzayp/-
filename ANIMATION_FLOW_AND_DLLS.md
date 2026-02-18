# AssetStudio Animation Export: 完全処理トレースとDLL一覧

## 0) このファイルで何をしたか（要約）

- Bundle読込から `.anim` 保存までの処理順を、分岐込みで順番に整理。
- `.anim` 変換で使うDLL（`acl` / `sracl` / `acldb`）と分岐条件を明記。
- YAML生成の実コード要素（`YAMLDocument` / `YAMLWriter` / `ExportYAMLDocument`）を明記。

このドキュメントは、ユーザー要求に合わせて以下を明示します。

1. **Bundle読込から `.anim` 保存までの処理順を、分岐込みで追跡可能な形で列挙**
2. **アニメ変換経路で使われるDLLを漏れなく列挙**
3. **プロジェクト全体で確認できる主要ネイティブDLLも別枠で列挙**

---

## 1) エンドツーエンド処理（分岐込みの実行順）

## 1.1 入口

- CLI/GUIのいずれでも、まず `AssetsManager.LoadFiles(...)` または `LoadFolder(...)` が呼ばれる。
- `LoadFiles(...)` は対象群をキュー化し、各要素に対して `LoadFile(...)` を実行する。

## 1.2 LoadFileの分岐

`AssetsManager.LoadFile(FileReader reader)` は `reader.FileType` により分岐し、次の処理へ進む。

- `AssetsFile` -> `LoadAssetsFromMemory(...)`
- `BundleFile` -> `LoadBundleFile(...)`
- `WebFile` -> `LoadWebFile(...)`
- `ZipFile` -> `LoadZipFile(...)`
- `MhyFile` -> `LoadMhyFile(...)`
- `BlkFile/BlbFile` -> `LoadBlbFile(...)`
- `GZip/Brotli` の場合は展開後に `LoadFile(...)` を再帰

## 1.3 Bundle経路の中身

`LoadBundleFile(...)` -> `new BundleFile(reader, ...)` の内部で:

1. `ReadBundleHeader(...)`
2. `ReadHeader(...)`
3. `ReadBlocksInfoAndDirectory(...)`
4. `ReadBlocks(...)`（圧縮ブロック展開）
5. `ReadFiles(...)`（ノード展開）

展開された子エントリは以下へ回される。
- `AssetsFile` -> `LoadAssetsFromMemory(...)`
- ネストした `BundleFile` -> `LoadBundleFile(...)` 再帰
- resource類 -> `resourceFileReaders` 側へ保存

## 1.4 オブジェクト復元

- 全読込後に `ReadAssets()` を実行。
- `SerializedFile.m_Objects` を巡回し、`ObjectReader` を作成。
- `ClassIDType` switchで具象クラスを生成。
- `ClassIDType.AnimationClip` は `new AnimationClip(objectReader)`。

## 1.5 参照接続

- `ProcessAssets()` が GameObject/Component/SpriteAtlas 等の参照を接続。
- この時点で `AnimationClip` は export可能状態になる。

## 1.6 ExportAnimationClip

- CLI: `AssetStudio.CLI/Exporter.cs::ExportAnimationClip(...)`
- GUI: `AssetStudio.GUI/Exporter.cs::ExportAnimationClip(...)`

どちらも同じ処理:
1. 出力パス決定 (`TryExportFile(..., ".anim")`)
2. `var str = m_AnimationClip.Convert();`
3. 空文字でなければ `File.WriteAllText(exportFullPath, str)`

## 1.7 AnimationClip.Convert

`AnimationClipExtensions.Convert(this AnimationClip clip)`:

1. `!clip.m_Legacy || clip.m_MuscleClip != null` の時のみ変換実施
2. `AnimationClipConverter.Process(clip)`
3. 生成カーブを `m_RotationCurves / m_EulerCurves / m_PositionCurves / m_ScaleCurves / m_FloatCurves / m_PPtrCurves` に `Union`
4. `ConvertSerializedAnimationClip(clip)` を返す

## 1.8 Converter本体（ProcessInner）

`AnimationClipConverter.Process(...)` -> `ProcessInner()` の実行順:

1. `m_Clip = animationClip.m_MuscleClip.m_Clip`
2. `bindings = animationClip.m_ClipBindingConstant`
3. `tos = animationClip.FindTOS()`
4. `streamedFrames = m_Clip.m_StreamedClip.ReadData()`
5. 必要に応じて `ProcessACLClip(...)`
6. `ProcessStreams(...)`
7. `ProcessDenses(...)`
8. 必要に応じて `ProcessACLClip(...)`（SR系）
9. `m_ConstantClip != null` なら `ProcessConstant(...)`
10. `CreateCurves()`

## 1.9 パス解決・属性解決

- `FindTOS()` は `Animator` / `Animation` / `Avatar` 経由でTOS辞書を構築。
- `BuildTOS(GameObject parent, ...)` が再帰し `CRC.CalculateDigestUTF8(path)` をキー化。
- 各カーブ処理は `GetCurvePath(tos, binding.path)` で hash->path を解決。
- 属性名は `CustomCurveResolver.ToAttributeName(...)`。
- humanoidは `binding.GetHumanoidMuscle().ToAttributeString()`（`MuscleHelper.cs` 拡張）。

## 1.10 YAML出力

`ConvertSerializedAnimationClip(...)`:
1. `YAMLWriter writer = new YAMLWriter();`
2. `YAMLDocument doc = ExportYAMLDocument(animationClip);`
3. `writer.AddDocument(doc);`
4. `writer.Write(stringWriter);`
5. 生成文字列を Exporter が `.anim` として保存

## 1.11 YAMLオブジェクト構築の内訳（実装要素）

`AnimationClipExtensions` 側のYAML化は、次のクラス/メソッドで構成される。

- `ExportYAMLDocument(AnimationClip animationClip)`
  - `var document = new YAMLDocument();`
  - `var root = document.CreateMappingRoot();`
  - `root.Add(ClassIDType.AnimationClip.ToString(), node);`
- `ExportYAML(this AnimationClip clip, int[] version)`
  - `YAMLMappingNode` に `m_Name`, `m_RotationCurves`, `m_FloatCurves` などを `Add(...)`
- 最終出力:
  - `YAMLWriter.AddDocument(doc)`
  - `YAMLWriter.Write(stringWriter)`

関連クラスの定義ファイル:
- `AssetStudio/YAML/Base/YAMLDocument.cs`
- `AssetStudio/YAML/Base/YAMLWriter.cs`
- `AssetStudio/YAML/Base/YAMLMappingNode.cs`
- `AssetStudio/YAML/Base/YAMLSequenceNode.cs`
- `AssetStudio/YAML/Base/YAMLScalarNode.cs`

関連ユーティリティ:
- `AssetStudio/YAML/Utils/Extensions/StringBuilderExtensions.cs`

---

## 2) `.anim` 変換経路で使われるDLL（必須）

## 2.1 ACL系ネイティブDLL

`AssetStudio.Utility/ACL/ACL.cs`:
- `acl`
- `sracl`
- `acldb`

各クラス:
- `ACL` / `SRACL` / `DBACL`
- 静的コンストラクタで `DllLoader.PreloadDll(DLL_NAME)` を実行
- `DllImport(DLL_NAME, CallingConvention = Cdecl)` で復号関数を呼び出す

## 2.2 実行時の選択分岐

`AssetStudio.Utility/ACL/ACLExtensions.cs::Process(...)`:
- SRグループ -> `SRACL.DecompressAll(...)`（`sracl`）
- `GIACLClip` -> `DBACL.DecompressTracks(...)`（`acldb`）
- `MHYACLClip` -> `ACL.DecompressAll(...)`（`acl`）

## 2.3 DllLoaderのOS別解決

`AssetStudio.PInvoke/DllLoader.cs`:
- Windows: `"{dllName}.dll"`
- Linux: `"lib{dllName}.so"`
- macOS: `"lib{dllName}.dylib"`

利用するシステムAPI:
- Windows: `LoadLibraryEx` from `kernel32.dll`
- POSIX: `dlopen` / `dlerror` from `libdl`

---

## 3) プロジェクト全体で確認できる主要ネイティブDLL（参考）

`.anim` 変換の必須ではないが、リポジトリ内の `DllImport` で確認できる代表例:

- `HLSLDecompiler`（Shader変換経路）
- `gdi32.dll`（Fontヘルパ）
- `kernel32.dll` / `user32.dll`（ConsoleHelper等）
- FMOD系（`fmod` バインディング経路）
- FBX wrapper経路で使用されるFBXネイティブDLL

※ この節は「全体参照」であり、`.anim` 出力だけに必要なDLLは **2章**。

---

## 4) 追跡対象ファイル（最小セット）

### 読込側
- `AssetStudio/AssetsManager.cs`
- `AssetStudio/BundleFile.cs`
- `AssetStudio/SerializedFile.cs`
- `AssetStudio/ObjectReader.cs`
- `AssetStudio/ClassIDType.cs`
- `AssetStudio/Classes/AnimationClip.cs`

### 変換側
- `AssetStudio.CLI/Exporter.cs`
- `AssetStudio.GUI/Exporter.cs`
- `AssetStudio.Utility/YAML/AnimationClipExtensions.cs`
- `AssetStudio.Utility/YAML/AnimationClipConverter.cs`
- `AssetStudio.Utility/YAML/CustomCurveResolver.cs`
- `AssetStudio.Utility/YAML/MuscleHelper.cs`
- `AssetStudio.Utility/YAML/AnimationClipExtensions.cs` の `ExportYAMLDocument` / `ExportYAML`
- `AssetStudio/YAML/Base/YAMLWriter.cs`
- `AssetStudio/YAML/Base/YAMLDocument.cs`
- `AssetStudio/YAML/Base/YAMLMappingNode.cs`
- `AssetStudio/YAML/Base/YAMLSequenceNode.cs`
- `AssetStudio/YAML/Base/YAMLScalarNode.cs`
- `AssetStudio/YAML/Utils/Extensions/StringBuilderExtensions.cs`

### DLL側
- `AssetStudio.Utility/ACL/ACL.cs`
- `AssetStudio.Utility/ACL/ACLExtensions.cs`
- `AssetStudio.PInvoke/DllLoader.cs`

---

## 5) 誤解しやすい点

- `MuscleHelper.cs` は存在するが、`MuscleHelper` 同名クラスではない（拡張群）。
- `StringBuilderExtensions` は存在する（YAMLユーティリティ拡張）。
- `HLSLDecompiler` は Shader経路で、`.anim` 変換中核とは別。

---

## 6) 実コード抜粋（YAML/DLLの実体）

### 6.1 Exporter -> Convert -> WriteAllText

```csharp
// AssetStudio.CLI/Exporter.cs (GUI側も同形)
var str = m_AnimationClip.Convert();
if (!string.IsNullOrEmpty(str))
{
    File.WriteAllText(exportFullPath, str);
}
```

### 6.2 ConvertSerializedAnimationClip

```csharp
// AssetStudio.Utility/YAML/AnimationClipExtensions.cs
public static string ConvertSerializedAnimationClip(AnimationClip animationClip)
{
    var sb = new StringBuilder();
    using (var stringWriter = new StringWriter(sb))
    {
        YAMLWriter writer = new YAMLWriter();
        YAMLDocument doc = ExportYAMLDocument(animationClip);
        writer.AddDocument(doc);
        writer.Write(stringWriter);
        return sb.ToString();
    }
}
```

### 6.3 ExportYAMLDocument / ExportYAML

```csharp
public static YAMLDocument ExportYAMLDocument(AnimationClip animationClip)
{
    var document = new YAMLDocument();
    var root = document.CreateMappingRoot();
    root.Tag = ((int)ClassIDType.AnimationClip).ToString();
    root.Anchor = ((int)ClassIDType.AnimationClip * 100000).ToString();
    var node = animationClip.ExportYAML(animationClip.version);
    root.Add(ClassIDType.AnimationClip.ToString(), node);
    return document;
}
```

```csharp
public static YAMLMappingNode ExportYAML(this AnimationClip clip, int[] version)
{
    var node = new YAMLMappingNode();
    node.Add(nameof(clip.m_Name), clip.m_Name);
    node.Add(nameof(clip.m_RotationCurves), clip.m_RotationCurves.ExportYAML(version));
    node.Add(nameof(clip.m_FloatCurves), clip.m_FloatCurves.ExportYAML(version));
    node.Add(nameof(clip.m_PPtrCurves), clip.m_PPtrCurves.ExportYAML(version));
    return node;
}
```

### 6.4 ACL DLL分岐（実コード形）

```csharp
// AssetStudio.Utility/ACL/ACLExtensions.cs
if (game.Type.IsSRGroup())
{
    SRACL.DecompressAll(aclClip.m_ClipData, out values, out times);
}
else
{
    switch (m_ACLClip)
    {
        case GIACLClip giaclClip:
            DBACL.DecompressTracks(giaclClip.m_ClipData, giaclClip.m_DatabaseData, out values, out times);
            break;
        case MHYACLClip mhyaclClip:
            ACL.DecompressAll(mhyaclClip.m_ClipData, out values, out times);
            break;
    }
}
```
