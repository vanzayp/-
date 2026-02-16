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
全面書き直し: Bundle読込→AnimationClip変換→`.anim`出力（責任追跡付き）

## 0. この章の目的
- 「どのコードを使うか」「どの順に処理されるか」を曖昧語なしで示す。
- これまでの説明で不足が出た点（最小経路だけ示す、分岐を省略する、パス解決を薄く書く）を明示し、再発防止の形で固定化する。

## 1. 不足が出た事実（記録）
1. 最小経路中心の説明で、`LoadFile` の全分岐を省略した。
2. `FindTOS -> GetCurvePath -> ToAttributeName` の連鎖を断片的に書いた。
3. 入口（Load/Export）と終点（WriteAllText）の間を、関数単位で一列に固定しきれていなかった。

## 2. 終点定義
- 終点は `Exporter.ExportAnimationClip` 内の `File.WriteAllText(exportFullPath, str)`。

## 3. 読込処理（A）
### 3-1. 入口
- `AssetsManager.LoadFiles`
- `AssetsManager.LoadFolder`
- `AssetsManager.Load`
- `AssetsManager.LoadFile(string)`
- `AssetsManager.LoadFile(FileReader)`

### 3-2. `LoadFile(FileReader)` の全分岐（省略なし）
- `AssetsFile` -> `LoadAssetsFile`
- `BundleFile` -> `LoadBundleFile`
- `WebFile` -> `LoadWebFile`
- `GZipFile` -> `DecompressGZip` -> `LoadFile`
- `BrotliFile` -> `DecompressBrotli` -> `LoadFile`
- `ZipFile` -> `LoadZipFile`
- `BlockFile` -> `LoadBlockFile`
- `BlkFile` -> `LoadBlkFile`
- `MhyFile` -> `LoadMhyFile`

### 3-3. Bundleルート
- `LoadBundleFile` -> `new BundleFile(reader, Game)`
- `BundleFile` 側で:
  1. `ReadBundleHeader`
  2. `ReadHeader`
  3. `ReadBlocksInfoAndDirectory`
  4. `ReadBlocks`
  5. `ReadFiles`
- `bundleFile.fileList` を走査し、`AssetsFile` は `LoadAssetsFromMemory`、それ以外は `resourceFileReaders` へ。

### 3-4. 派生ルート
- `LoadWebFile`:
  - `AssetsFile` -> `LoadAssetsFromMemory`
  - `BundleFile` -> `LoadBundleFile`
  - `WebFile` -> `LoadWebFile`（再帰）
  - `ResourceFile` -> キャッシュ
- `LoadZipFile`:
  - split結合後 `LoadFile`
  - 通常entry展開後 `LoadFile`
  - `ResourceFile` はキャッシュ
- `LoadBlockFile` / `LoadBlkFile` / `LoadMhyFile`:
  - 内部ストリームを `Bundle/Mhy/...` へ再分岐

### 3-5. AnimationClip復元
- `ReadAssets` で `ObjectReader` を作成。
- `ClassIDType.AnimationClip` のとき `new AnimationClip(objectReader)`。

## 4. 変換処理（B）
### 4-1. 入口
- `AssetStudio.CLI/Exporter.ExportAnimationClip`
- `AssetStudio.GUI/Exporter.ExportAnimationClip`

処理は共通:
1. `TryExportFile(..., ".anim", out exportFullPath)`
2. `m_AnimationClip.Convert()`
3. 空文字なら失敗
4. `File.WriteAllText(exportFullPath, str)`

### 4-2. `AnimationClipExtensions.Convert`
1. 条件 `!m_Legacy || m_MuscleClip != null`
2. 真なら `AnimationClipConverter.Process(clip)`
3. 返却カーブを `m_RotationCurves / m_EulerCurves / m_PositionCurves / m_ScaleCurves / m_FloatCurves / m_PPtrCurves` へ `Union`
4. `ConvertSerializedAnimationClip` を返却

### 4-3. `AnimationClipConverter.ProcessInner` の全順序
1. `m_Clip = animationClip.m_MuscleClip.m_Clip`
2. `bindings = animationClip.m_ClipBindingConstant`
3. `tos = animationClip.FindTOS()`
4. `streamedFrames = m_Clip.m_StreamedClip.ReadData()`
5. `lastDenseFrame / lastSampleFrame / lastFrame` 計算
6. `m_ACLClip.IsSet && !SR` なら `ProcessACLClip`
7. `ProcessStreams`
8. `ProcessDenses`
9. `m_ACLClip.IsSet && SR` なら `ProcessACLClip`
10. `m_ConstantClip != null` なら `ProcessConstant`
11. `CreateCurves`

### 4-4. フェーズ責務
- `ProcessStreams`: streamedキー展開、binding取得、Transform/Default/Custom振分
- `ProcessDenses`: dense配列を時間軸展開、同振分
- `ProcessACLClip`: ACL圧縮値展開、同振分
- `ProcessConstant`: 終端補完
- `CreateCurves`: 内部辞書を最終カーブへコミット

## 5. パス/属性解決（省略なし）
1. `FindTOS` が `Dictionary<uint,string>` を構築（初期値 `{0:""}`）
2. `AddAvatarTOS` / `AddAnimatorTOS` / `AddAnimationTOS` で候補追加
3. `BuildTOS` が `parent/child` パスを作り `CRC.CalculateDigestUTF8(path)` でハッシュ化
4. `GetCurvePath(tos, binding.path)`
   - hit: 実path
   - miss: `path_<hash>`
5. `AddCustomCurve` が `CustomCurveResolver.ToAttributeName(customType, attribute, path)` を呼び最終プロパティ名へ

## 6. YAML終端鎖
1. `ConvertSerializedAnimationClip`
2. `ExportYAMLDocument`
3. `YAMLWriter.AddDocument`
4. `YAMLWriter.Write`
5. Exporterへ戻る
6. `File.WriteAllText(... .anim)`

## 7. 使用ソースコード（この処理に直接登場）
### 7-1. 読込側
- `AssetStudio/AssetsManager.cs`
- `AssetStudio/BundleFile.cs`
- `AssetStudio/SerializedFile.cs`
- `AssetStudio/ObjectReader.cs`
- `AssetStudio/Classes/AnimationClip.cs`

### 7-2. 変換側
- `AssetStudio.CLI/Exporter.cs`
- `AssetStudio.GUI/Exporter.cs`
- `AssetStudio.Utility/YAML/AnimationClipExtensions.cs`
- `AssetStudio.Utility/YAML/AnimationClipConverter.cs`
- `AssetStudio.Utility/YAML/CustomCurveResolver.cs`

## 8. 再発防止チェック
- [x] `LoadFile` 全分岐を記載
- [x] 入口→終点を関数順で固定
- [x] パス解決連鎖を記載
- [x] YAML終端鎖を記載
- [x] 使用ソースコード名を列挙
