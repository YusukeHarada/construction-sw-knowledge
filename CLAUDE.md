# CLAUDE.md

このリポジトリはAUTOSAR準拠プラットフォームソフトウェア移行、およびSDV化に向けた建設機械向け総合検討資料をまとめたものである。

---

## リポジトリの目的

建設機械向けプラットフォームソフトウェアを、現状の内製開発体制からAUTOSAR準拠・外部委託体制へ移行し、段階的にSDV化を進めるための**個人の知識整理用リポジトリ**。業務使用ではなく知識を整理・蓄積する目的で管理している。

---

## ドキュメント構成（全29ファイル）

### 戦略・提案

| ファイル | 内容 | 対象読者 |
|---|---|---|
| `proposal.md` | 社内提案資料（なぜ変えるべきか） | 非エンジニアの管理職にも伝わる平易な文体 |
| `sdv.md` | SDV概念・建設機械への考察・ロードマップ | 開発部門・経営層 |
| `roadmap.md` | フェーズ別ロードマップ・必要技術・人材 | 開発部門・管理職 |
| `cost-budget.md` | ツール・委託・認証の費用試算とROI分析 | 管理職・経営層 |

### 外部委託・選定・移行

| ファイル | 内容 | 対象読者 |
|---|---|---|
| `vendor-selection.md` | 委託先選定基準・PoC評価チェックリスト | 技術担当者 |
| `migration-plan.md` | 既存内製コードからAUTOSARへの移行計画 | 技術担当者・管理職 |

### 技術仕様

| ファイル | 内容 | 対象読者 |
|---|---|---|
| `requirements-spec.md` | AUTOSARを前提とした要求仕様の作り方 | 技術担当者・委託先 |
| `communication-spec.md` | CAN・Ethernet・J1939通信仕様の書き方 | 技術担当者・委託先 |
| `toolchain.md` | AUTOSARコード生成ツールチェーン（DaVinci・EB tresos等） | 技術担当者 |
| `autosar-modules.md` | AUTOSARモジュール解説（SOME/IP・SoAD・SOA等）※SOAセクション追加済み | 技術担当者 |
| `diagnostics.md` | UDS・診断通信（Dcm・Dem・J1939 DM1/DM2・DoCAN・DoIP・SOVD）※セクション9〜11追加済み | 技術担当者 |

### アーキテクチャ

| ファイル | 内容 | 対象読者 |
|---|---|---|
| `architecture.md` | 将来アーキテクチャ・セキュリティ・ゲートウェイ設計 | 技術担当者・アーキテクト |
| `hypervisor.md` | SoC Hypervisorによる統合アーキテクチャ（QNX・XEN比較・デュアルSoC推奨理由） | 技術担当者・アーキテクト |
| `cloud-connectivity.md` | クラウド連携・OTA・IoT Core | 技術担当者・アーキテクト |
| `digital-twin.md` | デジタルツインの概念・建設機械への適用 | 技術担当者・アーキテクト |

### 規格対応

| ファイル | 内容 | 対象読者 |
|---|---|---|
| `functional-safety.md` | ISO 25119のHARA・Safety Plan・証跡管理実施ガイド | 技術担当者・安全担当 |
| `cybersecurity.md` | ISO/SAE 21434のTARA・SecOC・Linuxセキュリティ実施ガイド | 技術担当者 |

### 開発プロセス・品質

| ファイル | 内容 | 対象読者 |
|---|---|---|
| `dev-process.md` | CI/CD・アジャイル・TDD/ATDD・段階的整備 | 開発部門 |
| `dev-process-standards.md` | ASPICE・Linux系プロセス標準の適用整理（現状課題・To-Be） | 開発部門・管理職 |
| `verification-environment.md` | MIL/SIL/HIL検証環境・テスト自動化 | 技術担当者 |
| `automation.md` | コード生成・テスト・トレーサビリティの自動化 | 技術担当者 |
| `shift-left.md` | シフトレフトの考え方・具体的手法・AI活用 | 開発部門 |

### 知識整理・改善活動

| ファイル | 内容 | 対象読者 |
|---|---|---|
| `process-improvement.md` | 開発プロセス改善の過去・現在・これから | 開発部門 |
| `ai-utilization.md` | 生成AI・AIエージェントの活用 | 開発部門 |
| `ai-tools-landscape.md` | AIツール全体像（Claude Code・Copilot・ChatGPT等）と車載SW活用の将来展望 | 技術担当者 |
| `dev-tools-landscape.md` | 車載SW開発ツール全体像（ALM・静的解析・単体テスト・HIL等）予算規模別推奨構成付き | 技術担当者 |
| `mbd.md` | MBD（モデルベース開発）概要・Simulink/SCADE・MIL/SIL/HIL・建設機械への適用 | 技術担当者 |
| `org-design.md` | 組織設計ベストプラクティス（チームトポロジー・コンウェイの法則・逆コンウェイ） | 管理職・開発部門 |
| `ota-sbom.md` | OTA・SBOMの概念・規制（WP.29 R156・EU CRA）・ツール・建設機械ロードマップ | 技術担当者・アーキテクト |

---

## 背景・前提

- 対象：建設機械向けプラットフォームソフトウェア（BSW・RTE相当）
- RTOS系：油圧制御等の安全クリティカルな制御系（Classic AUTOSAR領域）
- Linux系：HMI・ゲートウェイ・人検知・クラウド通信（Linuxミドルウェア、Adaptive AUTOSARは将来検討）
- 機能安全（ISO 25119）およびサイバーセキュリティ（ISO/SAE 21434）への対応が必要
- 外部委託でClassic AUTOSAR準拠システムを構築し、内製コードを段階的に廃止する方針
- クラウド連携はLinuxゲートウェイ経由でAWS IoT Core / Azure IoT Hubに接続
- Hypervisor統合は2028年以降の次世代機で検討

### 現状の課題（重要な前提）

- 仕様書・設計書・テストが存在しない状態で4〜5年の開発が蓄積
- 機能安全は形式的な対応にとどまっており、実効性がない
- 問題認識が会社全体に共有されていない
- AUTOSARを自社で習得・導入するリソースがない

---

## 移行方針の概要

```
現状                          移行後（〜5年）              将来（5〜10年）
─────────────────────────────────────────────────────────────────────────
内製BSW・RTE（仕様書なし）  →  Classic AUTOSAR（外部委託）  →  安定稼働・機能安全認証
内製Linuxミドルウェア       →  Linuxミドルウェア（外部委託） →  Adaptive AUTOSAR検討
独自通信定義                →  DBC + ArXML 標準化           →  SOME/IP フル活用
クラウド未連携              →  Linux GW経由 IoT Core連携    →  OTA・予知保全・自律化
テストなし・CIなし          →  SIL + GitLab CI 構築         →  HIL・デジタルツイン連携
```

---

## 用語統一ルール

- Classic AUTOSAR（× AUTOSAR Classic）
- Adaptive AUTOSAR（× AUTOSAR Adaptive Platform、AP）
- ArXML（× arxml、ARXML）
- J1939（× SAE J1939）
- SOME/IP（× SomeIP、someip）

---

## AIアシスタントへの注意事項

- **具体的な数値・製品名・バージョン番号は自社確認済み情報以外は「要確定」と記載する**
- `proposal.md` は非エンジニアの幹部向けのため専門用語には必ず補足を付ける
- `requirements-spec.md`・`communication-spec.md` のサンプル値は仮の値。実際のシステムへの具体化はユーザー自身が行う
- 各ドキュメントは独立して読める構成にする（委託先・幹部が単独で受け取る場合を想定）
- ステータス管理（Draft/In Review/Approved）は不要。個人の知識整理リポジトリのため
- 新しいドキュメントを作成したらREADME.mdのドキュメント一覧とこのCLAUDE.mdのドキュメント構成表を必ず更新する
- ドキュメントを追記・修正したら必ずgit commit & git push origin mainまで行う

---

## これまでの作業で判明した重要な設計判断

### アーキテクチャ判断
- **現時点（2026年）はデュアルSoC物理分離が現実解**：Classic AUTOSAR ECU + Linux ECUの構成。Hypervisor統合はISO 25119認証実績なし・スキル不足のため2028年以降
- **診断はDoCAN主体・DoIPは将来**：現状J1939 DM1とUDS over CANの共存。DoIPはEthernet移行後・OTA書込高速化のタイミングで導入
- **SOVDは2030年以降の検討項目**：規格化途上。クラウド連携診断の将来像として把握しておく

### 未着手の検討候補トピック
以下は「キーワードは出てくるが独立したまとめがない」状態。優先度の高いものから整理を進める。

| トピック | 関連ファイル | 追加価値 |
|---|---|---|
| CAN FD / Ethernet TSN の技術概念 | `communication-spec.md`（仕様書の書き方はあるが技術解説がない） | ハード選定・委託先との仕様協議に必要 |
| E/Eアーキテクチャ変遷（ドメイン型→ゾーン型→集中型） | `sdv.md`（一言のみ） | AUTOSARやSDV移行の「なぜ」の理解に直結 |
| Bootloader / フラッシュプログラミング | `diagnostics.md`（OTA連携で少し言及） | OTA実装・UDS書込の詳細として重要 |
| ISO 26262 vs ISO 25119 の違い | `functional-safety.md`（ISO 25119のみ） | 委託先（自動車業界出身）との共通言語 |
| Adaptive AUTOSAR 主要モジュール（ara::com等） | `autosar-modules.md`（Classic中心） | 2028年以降の検討準備 |
