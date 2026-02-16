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
Bundle読込 → アニメファイル(`.anim`)出力フロー（詳細版）

> このREADMEでは「**もっと細かく全処理を提示する**」方針を選択。

```mermaid
flowchart TD
    A[ユーザー入力: file/folder] --> B[AssetsManager.LoadFiles or LoadFolder]
    B --> C[MergeSplitAssets / ProcessingSplitFiles]
    C --> D[Load import queue]
    D --> E[LoadFile(FileReader.PreProcessing(Game))]

    E --> F{FileType}
    F -->|AssetsFile| G[LoadAssetsFile]
    F -->|BundleFile| H[LoadBundleFile]
    F -->|WebFile| I[LoadWebFile]
    F -->|ZipFile| J[LoadZipFile]
    F -->|Block/Blk/Mhy| K[LoadBlockFile/LoadBlkFile/LoadMhyFile]

    H --> L[BundleFile.ctor]
    L --> M[ReadBundleHeader]
    M --> N[ReadHeader / ReadUnityCN / ReadBlocksInfoAndDirectory]
    N --> O[ReadBlocks: decompress LZMA/LZ4/Zstd/None]
    O --> P[ReadFiles: nodeごとにStreamFile作成]
    P --> Q[fileList列挙]
    Q --> R{entry type}
    R -->|AssetsFile| S[LoadAssetsFromMemory]
    R -->|ResourceFile| T[resourceFileReadersにキャッシュ]

    G --> U[assetsFileListへ登録 + externals解決]
    S --> U
    I --> U
    J --> U
    K --> U

    U --> V[ReadAssets]
    V --> W[ObjectReader生成]
    W --> X{ClassIDType switch}
    X -->|AnimationClip| Y[new AnimationClip(objectReader)]
    X -->|others| Z[new Texture2D/Mesh/etc]

    Y --> AA[ProcessAssets(参照解決・GOリンク)]
    AA --> AB[Studio: AssetItem化/フィルタ/グループ]
    AB --> AC[Exporter.ExportAnimationClip]
    AC --> AD[TryExportFile(.anim パス解決)]
    AD --> AE[AnimationClipExtensions.Convert]
    AE --> AF{legacy or muscle}
    AF --> AG[AnimationClipConverter.Process]
    AG --> AH[FindTOS / streamed+dense+constant+acl処理]
    AH --> AI[Rot/Euler/Pos/Scale/Float/PPtrカーブへ集約]
    AI --> AJ[ConvertSerializedAnimationClip]
    AJ --> AK[ExportYAMLDocument + YAMLWriter.Write]
    AK --> AL[File.WriteAllText(exportFullPath, yaml)]
```

処理の完全トレース（主要メソッド順）

1. **入力フェーズ**
   - `AssetsManager.LoadFiles` / `LoadFolder` が開始点。
   - 分割ファイルのマージ (`MergeSplitAssets`) と読込対象調整 (`ProcessingSplitFiles`) を実施。
   - `ResolveDependencies` が有効なら依存ファイル列挙を追加。

2. **ファイル判定フェーズ (`LoadFile`)**
   - `FileReader.PreProcessing(Game)` でゲーム別前処理。
   - `switch (reader.FileType)` で読み込みルートを分岐。
   - 本件の主ルートは `FileType.BundleFile -> LoadBundleFile`。

3. **Bundle解析フェーズ (`BundleFile`)**
   - `ReadBundleHeader`: signature/version/unityRevision を取得。
   - `ReadHeader`: flags, block info size を取得（ゲーム固有暗号や補正あり）。
   - 必要時 `ReadUnityCN`: UnityCN暗号を復号。
   - `ReadBlocksInfoAndDirectory`: ブロック情報+ノード情報を構築。
   - `ReadBlocks`: 圧縮種別ごとに展開（None/LZMA/LZ4/LZ4HC/Zstd 等）。
   - `ReadFiles`: ノード単位で `StreamFile` を作り、`fileList` へ展開。

4. **Assets化フェーズ (`LoadBundleFile`)**
   - `bundleFile.fileList` を走査。
   - 各 stream を `FileReader` 化。
   - `AssetsFile` なら `LoadAssetsFromMemory` へ、その他は `resourceFileReaders` へ保存。

5. **オブジェクト復元フェーズ (`ReadAssets`)**
   - すべての `SerializedFile.m_Objects` を巡回。
   - `ObjectReader` 生成後、`ClassIDType` の switch で具象クラス化。
   - `ClassIDType.AnimationClip` は `new AnimationClip(objectReader)` で復元。

6. **参照解決フェーズ (`ProcessAssets`)**
   - `GameObject` と各 Component の双方向参照を張る。
   - `SpriteAtlas` など補助参照も補完。
   - 以降 `Studio` 側で一覧 (`AssetItem`) 化され、エクスポートUI/CLI対象になる。

7. **AnimationClipエクスポート (`Exporter.ExportAnimationClip`)**
   - `TryExportFile` で重複回避した `*.anim` 出力パス確定。
   - `m_AnimationClip.Convert()` を呼ぶ。
   - 返却文字列が空でなければ `File.WriteAllText`。

8. **Convert内部（`.anim` YAML化）**
   - `AnimationClipExtensions.Convert` 開始。
   - `!m_Legacy || m_MuscleClip != null` の場合、`AnimationClipConverter.Process` 実行。
   - `Process` 内で:
     - `FindTOS`（Transformパス辞書）構築
     - streamed/dense/constant/acl 各カーブデータを抽出
     - `AddTransformCurve`, `AddCustomCurve`, `AddFloatKeyframe`, `AddPPtrKeyframe` でキー構築
   - 生成カーブを既存 `m_RotationCurves / m_EulerCurves / m_PositionCurves / m_ScaleCurves / m_FloatCurves / m_PPtrCurves` と統合。
   - `ConvertSerializedAnimationClip` -> `ExportYAMLDocument` -> `YAMLWriter.Write` でYAML文字列化。
   - 最終的にCLI/GUI Exporterが `.anim` として保存。

コード参照ポイント（追跡しやすい順）

- `AssetStudio/AssetsManager.cs`
  - `LoadFiles`, `LoadFolder`, `LoadFile`, `LoadBundleFile`, `LoadAssetsFromMemory`, `ReadAssets`, `ProcessAssets`
- `AssetStudio/BundleFile.cs`
  - `ReadBundleHeader`, `ReadHeader`, `ReadUnityCN`, `ReadBlocksInfoAndDirectory`, `ReadBlocks`, `ReadFiles`
- `AssetStudio.CLI/Exporter.cs` / `AssetStudio.GUI/Exporter.cs`
  - `ExportAnimationClip`
- `AssetStudio.Utility/YAML/AnimationClipExtensions.cs`
  - `Convert`, `ConvertSerializedAnimationClip`, `ExportYAMLDocument`, `ExportYAML`
- `AssetStudio.Utility/YAML/AnimationClipConverter.cs`
  - `Process` と各カーブ生成ロジック


9. **GUI/CLI 両ルートの分岐（省略なし）**
   - CLI実行: `AssetStudio.CLI/Program.cs` -> `Studio.cs` -> `Exporter.cs`。
   - GUI実行: `AssetStudio.GUI/Program.cs` -> `MainForm.cs` / `Studio.cs` -> `Exporter.cs`。
   - どちらも最終的に `AssetStudio` コア (`AssetsManager`, `BundleFile`, `SerializedFile`, `ObjectReader`, `Classes/*`) を使って同じ資産復元パイプラインを通ります。

Unity Editor取り込み用: 必要ソースコード一覧（GUI/CLI含む・省略なし）

> 下記は「Bundle読込→AnimationClip復元→`.anim`書き出し」処理をUnity Editor側へ移植する際に、追跡対象にすべきファイルを**カテゴリ別に全列挙**したもの。

### 1) エントリポイント層（CLI/GUI）
- `AssetStudio.CLI/Program.cs`
- `AssetStudio.CLI/Studio.cs`
- `AssetStudio.CLI/Exporter.cs`
- `AssetStudio.CLI/Settings.cs`
- `AssetStudio.GUI/Program.cs`
- `AssetStudio.GUI/MainForm.cs`
- `AssetStudio.GUI/Studio.cs`
- `AssetStudio.GUI/Exporter.cs`
- `AssetStudio.GUI/ExportOptions.cs`

### 2) 読み込みオーケストレーション層
- `AssetStudio/AssetsManager.cs`
- `AssetStudio/ImportHelper.cs`
- `AssetStudio/AssetsHelper.cs`
- `AssetStudio/AssetGroupOption.cs`
- `AssetStudio/ExportType.cs`
- `AssetStudio/ExportTypeList.cs`
- `AssetStudio/FileType.cs`

### 3) 入力フォーマット/コンテナ層（Bundle/Web/Zip/Block系）
- `AssetStudio/FileReader.cs`
- `AssetStudio/BundleFile.cs`
- `AssetStudio/WebFile.cs`
- `AssetStudio/BlbFile.cs`
- `AssetStudio/MhyFile.cs`
- `AssetStudio/StreamFile.cs`
- `AssetStudio/OffsetStream.cs`
- `AssetStudio/XORStream.cs`

### 4) 低レベル復号・圧縮解除
- `AssetStudio/SevenZipHelper.cs`
- `AssetStudio/LZ4/LZ4.cs`
- `AssetStudio/LZ4/LZ4Inv.cs`
- `AssetStudio/LZ4/LZ4Lit.cs`
- `AssetStudio/Brotli/*`
- `AssetStudio/Crypto/*`

### 5) SerializedFileデシリアライズ層
- `AssetStudio/SerializedFile.cs`
- `AssetStudio/SerializedFileHeader.cs`
- `AssetStudio/SerializedFileFormatVersion.cs`
- `AssetStudio/ObjectInfo.cs`
- `AssetStudio/ObjectReader.cs`
- `AssetStudio/ClassIDType.cs`
- `AssetStudio/TypeTree.cs`
- `AssetStudio/TypeTreeNode.cs`
- `AssetStudio/TypeTreeHelper.cs`
- `AssetStudio/SerializedType.cs`
- `AssetStudio/SerializedTypeHelper.cs`

### 6) 参照解決/アセットリンク層
- `AssetStudio/Classes/PPtr.cs`
- `AssetStudio/Classes/Object.cs`
- `AssetStudio/Classes/EditorExtension.cs`
- `AssetStudio/Classes/NamedObject.cs`
- `AssetStudio/Classes/Component.cs`
- `AssetStudio/Classes/GameObject.cs`
- `AssetStudio/Classes/Transform.cs`
- `AssetStudio/Classes/Animator.cs`
- `AssetStudio/Classes/Animation.cs`
- `AssetStudio/Classes/RuntimeAnimatorController.cs`
- `AssetStudio/Classes/AnimatorController.cs`
- `AssetStudio/Classes/AnimatorOverrideController.cs`

### 7) AnimationClip本体復元層
- `AssetStudio/Classes/AnimationClip.cs`
- `AssetStudio/Classes/Avatar.cs`
- `AssetStudio/Classes/RectTransform.cs`
- `AssetStudio/Math/*`

### 8) AnimationClip YAML変換層（`.anim`生成の中核）
- `AssetStudio.Utility/YAML/AnimationClipConverter.cs`
- `AssetStudio.Utility/YAML/AnimationClipExtensions.cs`
- `AssetStudio.Utility/YAML/CustomCurveResolver.cs`
- `AssetStudio.Utility/YAML/MuscleHelper.cs`
- `AssetStudio/YAML/Base/*`
- `AssetStudio/YAML/Utils/Extensions/*`

### 9) 出力層（最終ファイル書き込み）
- `AssetStudio.CLI/Exporter.cs` の `ExportAnimationClip`
- `AssetStudio.GUI/Exporter.cs` の `ExportAnimationClip`
- （モデル同時出力を使う場合）`AssetStudio.Utility/ModelConverter.cs`, `AssetStudio.Utility/ModelExporter.cs`

### 10) 補助（ログ・進捗・環境）
- `AssetStudio/Logger.cs`
- `AssetStudio/Progress.cs`
- `AssetStudio/ILogger.cs`
- `AssetStudio/GameManager.cs`
- `AssetStudio/AIVersionManager.cs`

Unity Editor移植時の実務メモ（省略なし）

1. まず `AssetsManager.LoadFiles/LoadFolder` と `BundleFile` をそのまま移植し、Bundleから `SerializedFile` へ到達する経路を固定。
2. 次に `ReadAssets` の `ClassIDType.AnimationClip` 復元が通ることを確認。
3. 続いて `AnimationClipExtensions.Convert` + `AnimationClipConverter.Process` を移植してYAML化を再現。
4. 最後に Unity Editor の `AssetDatabase` 書き込みフローへ接続（AssetStudio側は `File.WriteAllText` なので置換しやすい）。
5. GUI/CLI差分はUI層のみ。コア処理は `AssetStudio` / `AssetStudio.Utility` に集中しているため、移植対象の本丸はこの2プロジェクト。


完全ファイル一覧（自動生成・省略なし）

以下は移植対象検討用に `AssetStudio`, `AssetStudio.Utility`, `AssetStudio.CLI`, `AssetStudio.GUI` 配下の `.cs` を列挙した一覧です。

- `AssetStudio/7zip/Common/CRC.cs`
- `AssetStudio/7zip/Common/CommandLineParser.cs`
- `AssetStudio/7zip/Common/InBuffer.cs`
- `AssetStudio/7zip/Common/OutBuffer.cs`
- `AssetStudio/7zip/Compress/LZ/IMatchFinder.cs`
- `AssetStudio/7zip/Compress/LZ/LzBinTree.cs`
- `AssetStudio/7zip/Compress/LZ/LzInWindow.cs`
- `AssetStudio/7zip/Compress/LZ/LzOutWindow.cs`
- `AssetStudio/7zip/Compress/LZMA/LzmaBase.cs`
- `AssetStudio/7zip/Compress/LZMA/LzmaDecoder.cs`
- `AssetStudio/7zip/Compress/LZMA/LzmaEncoder.cs`
- `AssetStudio/7zip/Compress/RangeCoder/RangeCoder.cs`
- `AssetStudio/7zip/Compress/RangeCoder/RangeCoderBit.cs`
- `AssetStudio/7zip/Compress/RangeCoder/RangeCoderBitTree.cs`
- `AssetStudio/7zip/ICoder.cs`
- `AssetStudio/AIVersionManager.cs`
- `AssetStudio/AssetGroupOption.cs`
- `AssetStudio/AssetIndex.cs`
- `AssetStudio/AssetMap.cs`
- `AssetStudio/AssetsHelper.cs`
- `AssetStudio/AssetsManager.cs`
- `AssetStudio/BlbFile.cs`
- `AssetStudio/Brotli/BitReader.cs`
- `AssetStudio/Brotli/BrotliInputStream.cs`
- `AssetStudio/Brotli/BrotliRuntimeException.cs`
- `AssetStudio/Brotli/Context.cs`
- `AssetStudio/Brotli/Decode.cs`
- `AssetStudio/Brotli/Dictionary.cs`
- `AssetStudio/Brotli/Huffman.cs`
- `AssetStudio/Brotli/HuffmanTreeGroup.cs`
- `AssetStudio/Brotli/IntReader.cs`
- `AssetStudio/Brotli/Prefix.cs`
- `AssetStudio/Brotli/RunningState.cs`
- `AssetStudio/Brotli/State.cs`
- `AssetStudio/Brotli/Transform.cs`
- `AssetStudio/Brotli/Utils.cs`
- `AssetStudio/Brotli/WordTransformType.cs`
- `AssetStudio/BuildTarget.cs`
- `AssetStudio/BuildType.cs`
- `AssetStudio/BundleFile.cs`
- `AssetStudio/ClassIDType.cs`
- `AssetStudio/Classes/Animation.cs`
- `AssetStudio/Classes/AnimationClip.cs`
- `AssetStudio/Classes/Animator.cs`
- `AssetStudio/Classes/AnimatorController.cs`
- `AssetStudio/Classes/AnimatorOverrideController.cs`
- `AssetStudio/Classes/AssetBundle.cs`
- `AssetStudio/Classes/AudioClip.cs`
- `AssetStudio/Classes/Avatar.cs`
- `AssetStudio/Classes/Behaviour.cs`
- `AssetStudio/Classes/BuildSettings.cs`
- `AssetStudio/Classes/Component.cs`
- `AssetStudio/Classes/EditorExtension.cs`
- `AssetStudio/Classes/Font.cs`
- `AssetStudio/Classes/GameObject.cs`
- `AssetStudio/Classes/IndexObject.cs`
- `AssetStudio/Classes/Material.cs`
- `AssetStudio/Classes/Mesh.cs`
- `AssetStudio/Classes/MeshFilter.cs`
- `AssetStudio/Classes/MeshRenderer.cs`
- `AssetStudio/Classes/MiHoYoBinData.cs`
- `AssetStudio/Classes/MonoBehaviour.cs`
- `AssetStudio/Classes/MonoScript.cs`
- `AssetStudio/Classes/MovieTexture.cs`
- `AssetStudio/Classes/NamedObject.cs`
- `AssetStudio/Classes/Object.cs`
- `AssetStudio/Classes/PPtr.cs`
- `AssetStudio/Classes/PlayerSettings.cs`
- `AssetStudio/Classes/RectTransform.cs`
- `AssetStudio/Classes/Renderer.cs`
- `AssetStudio/Classes/ResourceManager.cs`
- `AssetStudio/Classes/RuntimeAnimatorController.cs`
- `AssetStudio/Classes/Shader.cs`
- `AssetStudio/Classes/SkinnedMeshRenderer.cs`
- `AssetStudio/Classes/Sprite.cs`
- `AssetStudio/Classes/SpriteAtlas.cs`
- `AssetStudio/Classes/TextAsset.cs`
- `AssetStudio/Classes/Texture.cs`
- `AssetStudio/Classes/Texture2D.cs`
- `AssetStudio/Classes/Transform.cs`
- `AssetStudio/Classes/VideoClip.cs`
- `AssetStudio/CommonString.cs`
- `AssetStudio/Crypto/AES.cs`
- `AssetStudio/Crypto/BlkUtils.cs`
- `AssetStudio/Crypto/CryptoHelper.cs`
- `AssetStudio/Crypto/FairGuardUtils.cs`
- `AssetStudio/Crypto/MT19937_64.cs`
- `AssetStudio/Crypto/Mr0kUtils.cs`
- `AssetStudio/Crypto/NetEaseUtils.cs`
- `AssetStudio/Crypto/OPFPUtils.cs`
- `AssetStudio/Crypto/UnityCN.cs`
- `AssetStudio/Crypto/XORShift128.cs`
- `AssetStudio/EndianBinaryReader.cs`
- `AssetStudio/EndianType.cs`
- `AssetStudio/ExportType.cs`
- `AssetStudio/ExportTypeList.cs`
- `AssetStudio/Extensions/ByteArrayExtensions.cs`
- `AssetStudio/Extensions/StreamExtensions.cs`
- `AssetStudio/FileIdentifier.cs`
- `AssetStudio/FileReader.cs`
- `AssetStudio/FileType.cs`
- `AssetStudio/GameManager.cs`
- `AssetStudio/Helpers/KVPConverter.cs`
- `AssetStudio/IImported.cs`
- `AssetStudio/ILogger.cs`
- `AssetStudio/ImportHelper.cs`
- `AssetStudio/LZ4/LZ4.cs`
- `AssetStudio/LZ4/LZ4Inv.cs`
- `AssetStudio/LZ4/LZ4Lit.cs`
- `AssetStudio/LocalSerializedObjectIdentifier.cs`
- `AssetStudio/Logger.cs`
- `AssetStudio/Math/Color.cs`
- `AssetStudio/Math/Float.cs`
- `AssetStudio/Math/Half.cs`
- `AssetStudio/Math/HalfHelper.cs`
- `AssetStudio/Math/Matrix4x4.cs`
- `AssetStudio/Math/Quaternion.cs`
- `AssetStudio/Math/Vector2.cs`
- `AssetStudio/Math/Vector3.cs`
- `AssetStudio/Math/Vector4.cs`
- `AssetStudio/Math/XForm.cs`
- `AssetStudio/MhyFile.cs`
- `AssetStudio/ObjectInfo.cs`
- `AssetStudio/ObjectReader.cs`
- `AssetStudio/OffsetStream.cs`
- `AssetStudio/Progress.cs`
- `AssetStudio/ResourceIndex.cs`
- `AssetStudio/ResourceMap.cs`
- `AssetStudio/ResourceReader.cs`
- `AssetStudio/SerializedFile.cs`
- `AssetStudio/SerializedFileFormatVersion.cs`
- `AssetStudio/SerializedFileHeader.cs`
- `AssetStudio/SerializedType.cs`
- `AssetStudio/SevenZipHelper.cs`
- `AssetStudio/StreamFile.cs`
- `AssetStudio/TypeFlags.cs`
- `AssetStudio/TypeTree.cs`
- `AssetStudio/TypeTreeHelper.cs`
- `AssetStudio/TypeTreeNode.cs`
- `AssetStudio/UnityCNManager.cs`
- `AssetStudio/WebFile.cs`
- `AssetStudio/XORStream.cs`
- `AssetStudio/YAML/Base/Emitter.cs`
- `AssetStudio/YAML/Base/IYAMLExportable.cs`
- `AssetStudio/YAML/Base/MappingStyle.cs`
- `AssetStudio/YAML/Base/MetaType.cs`
- `AssetStudio/YAML/Base/ScalarStyle.cs`
- `AssetStudio/YAML/Base/ScalarType.cs`
- `AssetStudio/YAML/Base/SequenceStyle.cs`
- `AssetStudio/YAML/Base/YAMLDocument.cs`
- `AssetStudio/YAML/Base/YAMLMappingNode.cs`
- `AssetStudio/YAML/Base/YAMLNode.cs`
- `AssetStudio/YAML/Base/YAMLNodeType.cs`
- `AssetStudio/YAML/Base/YAMLScalarNode.cs`
- `AssetStudio/YAML/Base/YAMLSequenceNode.cs`
- `AssetStudio/YAML/Base/YAMLTag.cs`
- `AssetStudio/YAML/Base/YAMLWriter.cs`
- `AssetStudio/YAML/Utils/Extensions/ArrayYAMLExtensions.cs`
- `AssetStudio/YAML/Utils/Extensions/BitConverterExtensions.cs`
- `AssetStudio/YAML/Utils/Extensions/EmitterExtensions.cs`
- `AssetStudio/YAML/Utils/Extensions/IDictionaryExportYAMLExtensions.cs`
- `AssetStudio/YAML/Utils/Extensions/IDictionaryYAMLExtensions.cs`
- `AssetStudio/YAML/Utils/Extensions/IEnumerableYAMLExtensions.cs`
- `AssetStudio/YAML/Utils/Extensions/IListYAMLExtensions.cs`
- `AssetStudio/YAML/Utils/Extensions/PrimitiveExtensions.cs`
- `AssetStudio/YAML/Utils/Extensions/StringBuilderExtensions.cs`
- `AssetStudio/YAML/Utils/Extensions/YAMLMappingNodeExtensions.cs`
- `AssetStudio.Utility/ACL/ACL.cs`
- `AssetStudio.Utility/ACL/ACLExtensions.cs`
- `AssetStudio.Utility/AssemblyLoader.cs`
- `AssetStudio.Utility/AudioClipConverter.cs`
- `AssetStudio.Utility/CSspv/Disassembler.cs`
- `AssetStudio.Utility/CSspv/EnumValuesExtensions.cs`
- `AssetStudio.Utility/CSspv/Instruction.cs`
- `AssetStudio.Utility/CSspv/Module.cs`
- `AssetStudio.Utility/CSspv/OperandType.cs`
- `AssetStudio.Utility/CSspv/ParsedInstruction.cs`
- `AssetStudio.Utility/CSspv/Reader.cs`
- `AssetStudio.Utility/CSspv/SpirV.Core.Grammar.cs`
- `AssetStudio.Utility/CSspv/SpirV.Meta.cs`
- `AssetStudio.Utility/CSspv/Types.cs`
- `AssetStudio.Utility/ConsoleHelper.cs`
- `AssetStudio.Utility/FMOD Studio API/fmod.cs`
- `AssetStudio.Utility/FMOD Studio API/fmod_dsp.cs`
- `AssetStudio.Utility/FMOD Studio API/fmod_errors.cs`
- `AssetStudio.Utility/FontHelper.cs`
- `AssetStudio.Utility/ImageExtensions.cs`
- `AssetStudio.Utility/ImageFormat.cs`
- `AssetStudio.Utility/ModelConverter.cs`
- `AssetStudio.Utility/ModelExporter.cs`
- `AssetStudio.Utility/MonoBehaviourConverter.cs`
- `AssetStudio.Utility/MyAssemblyResolver.cs`
- `AssetStudio.Utility/SerializedTypeHelper.cs`
- `AssetStudio.Utility/ShaderConverter.cs`
- `AssetStudio.Utility/Smolv/OpData.cs`
- `AssetStudio.Utility/Smolv/SmolvDecoder.cs`
- `AssetStudio.Utility/Smolv/SpvOp.cs`
- `AssetStudio.Utility/SpirVShaderConverter.cs`
- `AssetStudio.Utility/SpriteHelper.cs`
- `AssetStudio.Utility/Texture2DConverter.cs`
- `AssetStudio.Utility/Texture2DExtensions.cs`
- `AssetStudio.Utility/TypeDefinitionConverter.cs`
- `AssetStudio.Utility/Unity.CecilTools/CecilUtils.cs`
- `AssetStudio.Utility/Unity.CecilTools/ElementType.cs`
- `AssetStudio.Utility/Unity.CecilTools/Extensions/MethodDefinitionExtensions.cs`
- `AssetStudio.Utility/Unity.CecilTools/Extensions/ResolutionExtensions.cs`
- `AssetStudio.Utility/Unity.CecilTools/Extensions/TypeDefinitionExtensions.cs`
- `AssetStudio.Utility/Unity.CecilTools/Extensions/TypeReferenceExtensions.cs`
- `AssetStudio.Utility/Unity.SerializationLogic/UnityEngineTypePredicates.cs`
- `AssetStudio.Utility/Unity.SerializationLogic/UnitySerializationLogic.cs`
- `AssetStudio.Utility/YAML/AnimationClipConverter.cs`
- `AssetStudio.Utility/YAML/AnimationClipExtensions.cs`
- `AssetStudio.Utility/YAML/CustomCurveResolver.cs`
- `AssetStudio.Utility/YAML/MuscleHelper.cs`
- `AssetStudio.CLI/Components/AssetItem.cs`
- `AssetStudio.CLI/Components/CommandLine.cs`
- `AssetStudio.CLI/Exporter.cs`
- `AssetStudio.CLI/Program.cs`
- `AssetStudio.CLI/Settings.cs`
- `AssetStudio.CLI/Studio.cs`
- `AssetStudio.GUI/AssetBrowser.Designer.cs`
- `AssetStudio.GUI/AssetBrowser.cs`
- `AssetStudio.GUI/Components/AssetItem.cs`
- `AssetStudio.GUI/Components/GOHierarchy.cs`
- `AssetStudio.GUI/Components/GameObjectTreeNode.cs`
- `AssetStudio.GUI/Components/OpenFolderDialog.cs`
- `AssetStudio.GUI/Components/TypeTreeItem.cs`
- `AssetStudio.GUI/DirectBitmap.cs`
- `AssetStudio.GUI/ExportOptions.Designer.cs`
- `AssetStudio.GUI/ExportOptions.cs`
- `AssetStudio.GUI/Exporter.cs`
- `AssetStudio.GUI/GUILogger.cs`
- `AssetStudio.GUI/MainForm.Designer.cs`
- `AssetStudio.GUI/MainForm.cs`
- `AssetStudio.GUI/Program.cs`
- `AssetStudio.GUI/Properties/Resources.Designer.cs`
- `AssetStudio.GUI/Properties/Settings.Designer.cs`
- `AssetStudio.GUI/Studio.cs`
- `AssetStudio.GUI/UnityCNForm.Designer.cs`
- `AssetStudio.GUI/UnityCNForm.cs`



以下は「混ぜずに分ける」ための固定ルール。

## A. バンドル読み取り処理（Input/Decode/Deserialize）

### 役割
- ファイル入力、暗号解除、圧縮展開、`SerializedFile` 読み込み、`AnimationClip` オブジェクト生成まで。
- **この段階では `.anim` を作らない。**

### 主担当ファイル
- `AssetStudio/AssetsManager.cs`
- `AssetStudio/FileReader.cs`
- `AssetStudio/BundleFile.cs`
- `AssetStudio/WebFile.cs`
- `AssetStudio/MhyFile.cs`
- `AssetStudio/BlbFile.cs`
- `AssetStudio/SerializedFile.cs`
- `AssetStudio/ObjectReader.cs`
- `AssetStudio/Classes/AnimationClip.cs`
- `AssetStudio/Crypto/*`
- `AssetStudio/LZ4/*`
- `AssetStudio/Brotli/*`
- `AssetStudio/7zip/*`

### 出力（Aの成果物）
- `AssetsManager.assetsFileList`
- `SerializedFile.Objects` 内の `AssetStudio.AnimationClip` インスタンス

---

## B. アニメ変換処理（Convert/Serialize/Write）

### 役割
- `AnimationClip` をカーブ再構築し、YAML (`.anim`) 文字列化し、ファイルへ保存。
- **この段階ではバンドルを読まない。**

### 主担当ファイル
- `AssetStudio.Utility/YAML/AnimationClipConverter.cs`
- `AssetStudio.Utility/YAML/AnimationClipExtensions.cs`
- `AssetStudio.Utility/YAML/CustomCurveResolver.cs`
- `AssetStudio.Utility/YAML/MuscleHelper.cs`
- `AssetStudio/YAML/Base/*`
- `AssetStudio/YAML/Utils/Extensions/*`
- `AssetStudio.CLI/Exporter.cs` (`ExportAnimationClip`)
- `AssetStudio.GUI/Exporter.cs` (`ExportAnimationClip`)

### SAM原文（生データ）→ `.anim` 変換の実体
- ここでいう「SAM原文」は、`AnimationClip` 内の生カーブデータ群（`m_StreamedClip` / `m_DenseClip` / `m_ACLClip` / `m_ConstantClip`）を指す。
- 変換は `AnimationClipExtensions.Convert` から開始し、`AnimationClipConverter.Process` が中心処理。

#### 変換順序（実コード順）
1. `m_MuscleClip.m_Clip` と `m_ClipBindingConstant`、`FindTOS()` の結果を取得。
2. `m_StreamedClip.ReadData()` でストリームキー列をフレーム展開。
3. `ProcessACLClip`（条件付き）でACL圧縮データを通常キー列へ復元。
4. `ProcessStreams` で streamed キーをバインディング単位に割当。
5. `ProcessDenses` で dense サンプル配列を時系列キーへ変換。
6. `ProcessConstant` で constant 値を終端フレームへ補完。
7. 生成キーを `Rot/Euler/Pos/Scale/Float/PPtr` の各カーブへ集約。
8. 既存カーブと `Union` して `AnimationClip` へ反映。
9. `ConvertSerializedAnimationClip` → `ExportYAMLDocument` → `YAMLWriter.Write` で `.anim` YAML文字列化。

#### パス処理（不足していた部分）
1. `FindTOS()` で `Dictionary<uint, string>` を作る。
   - ルートは `0 -> ""`。
   - 子階層は `parent/child` 形式で連結し、`CRC.CalculateDigestUTF8(path)` をキーに登録。
2. 各カーブ処理 (`ProcessStreams / ProcessDenses / ProcessACLClip / ProcessConstant`) で、`binding.path` を `GetCurvePath(tos, binding.path)` に通す。
3. `tos` にハッシュが無い場合、`GetCurvePath` は `path_<hash>` を返す（UnknownPath扱い）。
4. `AddCustomCurve` では `CustomCurveResolver.ToAttributeName(type, attribute, path)` を呼び、`path` と `attribute` から最終プロパティ名を解決。
   - 例: `RendererMaterial`, `BlendShape`, `MonoBehaviour` などは `path` 先の実オブジェクトを辿って属性名を逆引き。
5. UnknownPath (`path_<hash>`) の場合は解決不能前提でフォールバック名（`material.<id>` 等）を使い、YAML出力自体は継続する。

#### どの値がどう変わるか（要点）
- 入力: 圧縮/分割されたキー列（streamed/dense/acl/constant）+ binding情報 + TOS(path辞書)
- 中間: `AddTransformCurve` / `AddCustomCurve` / `AddFloatKeyframe` / `AddPPtrKeyframe` で型付きKeyframe化
- 出力: Unityが読める `.anim` YAML（`m_RotationCurves` などが埋まった状態）

### 出力（Bの成果物）
- `.anim` ファイル（YAML）

---

## A/B 接続点（唯一の橋渡し）

- 接続データは `AssetStudio.AnimationClip` オブジェクトのみ。
- 接続メソッドは実質 `m_AnimationClip.Convert()`（`AnimationClipExtensions.Convert`）のみ。
- つまり移植時は以下の順に実装すればよい。
  1. Aを完成（Bundle -> `AnimationClip` 取得）
  2. Bを完成（`AnimationClip` -> `.anim` 出力）
  3. A/B間を `foreach (obj is AnimationClip)` で接続
