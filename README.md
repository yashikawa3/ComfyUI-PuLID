# ComfyUI-PuLID

PuLID v1.0/v1.1 両対応の ComfyUI カスタムノードです。
重みファイルのキー構造から v1.0/v1.1 を**自動判別**するため、ユーザーはバージョンを意識する必要がありません。

ベースとなったプロジェクト:
- [cubiq/PuLID_ComfyUI](https://github.com/cubiq/PuLID_ComfyUI) (v1.0対応、メンテナンスモード)
- [ToTheBeginning/PuLID](https://github.com/ToTheBeginning/PuLID) (公式リポジトリ)

## インストール

### 1. ノードのインストール

```bash
cd ComfyUI/custom_nodes/
git clone https://github.com/yashikawa3/ComfyUI-PuLID.git
cd ComfyUI-PuLID
pip install -r requirements.txt
```

### 2. モデルファイルの配置

#### PuLID 重みファイル

`ComfyUI/models/pulid/` に配置してください。

| バージョン | ファイル名 | 入手先 |
|-----------|-----------|--------|
| v1.0 | `ip-adapter_pulid_sdxl_fp16.safetensors` | [HuggingFace](https://huggingface.co/huchenlei/ipadapter_pulid/tree/main) |
| v1.1 | `pulid_v1.1.safetensors` | [HuggingFace](https://huggingface.co/guozinan/PuLID/tree/main) |

#### InsightFace (antelopev2)

`ComfyUI/models/insightface/models/antelopev2/` に配置してください。

[HuggingFace - antelopev2](https://huggingface.co/DIAMONIK7777/antelopev2/tree/main) から以下のファイルをダウンロード:
- `1k3d68.onnx`
- `2d106det.onnx`
- `genderage.onnx`
- `glintr100.onnx`
- `scrfd_10g_bnkps.onnx`

#### EVA-CLIP

初回実行時に自動ダウンロードされます。手動配置は不要です。

## ノード一覧

### Load PuLID Model (`PulidModelLoader`)

PuLIDの重みファイルをロードします。

| 入力 | 説明 |
|------|------|
| `pulid_file` | `ComfyUI/models/pulid/` 内の重みファイルを選択 |

| 出力 | 型 |
|------|-----|
| PULID | PuLIDモデル (v1.0/v1.1自動判別済み) |

ロード時にコンソールに `PuLID model loaded: v1.0` または `v1.1` と表示されます。

### Load InsightFace (`PulidInsightFaceLoader`)

顔検出・顔特徴抽出用の InsightFace (antelopev2) をロードします。

| 入力 | 説明 |
|------|------|
| `provider` | 推論デバイス: `CPU`, `CUDA`, `ROCM`, `CoreML` |

| 出力 | 型 |
|------|-----|
| FACEANALYSIS | InsightFaceモデル |

### Load Eva Clip (`PulidEvaClipLoader`)

EVA02-CLIP-L-14-336 をロードします。入力パラメータはありません。

| 出力 | 型 |
|------|-----|
| EVA_CLIP | EVA-CLIPモデル |

### Apply PuLID (`ApplyPulid`)

PuLIDをモデルに適用します。シンプルな設定で使いたい場合はこちらを使用してください。

| 入力 | 型 | デフォルト | 説明 |
|------|----|-----------|------|
| `model` | MODEL | - | ベースモデル (SDXL) |
| `pulid` | PULID | - | Load PuLID Model の出力 |
| `eva_clip` | EVA_CLIP | - | Load Eva Clip の出力 |
| `face_analysis` | FACEANALYSIS | - | Load InsightFace の出力 |
| `image` | IMAGE | - | 参照顔画像 (複数可) |
| `method` | enum | - | `fidelity`: ID忠実度重視 / `style`: スタイル重視 / `neutral`: ニュートラル |
| `weight` | FLOAT | 1.0 | PuLIDの適用強度 (-1.0 ~ 5.0) |
| `start_at` | FLOAT | 0.0 | 適用開始タイミング (0.0 ~ 1.0) |
| `end_at` | FLOAT | 1.0 | 適用終了タイミング (0.0 ~ 1.0) |
| `attn_mask` | MASK | (任意) | アテンションマスク |

| 出力 | 型 |
|------|-----|
| MODEL | PuLID適用済みモデル |

**method による動作の違い:**

| method | ortho | num_zero (v1.0) | num_zero (v1.1) |
|--------|-------|-----------------|-----------------|
| `fidelity` | ortho_v2 | 8 | 20 |
| `style` | ortho | 16 | 16 |
| `neutral` | なし | 0 | 0 |

### Apply PuLID Advanced (`ApplyPulidAdvanced`)

全パラメータを個別に制御できる上級者向けノードです。

| 入力 | 型 | デフォルト | 説明 |
|------|----|-----------|------|
| `model` | MODEL | - | ベースモデル (SDXL) |
| `pulid` | PULID | - | Load PuLID Model の出力 |
| `eva_clip` | EVA_CLIP | - | Load Eva Clip の出力 |
| `face_analysis` | FACEANALYSIS | - | Load InsightFace の出力 |
| `image` | IMAGE | - | 参照顔画像 (複数可) |
| `weight` | FLOAT | 1.0 | PuLIDの適用強度 (-1.0 ~ 5.0) |
| `projection` | enum | ortho_v2 | 射影方式: `ortho_v2` / `ortho` / `none` |
| `fidelity` | INT | 20 | ゼロパディング数 (0 ~ 32)。大きいほどID忠実度が上がる |
| `noise` | FLOAT | 0.0 | uncond埋め込みへのノイズ量 (-1.0 ~ 1.0) |
| `start_at` | FLOAT | 0.0 | 適用開始タイミング |
| `end_at` | FLOAT | 1.0 | 適用終了タイミング |
| `attn_mask` | MASK | (任意) | アテンションマスク |

## 基本ワークフロー

```
[Load Checkpoint (SDXL)] ──model──┐
[Load PuLID Model] ──────pulid───┤
[Load Eva Clip] ─────eva_clip────┤
[Load InsightFace] ─face_analysis┤
[Load Image] ──────────image─────┤
                                 ▼
                          [Apply PuLID]
                                 │
                               model
                                 ▼
                        [KSampler] → [VAE Decode] → [Save Image]
```

## v1.0 と v1.1 の違い

| 項目 | v1.0 | v1.1 |
|------|------|------|
| IDエンコーダ | IDEncoder (MLP, 10トークン) | IDFormer (Perceiver Resampler, 32トークン) |
| 複数参照画像 | 埋め込みの平均化 | IDFormerでネイティブ対応 (より高品質) |
| 推奨 fidelity | 8 | 20 |
| 推奨 projection | ortho_v2 | ortho_v2 |
| 対応ベースモデル | SDXL-Lightning | Juggernaut-XL, DreamShaper-XL 等に拡大 |

## 複数参照画像の使い方

`image` 入力に複数の顔画像をバッチとして渡すことで、複数の参照画像を使用できます。
ComfyUI の `Batch Images` ノードなどで画像を結合してください。

- **v1.1**: IDFormer が複数の参照画像をネイティブに処理します (メイン1枚 + 補助最大3枚)。各画像の特徴が個別に保持されるため、平均化よりも高品質な結果が得られます。
- **v1.0**: 各画像の埋め込みが平均化されます。

## トラブルシューティング

### 顔が検出されない

- 入力画像に正面向きの顔が含まれていることを確認してください
- 画像の解像度が低すぎる場合、検出に失敗することがあります
- コンソールに `Warning: No face detected in image X` と表示されます

### InsightFace のロードに失敗する

- `onnxruntime` (GPU環境では `onnxruntime-gpu`) がインストールされているか確認してください
- antelopev2 のモデルファイルが `ComfyUI/models/insightface/models/antelopev2/` に正しく配置されているか確認してください

### CUDA out of memory

- `weight` を下げるか、`start_at` / `end_at` の範囲を狭めてみてください
- InsightFace の provider を `CPU` に変更してVRAM使用量を減らせます

## ライセンス

本プロジェクトは [cubiq/PuLID_ComfyUI](https://github.com/cubiq/PuLID_ComfyUI) および [ToTheBeginning/PuLID](https://github.com/ToTheBeginning/PuLID) のコードを含みます。
各プロジェクトのライセンスに従ってください。
