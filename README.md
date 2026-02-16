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
ゼロベース再構築: 全コード始点→終点トレース（.anim出力）

この章は**継ぎ足し説明を破棄**して作り直した版です。
終点は固定で `File.WriteAllText(... .anim)` とします。

## 1) 終点の定義
- 終点: Exporter が `.anim` パスへ `File.WriteAllText` した時点。

## 2) 正規トレースID（終点到達チェーン）
- `CHAIN-L1`: Load入口 -> LoadFile分岐 -> コンテナ解析 -> ReadAssets -> ExportAnimationClip -> Convert -> YAML -> WriteAllText
- `CHAIN-E1`: ExportAnimationClip -> Convert -> ConvertSerializedAnimationClip -> YAMLWriter.Write -> File.WriteAllText
- `CHAIN-C1`: Convert -> Process -> ProcessInner -> (ACL/Stream/Dense/Constant) -> CreateCurves -> ConvertSerializedAnimationClip
- `CHAIN-P1`: FindTOS -> BuildTOS -> GetCurvePath -> AddCustomCurve -> ToAttributeName


## 3) 直接経路（関数単位）

### 3-1 読み込み側（終点へ向かう主鎖）
- `AssetsManager.LoadFiles / LoadFolder`
- `AssetsManager.Load`
- `AssetsManager.LoadFile(string/FileReader)`
- `AssetsManager.LoadBundleFile / LoadWebFile / LoadZipFile / LoadBlockFile / LoadBlkFile / LoadMhyFile`
- `AssetsManager.LoadAssetsFromMemory`
- `AssetsManager.ReadAssets`
- `ClassIDType.AnimationClip => new AnimationClip(objectReader)`

### 3-2 変換側（終点直前の主鎖）
- `Exporter.ExportAnimationClip` (CLI/GUI)
- `AnimationClipExtensions.Convert`
- `AnimationClipConverter.Process -> ProcessInner`
- `ProcessStreams / ProcessDenses / ProcessACLClip / ProcessConstant / CreateCurves`
- `ConvertSerializedAnimationClip -> ExportYAMLDocument -> YAMLWriter.Write`
- `File.WriteAllText(... .anim)`

### 3-3 パス解決鎖
- `FindTOS -> BuildTOS -> GetCurvePath -> AddCustomCurve -> CustomCurveResolver.ToAttributeName`

## 4) 全コードファイルの始点→終点関連表（.cs 全件）

凡例:
- 役割=そのファイルの責務
- 終点関連=`.anim`終点に対して直接/間接/非対象
- チェーンID=該当する正規トレースID

| ファイル | 役割 | 終点関連 | チェーンID |
|---|---|---|---|
| `AssetStudio.CLI/Components/AssetItem.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.CLI/Components/CommandLine.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.CLI/Exporter.cs` | 直接経路 | 到達(直接) | CHAIN-E1 |
| `AssetStudio.CLI/Program.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.CLI/Settings.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.CLI/Studio.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.FBXWrapper/Fbx.PInvoke.cs` | FBX/ネイティブ補助 | 非対象 | - |
| `AssetStudio.FBXWrapper/Fbx.cs` | FBX/ネイティブ補助 | 非対象 | - |
| `AssetStudio.FBXWrapper/FbxDll.cs` | FBX/ネイティブ補助 | 非対象 | - |
| `AssetStudio.FBXWrapper/FbxExporter.cs` | FBX/ネイティブ補助 | 非対象 | - |
| `AssetStudio.FBXWrapper/FbxExporterContext.PInvoke.cs` | FBX/ネイティブ補助 | 非対象 | - |
| `AssetStudio.FBXWrapper/FbxExporterContext.cs` | FBX/ネイティブ補助 | 非対象 | - |
| `AssetStudio.GUI/AssetBrowser.Designer.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/AssetBrowser.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/Components/AssetItem.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/Components/GOHierarchy.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/Components/GameObjectTreeNode.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/Components/OpenFolderDialog.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/Components/TypeTreeItem.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/DirectBitmap.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/ExportOptions.Designer.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/ExportOptions.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/Exporter.cs` | 直接経路 | 到達(直接) | CHAIN-E1 |
| `AssetStudio.GUI/GUILogger.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/MainForm.Designer.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/MainForm.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/Program.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/Properties/Resources.Designer.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/Properties/Settings.Designer.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/Studio.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/UnityCNForm.Designer.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.GUI/UnityCNForm.cs` | UI/操作層 | 到達(間接) | - |
| `AssetStudio.PInvoke/DllLoader.cs` | FBX/ネイティブ補助 | 非対象 | - |
| `AssetStudio.Utility/ACL/ACL.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/ACL/ACLExtensions.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/AssemblyLoader.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/AudioClipConverter.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/CSspv/Disassembler.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/CSspv/EnumValuesExtensions.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/CSspv/Instruction.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/CSspv/Module.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/CSspv/OperandType.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/CSspv/ParsedInstruction.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/CSspv/Reader.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/CSspv/SpirV.Core.Grammar.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/CSspv/SpirV.Meta.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/CSspv/Types.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/ConsoleHelper.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/FMOD Studio API/fmod.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/FMOD Studio API/fmod_dsp.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/FMOD Studio API/fmod_errors.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/FontHelper.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/ImageExtensions.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/ImageFormat.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/ModelConverter.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/ModelExporter.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/MonoBehaviourConverter.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/MyAssemblyResolver.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/SerializedTypeHelper.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/ShaderConverter.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Smolv/OpData.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Smolv/SmolvDecoder.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Smolv/SpvOp.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/SpirVShaderConverter.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/SpriteHelper.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Texture2DConverter.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Texture2DExtensions.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/TypeDefinitionConverter.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Unity.CecilTools/CecilUtils.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Unity.CecilTools/ElementType.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Unity.CecilTools/Extensions/MethodDefinitionExtensions.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Unity.CecilTools/Extensions/ResolutionExtensions.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Unity.CecilTools/Extensions/TypeDefinitionExtensions.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Unity.CecilTools/Extensions/TypeReferenceExtensions.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Unity.SerializationLogic/UnityEngineTypePredicates.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/Unity.SerializationLogic/UnitySerializationLogic.cs` | ユーティリティ補助 | 到達(間接) | - |
| `AssetStudio.Utility/YAML/AnimationClipConverter.cs` | 直接経路 | 到達(直接) | CHAIN-C1 |
| `AssetStudio.Utility/YAML/AnimationClipExtensions.cs` | 直接経路 | 到達(直接) | CHAIN-C1 |
| `AssetStudio.Utility/YAML/CustomCurveResolver.cs` | 直接経路 | 到達(直接) | CHAIN-P1 |
| `AssetStudio.Utility/YAML/MuscleHelper.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/7zip/Common/CRC.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Common/CommandLineParser.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Common/InBuffer.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Common/OutBuffer.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Compress/LZ/IMatchFinder.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Compress/LZ/LzBinTree.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Compress/LZ/LzInWindow.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Compress/LZ/LzOutWindow.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Compress/LZMA/LzmaBase.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Compress/LZMA/LzmaDecoder.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Compress/LZMA/LzmaEncoder.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Compress/RangeCoder/RangeCoder.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Compress/RangeCoder/RangeCoderBit.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/Compress/RangeCoder/RangeCoderBitTree.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/7zip/ICoder.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/AIVersionManager.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/AssetGroupOption.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/AssetIndex.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/AssetMap.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/AssetsHelper.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/AssetsManager.cs` | 直接経路 | 到達(直接) | CHAIN-L1 |
| `AssetStudio/BlbFile.cs` | 直接経路 | 到達(直接) | CHAIN-L1 |
| `AssetStudio/Brotli/BitReader.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/BrotliInputStream.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/BrotliRuntimeException.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/Context.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/Decode.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/Dictionary.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/Huffman.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/HuffmanTreeGroup.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/IntReader.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/Prefix.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/RunningState.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/State.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/Transform.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/Utils.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Brotli/WordTransformType.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/BuildTarget.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/BuildType.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/BundleFile.cs` | 直接経路 | 到達(直接) | CHAIN-L1 |
| `AssetStudio/ClassIDType.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Classes/Animation.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/AnimationClip.cs` | 直接経路 | 到達(直接) | CHAIN-L1 |
| `AssetStudio/Classes/Animator.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/AnimatorController.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/AnimatorOverrideController.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/AssetBundle.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/AudioClip.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Avatar.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Behaviour.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/BuildSettings.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Component.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/EditorExtension.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Font.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/GameObject.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/IndexObject.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Material.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Mesh.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/MeshFilter.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/MeshRenderer.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/MiHoYoBinData.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/MonoBehaviour.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/MonoScript.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/MovieTexture.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/NamedObject.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Object.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/PPtr.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/PlayerSettings.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/RectTransform.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Renderer.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/ResourceManager.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/RuntimeAnimatorController.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Shader.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/SkinnedMeshRenderer.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Sprite.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/SpriteAtlas.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/TextAsset.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Texture.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Texture2D.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/Transform.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/Classes/VideoClip.cs` | データ型/参照解決 | 到達(間接) | - |
| `AssetStudio/CommonString.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Crypto/AES.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Crypto/BlkUtils.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Crypto/CryptoHelper.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Crypto/FairGuardUtils.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Crypto/MT19937_64.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Crypto/Mr0kUtils.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Crypto/NetEaseUtils.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Crypto/OPFPUtils.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Crypto/UnityCN.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/Crypto/XORShift128.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/EndianBinaryReader.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/EndianType.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/ExportType.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/ExportTypeList.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Extensions/ByteArrayExtensions.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Extensions/StreamExtensions.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/FileIdentifier.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/FileReader.cs` | 直接経路 | 到達(直接) | CHAIN-L1 |
| `AssetStudio/FileType.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/GameManager.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Helpers/KVPConverter.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/IImported.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/ILogger.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/ImportHelper.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/LZ4/LZ4.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/LZ4/LZ4Inv.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/LZ4/LZ4Lit.cs` | 読込補助(復号/展開) | 到達(間接) | - |
| `AssetStudio/LocalSerializedObjectIdentifier.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Logger.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Math/Color.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Math/Float.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Math/Half.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Math/HalfHelper.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Math/Matrix4x4.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Math/Quaternion.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Math/Vector2.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Math/Vector3.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Math/Vector4.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Math/XForm.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/MhyFile.cs` | 直接経路 | 到達(直接) | CHAIN-L1 |
| `AssetStudio/ObjectInfo.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/ObjectReader.cs` | 直接経路 | 到達(直接) | CHAIN-L1 |
| `AssetStudio/OffsetStream.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/Progress.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/ResourceIndex.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/ResourceMap.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/ResourceReader.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/SerializedFile.cs` | 直接経路 | 到達(直接) | CHAIN-L1 |
| `AssetStudio/SerializedFileFormatVersion.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/SerializedFileHeader.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/SerializedType.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/SevenZipHelper.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/StreamFile.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/TypeFlags.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/TypeTree.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/TypeTreeHelper.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/TypeTreeNode.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/UnityCNManager.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/WebFile.cs` | 直接経路 | 到達(直接) | CHAIN-L1 |
| `AssetStudio/XORStream.cs` | コア補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/Emitter.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/IYAMLExportable.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/MappingStyle.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/MetaType.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/ScalarStyle.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/ScalarType.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/SequenceStyle.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/YAMLDocument.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/YAMLMappingNode.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/YAMLNode.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/YAMLNodeType.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/YAMLScalarNode.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/YAMLSequenceNode.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/YAMLTag.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Base/YAMLWriter.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Utils/Extensions/ArrayYAMLExtensions.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Utils/Extensions/BitConverterExtensions.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Utils/Extensions/EmitterExtensions.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Utils/Extensions/IDictionaryExportYAMLExtensions.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Utils/Extensions/IDictionaryYAMLExtensions.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Utils/Extensions/IEnumerableYAMLExtensions.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Utils/Extensions/IListYAMLExtensions.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Utils/Extensions/PrimitiveExtensions.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Utils/Extensions/StringBuilderExtensions.cs` | 変換補助 | 到達(間接) | - |
| `AssetStudio/YAML/Utils/Extensions/YAMLMappingNodeExtensions.cs` | 変換補助 | 到達(間接) | - |

## 5) 抜け確認
- [x] 全 `.cs` ファイルを一覧化
- [x] 直接経路の関数名を明示
- [x] パス解決関数鎖を明示
- [x] 終点を `File.WriteAllText(.anim)` に固定
