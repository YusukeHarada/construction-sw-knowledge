# MISRA-C準拠・コーディング標準・静的解析 要件定義

対象読者：技術担当者・委託先  
関連ファイル：`requirements-spec.md` / `dev-process.md` / `functional-safety.md` / `vendor-selection.md`

---

## 1. MISRA-Cとは

### 1.1 概要

MISRA（Motor Industry Software Reliability Association）は、英国の自動車産業を中心に設立された団体であり、組込みCソフトウェアの安全性・信頼性を高めるためのコーディングガイドラインを策定している。

MISRA-Cは元々自動車向けに策定されたが、現在は航空・鉄道・医療・建設機械など安全クリティカルなシステム全般で広く採用されている。

### 1.2 バージョンと現在の主流

| バージョン | 発行年 | 状態 |
|---|---|---|
| MISRA-C:1998 | 1998年 | 廃止 |
| MISRA-C:2004 | 2004年 | 廃止推奨 |
| MISRA-C:2012 | 2012年（2019年Amendment 1追加） | **現在の主流** |

**MISRA-C:2012（第3版）** を適用すること。旧バージョン（2004）のままの委託先には必ずバージョンアップを要求する。

### 1.3 ルール体系

MISRA-C:2012のルールは以下の3分類で構成される。

| 分類 | 説明 | 対応方針 |
|---|---|---|
| **Mandatory** | 例外なく遵守が必要。逸脱不可 | 違反はリリースブロッカー |
| **Required** | 原則遵守。逸脱には文書化が必要 | 逸脱管理プロセスを適用 |
| **Advisory** | 推奨ルール。逸脱は許容されるが記録を残す | プロジェクト方針に応じて判断 |

ルール総数：175ルール＋16ガイドライン（要確定。Amendment 1追加後の数値）。

---

## 2. ISO 25119 / Classic AUTOSARとの関係

### 2.1 ISO 25119 におけるMISRA-C準拠の位置づけ

ISO 25119（農業・林業・建設機械の機能安全規格）は、ISO 26262をベースに農機・建機領域へ適用したものである。いずれもソフトウェア開発プロセスに「コーディングガイドラインの適用」を要求しており、MISRA-Cは事実上の標準として業界で認知されている。

- **AgPL（Agriculture Performance Level）b以上** の開発では、コーディング標準の適用と静的解析による検証が求められる
- 認証審査において、MISRA-C適合レポートは客観的証跡として有効

> ISO 26262との比較は `functional-safety.md` を参照。委託先が自動車業界出身の場合、ISO 26262のSIL相当概念をISO 25119のAgPLに読み替えて説明すると共通言語として機能する。

### 2.2 AUTOSARのBSW生成コードとMISRA-Cの関係

DaVinci Developer / EB tresosなどのAUTOSARツールチェーンが生成するBSWコードは、ツールベンダーがMISRA-C準拠を宣言していることが多い。ただし以下の点に注意する。

- **生成コードには適用除外（Deviation）が認められる場合がある**：ツールが生成するコードに対してエンジニアが手動でMISRA-Cを修正することは現実的でなく、ツールベンダーの適合宣言をもって代替とするのが一般的
- **手書きコード（Application Layer・SWC）は必須適用対象**：委託先が実装するSWCおよびインテグレーションコードは例外なくMISRA-C:2012を適用すること
- ツールベンダーの「MISRA-C適合証明書」または「Deviation記録」を委託先から入手・保管すること

---

## 3. MISRA-Cの主要なルールカテゴリと違反例

### 3.1 必須ルール（Mandatory）の代表例

| ルール番号 | 内容 |
|---|---|
| Rule 1.3 | 未定義動作（Undefined Behavior）を引き起こすコードの禁止 |
| Rule 2.1 | 到達不能コードの禁止 |
| Rule 13.6 | `sizeof` 演算子のオペランドに副作用のある式を使用しない |
| Rule 17.3 | 暗黙的な関数宣言の禁止 |

### 3.2 よくある違反パターンと修正例

**（1）符号なし整数のラップアラウンド（Rule 12.4 / Required）**

```c
/* NG: uint8_t の最大値を超えるとラップアラウンドする */
uint8_t count = 255U;
count++;  /* 0になる。意図しない動作 */

/* OK: 上限チェックを追加 */
if (count < 255U) {
    count++;
}
```

**（2）暗黙的な型変換（Rule 10.3 / Required）**

```c
/* NG: int から uint8_t への暗黙変換（データ損失のリスク） */
int32_t val = 300;
uint8_t buf = val;  /* 上位ビットが切り捨てられる */

/* OK: 明示的なキャストとチェック */
if ((val >= 0) && (val <= 255)) {
    uint8_t buf = (uint8_t)val;
}
```

**（3）関数の戻り値の無視（Rule 17.7 / Required）**

```c
/* NG: 戻り値を捨てている */
memcpy(dst, src, size);

/* OK: 戻り値を使用するか、明示的に破棄を示す */
(void)memcpy(dst, src, size);
```

**（4）動的メモリ割り当ての禁止（Rule 21.3 / Required）**

```c
/* NG: malloc / free は使用禁止 */
uint8_t *buf = (uint8_t *)malloc(128U);

/* OK: 静的配列またはスタック上の変数を使用 */
static uint8_t buf[128U];
```

### 3.3 逸脱（Deviation）の管理方法

Mandatory以外のルールに対して逸脱が必要な場合は、以下のプロセスで管理する。

1. **逸脱理由の記述**：なぜそのルールに従えないかを技術的に説明
2. **代替安全策の記載**：ルールの意図をどのように別の手段で担保するか
3. **レビュー・承認**：技術リードおよびプロジェクト責任者の承認
4. **ツールへの登録**：静的解析ツールのSuppression機能に逸脱を登録し、管理番号を付与
5. **定期的な見直し**：毎リリースサイクルで逸脱一覧を棚卸し

---

## 4. 静的解析ツール

### 4.1 主要ツール比較

| ツール | ベンダー | MISRA-C:2012対応 | CI/CD連携 | 特徴 |
|---|---|---|---|---|
| **Helix QAC** | Perforce | ◎ | GitLab CI / Jenkins対応 | MISRA-C準拠確認に最も実績が多い |
| **Polyspace Bug Finder** | MathWorks | ◎ | Jenkins / GitHub Actions対応 | MBD（Simulinkコード生成）との親和性が高い |
| **PC-lint Plus** | Gimpel Software | ○ | Makefileやシェル経由で統合可能 | 軽量・低コスト。大規模プロジェクトには不向きな場合あり |
| **Klocwork** | Perforce | ○ | CI連携プラグインあり | セキュリティ解析との複合利用に強み |
| **Coverity** | Synopsys | △（MISRA対応は限定的） | CI/CD対応 | バグ検出に強いが、MISRA準拠確認にはQACが上位 |

- ツール選定は委託先との協議の上で決定すること（要確定）
- 本プロジェクトでは **Helix QAC または Polyspace** を第一候補とする

### 4.2 CI/CDパイプラインへの組み込み

GitLab CIを例とした組み込み方法の概要。

```yaml
# .gitlab-ci.yml（概略）
stages:
  - build
  - static-analysis
  - test

misra-check:
  stage: static-analysis
  script:
    - qac analyze --project misra_project.xml --output-format csv
    - qac report --fail-on-mandatory  # Mandatoryルール違反でパイプライン失敗
  artifacts:
    paths:
      - misra_report.csv
    expire_in: 30 days
```

- **Mandatoryルール違反はパイプライン失敗（ブロッカー）** とすること
- Requiredルール違反は警告として記録し、週次レビューで件数を管理する
- AdvisoryルールはTrendレポートで追跡する

---

## 5. 委託先への要件定義ポイント

### 5.1 SOW（Statement of Work）への記載事項

| 項目 | 要求内容 |
|---|---|
| 適用規格バージョン | MISRA-C:2012（Amendment 1含む）。旧バージョンは不可 |
| ルール適用レベル | Mandatory：全適用・例外なし。Required：逸脱は逸脱管理票で管理。Advisory：適用目標とし逸脱は記録のみ |
| 静的解析ツール | Helix QAC または Polyspace（要協議・要確定）。ツールのバージョンも指定 |
| 解析実施タイミング | 全コミット時（CI自動実行）＋マイルストーン成果物として最終レポート提出 |
| 逸脱管理プロセス | 逸脱管理票（逸脱理由・代替安全策・承認者・管理番号）の提出を必須とする |
| 生成コードの扱い | ツールベンダーのMISRA適合証明または逸脱記録を提出すること |
| レポート形式 | CSV + HTMLサマリー。Mandatory違反件数がゼロであることを確認可能な形式 |

### 5.2 生成コードの扱い方針

```
適用区分：

  [AUTOSARツール生成コード（BSW・RTE）]
      → ツールベンダーの適合証明書で代替。個別適用は免除
      → ただし証明書の提出を成果物として要求

  [委託先が手書きするコード（SWC・Application Layer・統合部）]
      → MISRA-C:2012 全面適用。Mandatoryゼロを納品条件とする

  [既存内製コードを流用する部分]
      → 「既存流用」であることを明示し、別管理（後述 §6 参照）
```

### 5.3 レポート提出フォーマット

- **マイルストーン毎**（設計完了・コーディング完了・統合テスト完了）に静的解析レポートを提出
- 最低限の記載項目：
  - 解析対象ファイル一覧
  - ルール違反件数サマリー（Mandatory / Required / Advisory 別）
  - 逸脱管理票（逸脱件数・管理番号・理由の要約）
  - 前回比較（違反件数トレンド）

---

## 6. 建設機械固有の考慮事項

### 6.1 既存内製コードのMISRA-C適合状況の想定

本プロジェクトの出発点として、現状の内製コードは以下の状態と想定される。

- 仕様書なし・コードレビューなし・静的解析ツールの適用実績なし
- コーディングルールが存在しない、または文書化されていない
- MISRA-C違反は多数存在する可能性が高い（要確定。実態調査が先決）

このため、**既存内製コードをそのまま流用することは原則禁止**とし、新規開発またはリファクタリング（MISRA対応済み）したコードに置き換える方針とする。

### 6.2 新規開発分と既存流用分の切り分け方針

| 区分 | 対象 | MISRA-C要件 |
|---|---|---|
| **新規開発** | AUTOSARへの移行に伴い新規に実装するSWC・ドライバ | MISRA-C:2012 全面適用（Mandatoryゼロ必須） |
| **改修流用** | 既存内製コードをベースにリファクタリングするもの | 同上。リファクタリング完了を納品条件とする |
| **暫定流用（移行期限付き）** | 短期的にやむなく既存コードを流用するもの | MISRA非準拠を明示し、移行フェーズ内に置き換えを完了させる。AgPL b未満の非安全クリティカル機能に限定 |

> **暫定流用は例外的措置**である。AgPLb以上の機能安全要求がある機能に対しては、工数・期間に関わらず新規開発またはMISRA準拠改修を行うこと。

### 6.3 移行フェーズにおける現実的な進め方

1. **Phase 1（〜1年目）**：新規SWCのみMISRA-C全面適用。既存コードの違反実態を静的解析で可視化
2. **Phase 2（〜3年目）**：安全クリティカル機能の既存コードを順次置き換え。Mandatoryゼロを目標
3. **Phase 3（〜5年目）**：全コードベースのMISRA-C準拠完了。IS0 25119認証に向けた証跡整備

---

## 参考資料

- MISRA C:2012 – Guidelines for the use of the C language in critical systems（MISRA公式）
- AUTOSAR Release 4.x – General Requirements on Basic Software Modules
- ISO 25119-3:2018 – Tractors and machinery for agriculture and forestry (参考)
- `functional-safety.md`：ISO 25119 HARA・Safety Plan・証跡管理
- `toolchain.md`：AUTOSARコード生成ツールチェーン
- `dev-process.md`：CI/CD・GitLab構成
- `vendor-selection.md`：委託先選定基準・PoC評価チェックリスト
