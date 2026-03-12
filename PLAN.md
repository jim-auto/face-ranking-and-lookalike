# Face Ranking & Lookalike - 実装計画

## 概要
芸能人・インフルエンサーの顔スコアランキングサイトを GitHub Pages で公開。
ユーザーが自分の顔写真をアップロードすると、スコア算出 + 似ている芸能人を表示する。

---

## アーキテクチャ

```
[オフライン前処理 (Python)]          [フロントエンド (GitHub Pages)]
  ├─ 画像収集                         ├─ ランキング表示
  ├─ 顔検出 & ランドマーク抽出         ├─ 顔アップロード → スコア算出
  ├─ スコア算出                       ├─ 似ている芸能人表示
  ├─ 顔特徴ベクトル (embedding) 抽出   └─ face-api.js (クライアント側推論)
  └─ JSON データ出力
```

### なぜこの構成か
- GitHub Pages = 静的サイトのみ → **バックエンドなし**
- 顔の推論は **face-api.js (TensorFlow.js ベース)** でブラウザ上実行
- 芸能人データは **オフラインで前処理** → JSON + サムネイル画像として配置

---

## Phase 1: オフライン前処理パイプライン (Python)

### 1-1. 画像収集
- 対象: 芸能人・インフルエンサー 50〜100人程度（初期）
- ソース: フリー素材 or 手動収集（著作権に注意）
- 1人あたり正面顔画像 1〜3枚

### 1-2. 顔スコア算出ロジック
**黄金比ベースの顔面偏差値アルゴリズム:**

| 指標 | 内容 | 重み |
|------|------|------|
| 顔の対称性 | 左右ランドマークの鏡像距離 | 20% |
| 黄金比適合度 | 顔幅:顔長、目間距離:顔幅 等 | 25% |
| 目の大きさ・形状 | 目の縦横比、白目と黒目のバランス | 15% |
| 鼻の比率 | 鼻幅:顔幅、鼻長:顔長 | 10% |
| 口の比率 | 口幅:鼻幅、上唇:下唇 | 10% |
| 輪郭の滑らかさ | 顎ラインのフィッティング誤差 | 10% |
| 肌の均一性 | 色ムラ・テクスチャ均一度 | 10% |

- スコアは **0〜100点** に正規化
- 68点ランドマーク (dlib) を使用

### 1-3. 顔特徴ベクトル抽出
- **dlib** or **face_recognition** ライブラリで 128次元 embedding を抽出
- face-api.js 互換の embedding も並行抽出（ブラウザ側マッチング用）

### 1-4. 出力データ
```
data/
  celebrities.json     # 名前, スコア, embedding, サムネパス
  thumbnails/           # 顔サムネイル画像 (200x200)
```

**celebrities.json 構造:**
```json
[
  {
    "id": "celeb_001",
    "name": "山田太郎",
    "category": "actor",
    "score": 85.3,
    "details": {
      "symmetry": 88,
      "golden_ratio": 82,
      "eyes": 90,
      "nose": 80,
      "mouth": 85,
      "contour": 83,
      "skin": 84
    },
    "embedding": [0.12, -0.34, ...],
    "thumbnail": "thumbnails/celeb_001.jpg"
  }
]
```

---

## Phase 2: フロントエンド (GitHub Pages)

### 2-1. 技術スタック
- **Vite + React + TypeScript**
- **face-api.js** — ブラウザ上で顔検出・ランドマーク・embedding 抽出
- **Tailwind CSS** — スタイリング
- デプロイ: `gh-pages` ブランチ or GitHub Actions

### 2-2. ページ構成

#### ランキングページ (`/`)
- 芸能人スコアランキング（カード形式）
- フィルタ: カテゴリ別 (俳優/女優/アイドル/インフルエンサー)
- ソート: スコア順 / 名前順
- 各カードにスコアレーダーチャート表示

#### 診断ページ (`/diagnose`)
- 顔写真アップロード (ドラッグ&ドロップ対応)
- **すべてブラウザ内処理 (画像はサーバーに送信しない)**
- 処理フロー:
  1. face-api.js で顔検出 & ランドマーク取得
  2. ランドマークからスコア算出 (同じアルゴリズム JS 移植版)
  3. embedding 抽出 → celebrities.json の embedding とコサイン類似度計算
  4. 結果表示: スコア + 似ている芸能人 Top5

#### 結果表示
- 自分のスコア (レーダーチャート付き)
- 似ている芸能人 Top5 (類似度 % 付き)
- SNS シェアボタン (スコアのみ共有、画像は含めない)

### 2-3. UI/UX
```
┌─────────────────────────────────────┐
│  🏆 顔面偏差値ランキング            │
│  [ランキング] [診断する]             │
├─────────────────────────────────────┤
│  1. ██████ - 95.2点                 │
│  2. ██████ - 93.8点                 │
│  3. ██████ - 91.5点                 │
│  ...                                │
├─────────────────────────────────────┤
│  📸 あなたの顔を診断                │
│  ┌───────────────┐                  │
│  │  Drop Image   │                  │
│  └───────────────┘                  │
│                                     │
│  あなたのスコア: 78.5               │
│  似ている芸能人:                    │
│  1. ██████ (87% 一致)               │
│  2. ██████ (82% 一致)               │
│  3. ██████ (79% 一致)               │
└─────────────────────────────────────┘
```

---

## Phase 3: スコアアルゴリズム詳細 (Python & JS 共通ロジック)

### 黄金比の主要比率
```
顔幅 : 顔長 = 1 : 1.618
目間距離 : 顔幅 = 1 : 2.618  (≈ 0.382)
鼻幅 : 顔幅 ≈ 0.26
口幅 : 顔幅 ≈ 0.50
目の高さ位置 : 顔長 ≈ 0.50
鼻先位置 : 顔長 ≈ 0.67
```

### 対称性スコア
```python
symmetry = 1 - mean(|left_landmark - mirror(right_landmark)|) / face_width
```

### 総合スコア計算
```python
score = (
    symmetry * 0.20 +
    golden_ratio_fit * 0.25 +
    eye_score * 0.15 +
    nose_score * 0.10 +
    mouth_score * 0.10 +
    contour_score * 0.10 +
    skin_score * 0.10
) * 100
```

---

## Phase 4: プライバシー & セキュリティ

### 身バレ防止対策
- [ ] `.gitignore` にローカル固有情報を徹底排除
  - OS 固有ファイル、IDE 設定、ローカルパス含むファイル
- [ ] Git の author 情報を確認・必要に応じ匿名化
  - `git config user.name` / `user.email` を公開用に設定
- [ ] コード内にローカルパス・ユーザー名をハードコードしない
- [ ] 画像の EXIF 情報を除去してからコミット
- [ ] `package.json` の author フィールドを匿名に
- [ ] GitHub リポジトリ設定で個人情報非公開を確認

### ユーザーのプライバシー
- アップロード画像は **ブラウザ内のみで処理**（サーバー送信なし）
- 画像データは処理後即破棄（localStorage にも保存しない）
- プライバシーポリシーをサイトに明記

---

## ディレクトリ構成

```
face-ranking-and-lookalike/
├── PLAN.md
├── .gitignore
├── scripts/                    # オフライン前処理 (Python)
│   ├── requirements.txt
│   ├── collect_images.py       # 画像収集ヘルパー
│   ├── process_faces.py        # 顔検出・スコア算出・embedding抽出
│   ├── generate_data.py        # celebrities.json 生成
│   └── strip_exif.py           # EXIF除去
├── web/                        # フロントエンド (Vite + React)
│   ├── package.json
│   ├── vite.config.ts
│   ├── index.html
│   ├── public/
│   │   ├── data/
│   │   │   ├── celebrities.json
│   │   │   └── thumbnails/
│   │   └── models/             # face-api.js モデルファイル
│   └── src/
│       ├── App.tsx
│       ├── main.tsx
│       ├── pages/
│       │   ├── RankingPage.tsx
│       │   └── DiagnosePage.tsx
│       ├── components/
│       │   ├── CelebrityCard.tsx
│       │   ├── ScoreRadar.tsx
│       │   ├── ImageUploader.tsx
│       │   └── LookalikeResult.tsx
│       ├── lib/
│       │   ├── faceScoring.ts  # スコア算出ロジック (JS版)
│       │   ├── embedding.ts    # 類似度計算
│       │   └── faceDetection.ts
│       └── types/
│           └── celebrity.ts
└── .github/
    └── workflows/
        └── deploy.yml          # GitHub Pages デプロイ
```

---

## 実装順序

| Step | 内容 | 見積 |
|------|------|------|
| 1 | プロジェクト初期化 (.gitignore, Git匿名化, Vite セットアップ) | - |
| 2 | Python 前処理パイプライン (スコア算出アルゴリズム) | - |
| 3 | サンプルデータで celebrities.json 生成 | - |
| 4 | ランキングページ UI | - |
| 5 | face-api.js 統合 & 診断ページ | - |
| 6 | 類似芸能人マッチング実装 | - |
| 7 | レーダーチャート & 結果 UI | - |
| 8 | GitHub Pages デプロイ設定 | - |
| 9 | テスト & 調整 | - |

---

## 注意事項
- スコアはあくまで数学的指標であり、美の絶対評価ではないことをサイトに明記
- face-api.js のモデルファイルはサイズが大きい (~6MB)。CDN 利用も検討
