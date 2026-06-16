# SoC Hypervisorによる統合アーキテクチャ

**作成日：** 2026年6月16日  
**対象読者：** 技術担当者・アーキテクト  
**資料種別：** 社内検討用（社外秘）

---

## 概要

Classic AUTOSAR（RTOS系）とAdaptive AUTOSAR（Linux系）を単一SoC上のHypervisorで統合するアーキテクチャについて整理する。物理的に別SoCで分離する構成（デュアルSoC）との比較も含める。

---

## 1. Hypervisorとは

Hypervisorとは複数のOSを同一ハードウェア上で安全に分離・共存させるソフトウェア層。車載・組み込み向けにはType 1（ベアメタル型）が主流。

```
┌─────────────────────────────────┐
│  Classic AUTOSAR    │  Linux    │
│  （RTOS制御系）      │  （HMI・  │
│  ASIL-D              │  通信系）  │
│                     │  QM/ASIL-B│
├─────────────────────┴───────────┤
│         Hypervisor（Type 1）     │
│  干渉からの自由（FFI）を保証     │
├─────────────────────────────────┤
│            SoC ハードウェア       │
│  （NXP S32G / Renesas R-Car等）  │
└─────────────────────────────────┘
```

### Type 1 vs Type 2

| 種別 | 動作位置 | 特徴 | 車載採用 |
|---|---|---|---|
| **Type 1（ベアメタル）** | HW上で直接動作 | FFI保証・ASIL認証可・高リアルタイム性 | 主流 |
| **Type 2（ホストOS上）** | ホストOS上で動作 | 性能・安全性で劣る | 車載安全系には不向き |

---

## 2. 主要な車載Hypervisor製品

| 製品 | ベンダー | 安全認証 | 対応SoC |
|---|---|---|---|
| **QNX Hypervisor 8.0 for Safety** | BlackBerry | ISO 26262 ASIL-D | NXP S32G、Renesas R-Car S4 |
| **INTEGRITY-178 tuMP** | Green Hills | DO-178C / ASIL-D | 多数 |
| **XEN Project** | OSS（Linux Foundation） | ASIL認証なし | x86・ARM |
| **ACRN** | Linux Foundation | 組込み向けType 1 | Intel系 |

商用の安全認証済みHypervisorとしては**QNX Hypervisor for Safety**が車載実績で最も多い。

---

## 3. Classic + Adaptive を統合するメリット・デメリット

### メリット

- 単一SoC上でCAN/LIN制御（Classic）とEthernet/SOME-IP通信・AI処理（Adaptive/Linux）を共存
- SoC数削減によるコスト低減（デュアルSoC比で約20%コスト削減との試算あり）
- 配線・消費電力・基板スペースの削減
- OTA更新をAdaptive側で柔軟に対応できる

### デメリット

- Hypervisor自体のASIL認証・検証コストが高い
- リアルタイム性（レイテンシ・ジッタ）の保証がType 1でも課題になることがある
- ツールチェーン・デバッグが複雑化し、開発期間が伸びる傾向
- SoC故障で全系が停止するリスク

---

## 4. 機能安全とHypervisorの関係

### ASIL分解（Decomposition）

ASIL-Dの安全要求をASIL-B + ASIL-Bの独立チャネルに分割する手法。Hypervisorがゲスト間の干渉遮断（FFI）を保証することで成立する。

```
ASIL-D 安全要求
    ↓ 分解
ASIL-B チャンネルA（Classic AUTOSAR RTOS）
    ＋
ASIL-B チャンネルB（Adaptive AUTOSAR / Linux）
    ↑
Hypervisorが二者間のFFIを保証
```

実用例：Volvo EX90のボディコントローラがこのASIL混在構成を採用。

### ISO 25119（建設機械）への適用

ISO 25119はISO 26262をベースに農機・建機向けに定義されており、Hypervisorを用いたFFI実証はISO 25119でも適用可能と解釈されつつある。ただし**ISO 25119認証実績を持つHypervisor製品はほぼ存在しない**のが現状で、ISO 26262認証からの転用・論拠付けが必要になる。

---

## 5. 物理分離（デュアルSoC）との比較

| 観点 | 単一SoC + Hypervisor | デュアルSoC物理分離 |
|---|---|---|
| コスト | 低い（ハード削減） | 高い（SoC・基板追加） |
| FFI証明 | Hypervisor認証で対応 | 物理分離で自明 |
| リアルタイム性 | Hypervisorオーバーヘッドあり | 独立保証しやすい |
| 開発複雑度 | 高い（Hypervisor統合が必要） | 低い（独立して開発可能） |
| 障害影響範囲 | SoC故障で全系停止リスク | 分離によりフェールセーフしやすい |
| デバッグ | 複雑（クロスVM追跡が必要） | 比較的容易 |

---

## 6. 建設機械への採用可能性

### 適用が有望なケース

- 自律化（センサー融合・AI処理）とリアルタイム制御を同一筐体に搭載する次世代機
- 電動化によりECU統合のニーズが高まる機種
- スペース・重量・消費電力制約が厳しいコンパクト機

### 現状の課題

- 商用HypervisorのライセンスコストがECU単価に対して高い
- 建設機械OEMのHypervisorツールチェーン整備が未成熟
- ISO 25119認証実績を持つHypervisor製品がほぼ存在しない
- 過酷な電源品質（電圧変動・サージ）への対処がSoC選定で必要

### 農機での先行事例

農機オペレーションターミナル上でHypervisorを用いた安全系・非安全系の分離実行の特許が存在する。建設機械への展開はこれに続く形で進む可能性がある。

---

## 7. 建設機械における推奨判断

| 状況 | 推奨構成 |
|---|---|
| **現時点（2026年）・安全性優先** | デュアルSoC物理分離（Classic AUTOSAR ECU + Linux ECU）。実績・安全性・開発難易度のバランスが最良 |
| **2028年以降・コスト・スペース優先** | 認証済みHypervisor（QNX等）を用いた単一SoC統合を検討。ただしISO 25119認証の論拠構築が前提 |
| **自律化・AI処理が本格化する次世代機** | Hypervisor統合が有望。NXP S32G + QNX Hypervisorの組み合わせが現時点の有力候補 |

**現実的な結論：** 今すぐHypervisor統合に踏み込む必要はない。まずClassic AUTOSAR + Linuxのデュアル構成で土台を作り、Hypervisor統合は次世代機の設計段階で検討するロードマップが妥当。
