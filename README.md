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
Bundle読込とAnimation変換を最初から整理（完全版）

この章は、**AssetStudioの「Bundle読込」と「.anim変換」を責務分離して、実コードの順序で追跡**するための再構成版です。

## 0. 結論（先に要点）

- **A: Bundle読込系** は「ファイル -> 復号/解凍 -> SerializedFile -> `AssetStudio.AnimationClip`生成」まで。
- **B: Animation変換系** は「`AssetStudio.AnimationClip` -> カーブ再構築 -> YAML文字列 -> `.anim`書込」まで。
- AとBの橋渡しは `AssetStudio.AnimationClip` オブジェクトのみ。

---

## A. Bundle読込処理（Input / Decode / Deserialize）

### A-1. エントリ
- `AssetsManager.LoadFiles(...)` / `AssetsManager.LoadFolder(...)`
- 分割ファイル処理: `MergeSplitAssets`, `ProcessingSplitFiles`
- ループで `LoadFile(...)` を呼び出し

### A-2. ファイルタイプ分岐（`AssetsManager.LoadFile`）
- `AssetsFile` -> `LoadAssetsFile`
- `BundleFile` -> `LoadBundleFile`
- `WebFile` -> `LoadWebFile`
- `ZipFile` -> `LoadZipFile`
- `Block/Blk/Mhy` -> `LoadBlockFile` / `LoadBlkFile` / `LoadMhyFile`

### A-3. Bundle解析（`BundleFile`）
1. `ReadBundleHeader`
2. `ReadHeader`（必要なら `ReadUnityCN`）
3. `ReadBlocksInfoAndDirectory`
4. `ReadBlocks`（LZMA/LZ4/Zstd/None などを展開）
5. `ReadFiles`（`StreamFile`群へ展開）

### A-4. Assets化
- `LoadBundleFile` が `bundleFile.fileList` を走査
- 各streamを `FileReader` 化
- `AssetsFile` は `LoadAssetsFromMemory`
- 非AssetsFileは `resourceFileReaders` にキャッシュ

### A-5. オブジェクト復元
- `ReadAssets` で `ObjectReader` を作成
- `ClassIDType` で実クラスへ分岐
- `ClassIDType.AnimationClip` は `new AnimationClip(objectReader)`

### A-6. Aの成果物
- `assetsFileList`
- `SerializedFile.Objects` に `AssetStudio.AnimationClip` が載る

---

## B. Animation変換処理（Convert / Serialize / Write）

### B-1. エントリ
- `Exporter.ExportAnimationClip`（CLI/GUI）
- `m_AnimationClip.Convert()` を呼ぶ

### B-2. `AnimationClipExtensions.Convert`
- 条件 `!clip.m_Legacy || clip.m_MuscleClip != null` で `AnimationClipConverter.Process(clip)`
- 返ってきた曲線群を既存カーブへ `Union`
- `ConvertSerializedAnimationClip` でYAML化

### B-3. SAM原文（生データ）
ここでいうSAM原文は `AnimationClip` 内の次の生データ群:
- `m_StreamedClip`
- `m_DenseClip`
- `m_ACLClip`
- `m_ConstantClip`

### B-4. 生データ -> カーブの変換順（実コード順）
1. `ProcessInner` で `m_MuscleClip.m_Clip`, `m_ClipBindingConstant`, `FindTOS()` を取得
2. `m_StreamedClip.ReadData()` でstreamフレームを復元
3. `ProcessACLClip`（条件付き）でACLデータを通常値列へ復元
4. `ProcessStreams` でstreamキーをbindingへ割当
5. `ProcessDenses` でdense配列を時系列キーへ展開
6. `ProcessConstant` でconstant値を終端へ補完
7. `CreateCurves` で Transform/Float/PPtr カーブへ集約
8. `ConvertSerializedAnimationClip` -> `ExportYAMLDocument` -> `YAMLWriter.Write`

### B-5. パス処理（重要）
1. `FindTOS()` が `Dictionary<uint,string>` を作る
   - ルートは `0 -> ""`
   - 子ノードを `parent/child` で連結し、`CRC.CalculateDigestUTF8(path)` をキー化
2. 各処理で `binding.path` を `GetCurvePath(tos, binding.path)` に通す
3. 見つからないハッシュは `path_<hash>`（UnknownPath）
4. `AddCustomCurve` が `CustomCurveResolver.ToAttributeName(type, attribute, path)` で最終プロパティ名へ変換
5. UnknownPathはフォールバック名で出力継続（変換は止めない）

### B-6. Bの成果物
- `.anim`（YAMLテキスト）

---

## A/B 接続点（実装境界）

- 接続データ: `AssetStudio.AnimationClip`
- 接続呼び出し: `AnimationClipExtensions.Convert`
- 境界を守る実装順:
  1. Aを先に完成（Bundle -> AnimationClip抽出）
  2. Bを完成（AnimationClip -> .anim）
  3. 最後にA/Bを接続

---

## 追跡に使う主要ソース（最小セット）

### A側
- `AssetStudio/AssetsManager.cs`
- `AssetStudio/BundleFile.cs`
- `AssetStudio/FileReader.cs`
- `AssetStudio/SerializedFile.cs`
- `AssetStudio/ObjectReader.cs`
- `AssetStudio/Classes/AnimationClip.cs`
- `AssetStudio/Crypto/*`, `AssetStudio/LZ4/*`, `AssetStudio/Brotli/*`, `AssetStudio/7zip/*`

### B側
- `AssetStudio.Utility/YAML/AnimationClipConverter.cs`
- `AssetStudio.Utility/YAML/AnimationClipExtensions.cs`
- `AssetStudio.Utility/YAML/CustomCurveResolver.cs`
- `AssetStudio.Utility/YAML/MuscleHelper.cs`
- `AssetStudio/YAML/Base/*`
- `AssetStudio/YAML/Utils/Extensions/*`
- `AssetStudio.CLI/Exporter.cs`, `AssetStudio.GUI/Exporter.cs`
