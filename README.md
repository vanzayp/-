# Studio

# NOTICE: Project has been temporarily suspended till further notice.

Check out the [original AssetStudio project](https://github.com/Perfare/AssetStudio) for more information.

Note: Requires Internet connection to fetch asset_index jsons.
_____________________________________________________________________________________________________________________________
How to use:

Check the tutorial [here](https://gist.github.com/Modder4869/0f5371f8879607eb95b8e63badca227e) (Thanks to Modder4869 for the tutorial)
_____________________________________________________________________________________________________________________________
CLI Version:
```
Description:

Usage:
  AssetStudioCLI <input_path> <output_path> [options]

Arguments:
  <input_path>   Input file/folder.
  <output_path>  Output folder.

Options:
  --silent                                                Hide log messages.
  --type <Texture2D|Sprite|etc..>                         Specify unity class type(s)
  --filter <filter>                                       Specify regex filter(s).
  --game <BH3|CB1|CB2|CB3|GI|SR|TOT|ZZZ> (REQUIRED)       Specify Game.
  --map_op <AssetMap|Both|CABMap|None>                    Specify which map to build. [default: None]
  --map_type <JSON|XML>                                   AssetMap output type. [default: XML]
  --map_name <map_name>                                   Specify AssetMap file name.
  --group_assets_type <ByContainer|BySource|ByType|None>  Specify how exported assets should be grouped. [default: 0]
  --no_asset_bundle                                       Exclude AssetBundle from AssetMap/Export.
  --no_index_object                                       Exclude IndexObject/MiHoYoBinData from AssetMap/Export.
  --xor_key <xor_key>                                     XOR key to decrypt MiHoYoBinData.
  --ai_file <ai_file>                                     Specify asset_index json file path (to recover GI containers).
  --version                                               Show version information
  -?, -h, --help                                          Show help and usage information
```
_____________________________________________________________________________________________________________________________
NOTES:
```
- in case of any "MeshRenderer/SkinnedMeshRenderer" errors, make sure to enable "Disable Renderer" option in "Export Options" before loading assets.
- in case of need to export models/animators without fetching all animations, make sure to enable "Ignore Controller Anim" option in "Options -> Export Options" before loading assets.
```
_____________________________________________________________________________________________________________________________
Special Thank to:
- Perfare: Original author.
- Khang06: [Project](https://github.com/khang06/genshinblkstuff) for extraction.
- Radioegor146: [Asset-indexes](https://github.com/radioegor146/gi-asset-indexes) for recovered/updated asset_index's.
- Ds5678: [AssetRipper](https://github.com/AssetRipper/AssetRipper)[[discord](https://discord.gg/XqXa53W2Yh)] for information about Asset Formats & Parsing.
- mafaca: [uTinyRipper](https://github.com/mafaca/UtinyRipper) for `YAML` and `AnimationClipConverter`. 


_____________________________________________________________________________________________________________________________
Bundle読込 → AnimationClip変換 → `.anim` 出力（再整理版）

この章は、継ぎ足しをやめて **処理を1本の経路として再整理** したものです。

## 1. 終点（ゴール）
- 終点は `Exporter.ExportAnimationClip` 内の `File.WriteAllText(exportFullPath, str)`。
- ここで `.anim`（YAML文字列）がファイルとして保存される。

## 2. 読込パイプライン（A: Bundle/Input 側）

### A-1. 入口
1. `AssetsManager.LoadFiles` または `AssetsManager.LoadFolder`
2. `Load` で内部キューを処理
3. `LoadFile(string)` -> `LoadFile(FileReader)`

### A-2. FileType分岐
`LoadFile(FileReader)` で下記へ分岐:
- `AssetsFile` -> `LoadAssetsFile`
- `BundleFile` -> `LoadBundleFile`
- `WebFile` -> `LoadWebFile`
- `ZipFile` -> `LoadZipFile`
- `BlockFile` -> `LoadBlockFile`
- `BlkFile` -> `LoadBlkFile`
- `MhyFile` -> `LoadMhyFile`

### A-3. Bundle処理
`LoadBundleFile` では `new BundleFile(reader, Game)` を作成し、`BundleFile` 側で:
1. `ReadBundleHeader`
2. `ReadHeader`
3. `ReadBlocksInfoAndDirectory`
4. `ReadBlocks`
5. `ReadFiles`
を実行して `fileList` を展開。

### A-4. Assets化とオブジェクト復元
- `fileList` の `AssetsFile` は `LoadAssetsFromMemory` で `SerializedFile` 化。
- `ReadAssets` が `ObjectReader` を使ってオブジェクトを復元。
- `ClassIDType.AnimationClip` のとき `new AnimationClip(objectReader)` が生成される。

## 3. 変換パイプライン（B: Animation 側）

### B-1. 入口
- `AssetStudio.CLI/Exporter.ExportAnimationClip`
- `AssetStudio.GUI/Exporter.ExportAnimationClip`

どちらも:
1. `TryExportFile(..., ".anim", ...)`
2. `m_AnimationClip.Convert()`
3. 文字列が空でなければ `File.WriteAllText`

### B-2. Convert外枠
`AnimationClipExtensions.Convert`:
1. 条件 `!m_Legacy || m_MuscleClip != null` で `AnimationClipConverter.Process(clip)`
2. 返却カーブを既存 `m_RotationCurves / m_EulerCurves / m_PositionCurves / m_ScaleCurves / m_FloatCurves / m_PPtrCurves` に `Union`
3. `ConvertSerializedAnimationClip` で YAML文字列化

### B-3. 変換本体（`AnimationClipConverter.ProcessInner`）
実行順序:
1. `m_Clip`, `bindings`, `tos = FindTOS()` 取得
2. `m_StreamedClip.ReadData()`
3. ACL先行処理（条件）
4. `ProcessStreams`
5. `ProcessDenses`
6. ACL後行処理（SR条件）
7. `ProcessConstant`（条件）
8. `CreateCurves`

### B-4. フェーズ別の役割
- `ProcessStreams`: streamキー列を binding へ割当、Transform/Default/Custom に振分
- `ProcessDenses`: dense配列を時間軸へ展開して同様に振分
- `ProcessACLClip`: ACL圧縮データを展開して同様に振分
- `ProcessConstant`: 終端値を補完

## 4. パス解決（変換で必須）

1. `FindTOS()` が `Dictionary<uint, string>` を作成
2. 各フェーズで `GetCurvePath(tos, binding.path)` を使い、ハッシュから文字列pathへ変換
3. 見つからない場合は `path_<hash>`
4. `AddCustomCurve` -> `CustomCurveResolver.ToAttributeName(customType, attribute, path)` で最終属性名を決定

## 5. YAML生成と書込終端

1. `ConvertSerializedAnimationClip`
2. `ExportYAMLDocument`
3. `YAMLWriter.AddDocument`
4. `YAMLWriter.Write`
5. Exporterが `File.WriteAllText(... .anim)`

## 6. 使用コード（この経路で使う主要ファイル）

### 読込側
- `AssetStudio/AssetsManager.cs`
- `AssetStudio/BundleFile.cs`
- `AssetStudio/SerializedFile.cs`
- `AssetStudio/ObjectReader.cs`
- `AssetStudio/Classes/AnimationClip.cs`

### 変換側
- `AssetStudio.CLI/Exporter.cs`
- `AssetStudio.GUI/Exporter.cs`
- `AssetStudio.Utility/YAML/AnimationClipExtensions.cs`
- `AssetStudio.Utility/YAML/AnimationClipConverter.cs`
- `AssetStudio.Utility/YAML/CustomCurveResolver.cs`


## 7. 非省略版: 全分岐ルート（省略語なし）

> ここは「最小経路」ではなく、`AssetsManager` と変換側の分岐をそのまま列挙する。

### 7-1. `AssetsManager.LoadFile(FileReader)` の全分岐
- `FileType.AssetsFile` -> `LoadAssetsFile`
- `FileType.BundleFile` -> `LoadBundleFile`
- `FileType.WebFile` -> `LoadWebFile`
- `FileType.GZipFile` -> `DecompressGZip` -> `LoadFile`
- `FileType.BrotliFile` -> `DecompressBrotli` -> `LoadFile`
- `FileType.ZipFile` -> `LoadZipFile`
- `FileType.BlockFile` -> `LoadBlockFile`
- `FileType.BlkFile` -> `LoadBlkFile`
- `FileType.MhyFile` -> `LoadMhyFile`

### 7-2. `LoadBundleFile` 内部
- `new BundleFile(reader, Game)`
- `bundleFile.fileList` を走査
  - `subReader.FileType == AssetsFile` -> `LoadAssetsFromMemory`
  - それ以外 -> `resourceFileReaders.TryAdd`

### 7-3. `BundleFile` コンストラクタの全分岐
- `signature == UnityArchive` -> 現状TODO
- `signature in {UnityWeb, UnityRaw}`
  - version 6 なら UnityFS ルートへ
  - それ以外: `ReadHeaderAndBlocksInfo` -> `ReadBlocksAndDirectory` -> `ReadFiles`
- `signature in {UnityFS, ENCR}`
  - `ReadHeader`
  - `game.Type.IsUnityCN()` なら `ReadUnityCN`
  - `ReadBlocksInfoAndDirectory`
  - `ReadBlocks`
  - `ReadFiles`

### 7-4. `LoadWebFile` 内部再帰分岐
Webエントリごとに `subReader.FileType` で:
- `AssetsFile` -> `LoadAssetsFromMemory`
- `BundleFile` -> `LoadBundleFile`
- `WebFile` -> `LoadWebFile`（再帰）
- `ResourceFile` -> `resourceFileReaders.TryAdd`

### 7-5. `LoadZipFile` 内部
- splitエントリ (`.splitN`) を結合して `LoadFile(entryReader)`
- 通常entryをメモリ展開して `LoadFile(entryReader)`
- `ResourceFile` の場合は `resourceFileReaders.TryAdd`

### 7-6. `LoadBlockFile` / `LoadBlkFile` / `LoadMhyFile`
- `LoadBlockFile`: offsetごとに `ENCR|Bundle -> LoadBundleFile`, `Blb -> LoadBlbFile`, `Mhy -> LoadMhyFile`
- `LoadBlkFile`: 復号後offsetごとに `Bundle -> LoadBundleFile`, `Mhy -> LoadMhyFile`
- `LoadMhyFile`: `fileList` 走査、`AssetsFile -> LoadAssetsFromMemory`, その他は `resourceFileReaders.TryAdd`

### 7-7. `ReadAssets` の型分岐（Animation出力に関係する部分）
- `ClassIDType.AnimationClip` -> `new AnimationClip(objectReader)`
- 関連参照構築は `ProcessAssets`（GameObject-Component関連付け）で実施

### 7-8. Exporter側の全経路
- `ExportAnimationClip`（CLI/GUI）
  1. `TryExportFile(..., ".anim", out exportFullPath)`
  2. `m_AnimationClip.Convert()`
  3. `string.IsNullOrEmpty(str)` なら失敗
  4. `File.WriteAllText(exportFullPath, str)`
- `ExportConvertFile`（CLI/GUI）
  - `switch(item.TypeString)`
  - `ClassIDType.AnimationClip` の場合 `ExportAnimationClip` に合流

### 7-9. `AnimationClipExtensions.Convert` の全処理
1. `!clip.m_Legacy || clip.m_MuscleClip != null` 判定
2. 真なら `AnimationClipConverter.Process(clip)`
3. `converter.Rotations/Eulers/Translations/Scales/Floats/PPtrs` を既存カーブへ `Union`
4. `ConvertSerializedAnimationClip(clip)` を返却

### 7-10. `AnimationClipConverter.ProcessInner` の全順序
1. `m_Clip = animationClip.m_MuscleClip.m_Clip`
2. `bindings = animationClip.m_ClipBindingConstant`
3. `tos = animationClip.FindTOS()`
4. `streamedFrames = m_Clip.m_StreamedClip.ReadData()`
5. `lastDenseFrame` / `lastSampleFrame` / `lastFrame` 計算
6. `m_Clip.m_ACLClip.IsSet && !SR` なら `ProcessACLClip`
7. `ProcessStreams`
8. `ProcessDenses`
9. `m_Clip.m_ACLClip.IsSet && SR` なら `ProcessACLClip`
10. `m_Clip.m_ConstantClip != null` なら `ProcessConstant`
11. `CreateCurves`

### 7-11. パス解決の全関数鎖
- `FindTOS`（初期値 `{0:""}`）
- `AddAvatarTOS` / `AddAnimatorTOS` / `AddAnimationTOS`
- `BuildTOS`（`parent/child` を構築し `CRC.CalculateDigestUTF8(path)` をキー化）
- `GetCurvePath(tos, binding.path)`
  - hit: 実path
  - miss: `path_<hash>`
- `AddCustomCurve` -> `CustomCurveResolver.ToAttributeName(customType, attribute, path)`

### 7-12. YAML出力の全終端鎖
- `ConvertSerializedAnimationClip`
- `ExportYAMLDocument`
- `YAMLWriter.AddDocument`
- `YAMLWriter.Write`
- `Exporter.ExportAnimationClip` へ戻る
- `File.WriteAllText(... .anim)`
