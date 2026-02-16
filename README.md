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

### B-1. 入口（Exporter）
- `AssetStudio.CLI/Exporter.cs` と `AssetStudio.GUI/Exporter.cs` の `ExportAnimationClip` が入口。
- `m_AnimationClip.Convert()` を呼び、戻り値文字列を `File.WriteAllText(... .anim)` で保存。

### B-2. Convertの外枠（`AnimationClipExtensions.Convert`）
1. 対象が `!m_Legacy || m_MuscleClip != null` の場合、`AnimationClipConverter.Process(clip)` を実行。
2. `Process` が生成したカーブ群を既存カーブへ `Union`:
   - `m_RotationCurves`
   - `m_EulerCurves`
   - `m_PositionCurves`
   - `m_ScaleCurves`
   - `m_FloatCurves`
   - `m_PPtrCurves`
3. `ConvertSerializedAnimationClip` で YAML 文字列へ変換。

### B-3. 変換の中心（`AnimationClipConverter.ProcessInner`）
`AnimationClipConverter.Process` は内部で `ProcessInner` を実行し、以下の順で処理する。

1. **前処理**
   - `m_Clip = animationClip.m_MuscleClip.m_Clip`
   - `bindings = animationClip.m_ClipBindingConstant`
   - `tos = animationClip.FindTOS()`
2. **生データ展開**
   - `streamedFrames = m_Clip.m_StreamedClip.ReadData()`
   - `lastFrame`（dense/stream 由来）を計算
3. **ACL先行処理（条件付き）**
   - `m_ACLClip.IsSet && !SR` の場合、先に `ProcessACLClip` を実行
4. **stream処理**
   - `ProcessStreams(streamedFrames, bindings, tos, sampleRate)`
5. **dense処理**
   - `ProcessDenses(m_Clip, bindings, tos)`
6. **ACL後行処理（SR条件）**
   - `m_ACLClip.IsSet && SR` の場合、ここで `ProcessACLClip`
7. **constant処理**
   - `m_ConstantClip != null` の場合、`ProcessConstant`
8. **カーブ確定**
   - `CreateCurves()` で内部辞書から最終カーブ配列へ反映

### B-4. 各フェーズで何を変換しているか
- `ProcessStreams`
  - streamフレームのキー列を走査し、`bindings.FindBinding(index)` でバインディング特定。
  - Transform系は `AddTransformCurve`、それ以外は `AddDefaultCurve`/`AddCustomCurve`。
- `ProcessDenses`
  - dense配列を `frameIndex / sampleRate` の時間軸で展開。
  - binding種別ごとに Transform/Default/Custom へ分配。
- `ProcessACLClip`
  - `acl.Process(...)` でACL圧縮値を展開し、時間付きキー列へ変換。
- `ProcessConstant`
  - constant値を `lastFrame` に配置し、不足終端を補完。

### B-5. パス解決（変換で必須）
1. `FindTOS()` が `Dictionary<uint, string>` を構築（`CRC.CalculateDigestUTF8(path)` -> `path`）。
2. 各フェーズで `binding.path` を `GetCurvePath(tos, binding.path)` で文字列パスへ変換。
3. 見つからないハッシュは `path_<hash>` を返して継続。
4. `AddCustomCurve` は `CustomCurveResolver.ToAttributeName(customType, attribute, path)` で最終プロパティ名へ解決。

### B-6. YAML化（`ConvertSerializedAnimationClip`）
1. `ExportYAMLDocument(animationClip)` でYAMLドキュメント作成
2. `YAMLWriter.AddDocument(doc)`
3. `YAMLWriter.Write(StringWriter)`
4. 生成文字列をExporterが `.anim` として保存

### B-7. Bの成果物
- `.anim`（Unity YAML形式）


### B-8. 抜けゼロ検証表（コード照合）

| フェーズ | 入力 | 主要分岐 | パス利用 | 出力 |
|---|---|---|---|---|
| `ProcessStreams` | `streamFrames`, `bindings`, `tos`, `sampleRate` | `Transform` / `customType==None` / `Custom` | `GetCurvePath(tos, binding.path)` | Transform/Default/Customキー追加 |
| `ProcessDenses` | `dense.m_SampleArray`, `bindings`, `tos` | `Transform` / `customType==None` / `Custom` | `GetCurvePath(tos, binding.path)` | 時系列キー追加 |
| `ProcessACLClip` | `acl.Process(...)` 展開値, `bindings`, `tos` | `Transform` / `customType==None` / `Custom` | `GetCurvePath(tos, binding.path)` | ACL由来キー追加 |
| `ProcessConstant` | `constant.data`, `bindings`, `tos`, `lastFrame` | `Transform` / `customType==None` / `Custom` | `GetCurvePath(tos, binding.path)` | 終端補完キー追加 |
| `CreateCurves` | 内部辞書 (`m_translations` など) | なし | なし | `Translations/Rotations/...` へ確定 |
| `ConvertSerializedAnimationClip` | `AnimationClip` | なし | なし | YAML文字列 |

### B-9. 未記載項目なしチェックリスト

- [x] 入口 (`ExportAnimationClip`)
- [x] `AnimationClipExtensions.Convert` の条件分岐
- [x] `ProcessInner` の実行順序
- [x] `ProcessStreams` / `ProcessDenses` / `ProcessACLClip` / `ProcessConstant`
- [x] `FindTOS` と `GetCurvePath`
- [x] UnknownPath (`path_<hash>`) フォールバック
- [x] `CustomCurveResolver.ToAttributeName`
- [x] `CreateCurves` での最終反映
- [x] `ConvertSerializedAnimationClip` -> `ExportYAMLDocument` -> `YAMLWriter.Write`
- [x] `.anim` ファイル書き込み (`File.WriteAllText`)

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


## C. 始点からの再追跡ログ（実コードを1本ずつ辿った記録）

> ここは「どのコードを起点に、どの関数を通って、何を作るか」を再追跡した監査ログ。
> A/Bの要約ではなく、始点ごとの通過点を固定化する。

### C-1. 始点: CLIの単体AnimationClip出力
1. `AssetStudio.CLI/Program.cs` でCLI引数処理開始
2. `AssetStudio.CLI/Studio.cs` で読み込み・オブジェクト列挙
3. `AssetStudio.CLI/Exporter.ExportAnimationClip`
4. `AnimationClipExtensions.Convert`
5. `AnimationClipConverter.Process` -> `ProcessInner`
6. `ConvertSerializedAnimationClip` -> `YAMLWriter.Write`
7. `File.WriteAllText(... .anim)`

### C-2. 始点: GUIの単体AnimationClip出力
1. `AssetStudio.GUI/MainForm.cs` / `AssetStudio.GUI/Studio.cs` からエクスポート指示
2. `AssetStudio.GUI/Exporter.ExportAnimationClip`
3. 以降はCLIと同一（`Convert` -> `ProcessInner` -> YAML化 -> 書込）

### C-3. 始点: Bundle入力（最頻出）
1. `AssetsManager.LoadFiles` / `LoadFolder`
2. `LoadFile` の `FileType` 分岐
3. `FileType.BundleFile` で `LoadBundleFile`
4. `new BundleFile(reader, game)`
5. `ReadBundleHeader` -> `ReadHeader` -> `ReadBlocksInfoAndDirectory`
6. `ReadBlocks` で展開
7. `ReadFiles` で `StreamFile` 切り出し
8. `LoadAssetsFromMemory` で `SerializedFile` 化
9. `ReadAssets` で `ClassIDType.AnimationClip => new AnimationClip(objectReader)`
10. `Exporter.ExportAnimationClip` へ渡してB系処理へ接続

### C-4. 始点: Web/Zip/Block入力（派生ルート）
- `LoadWebFile`:
  - Web内部エントリを再帰読み込みし、AssetsFile/BundleFile/ResourceFileへ再分岐
- `LoadZipFile`:
  - zip entryをメモリ展開して `LoadFile` 再投入
- `LoadBlockFile` / `LoadBlkFile` / `LoadMhyFile`:
  - ブロック単位で内部ストリームを切り、Bundle/Mhy等へ再分岐
- どの派生ルートも最終的に `LoadAssetsFromMemory` と `ReadAssets` へ合流する

### C-5. Animation変換フェーズの関数責務（再監査）
- `ProcessInner`
  - 全体実行順序制御（ACL先行/後行分岐含む）
- `ProcessStreams`
  - StreamedFrameを読み、補間傾き(in/out slope)を計算してTransform/Default/Customへ振分
- `ProcessDenses`
  - Dense sampleを時系列展開して同様に振分
- `ProcessACLClip`
  - ACL圧縮値を展開して同様に振分
- `ProcessConstant`
  - 終端フレーム補完
- `CreateCurves`
  - 内部辞書を最終カーブ配列へコミット

### C-6. パス/属性解決の関数責務（再監査）
- `AnimationClipExtensions.FindTOS`
  - Avatar / Animator / Animation 起点でpath辞書を収集
- `BuildTOS`
  - Transform階層から `parent/child` を構築し、CRCハッシュ化
- `GetCurvePath`
  - `binding.path` ハッシュ -> 文字列path、見つからなければ `path_<hash>`
- `CustomCurveResolver.ToAttributeName`
  - `customType + attribute + path` から最終プロパティ名へ変換

### C-7. 出力終端（再監査）
1. `ConvertSerializedAnimationClip`
2. `ExportYAMLDocument`
3. `YAMLWriter.AddDocument`
4. `YAMLWriter.Write`
5. Exporterの `File.WriteAllText` で `.anim` 保存

### C-8. 再監査チェック（始点別）
- [x] CLI始点
- [x] GUI始点
- [x] Bundle始点
- [x] Web始点
- [x] Zip始点
- [x] Block/Blk/Mhy始点
- [x] Path解決始点
- [x] YAML出力終端
