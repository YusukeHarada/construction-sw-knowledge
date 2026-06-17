# 建設機械向け 車載ソフトウェア開発 総合調査リポジトリ

**対象読者：** 開発部門内関係者  
**機密レベル：** 社外秘

建設機械向け車載ソフトウェア開発の現状課題・将来技術・移行戦略に関する個人知識整理リポジトリ。AUTOSAR移行・SDV化・機能安全・サイバーセキュリティ・E/Eアーキテクチャ・クラウド連携など、車載ソフトウェア開発全般を対象とする。

---

## 背景

建設機械の制御系・通信系ソフトウェアを内製で開発してきたが、以下の課題がある。

- 機能安全（ISO 25119）・セキュリティ規格への対応が必要だが、現状の内製体制では対応不可能
- 仕様書・設計書・テストが存在しない状態で4〜5年の開発が蓄積されている
- AUTOSARを自社で習得・導入するリソースがない
- SDV化（自律施工・フリート管理・OTA）の流れが建設機械にも波及しつつある

これらを踏まえ、**Classic AUTOSAR（RTOS系）＋ Linuxミドルウェア（Linux系）のハイブリッド構成を外部委託で構築**し、段階的にSDV化を進める方針を検討している。またE/Eアーキテクチャ・ビークルOS・V2X・遠隔操作・デジタルツインといった将来技術の調査・整理も並行して進める。

---

## ドキュメント一覧

### 戦略・提案

| ファイル | 内容 |
|---|---|
| [proposal.md](proposal.md) | 社内提案資料（なぜ変えるべきか） |
| [sdv.md](sdv.md) | SDVの概念と建設機械への考察 |
| [roadmap.md](roadmap.md) | フェーズ別ロードマップ・必要技術・人材 |
| [cost-budget.md](cost-budget.md) | ツール・委託・認証の費用試算とROI分析 |
| [construction-vs-automotive.md](construction-vs-automotive.md) | 建設機械が車載ソフトに遅れている理由・構造分析・今後の見込み |

### 外部委託・選定・移行

| ファイル | 内容 |
|---|---|
| [vendor-selection.md](vendor-selection.md) | 委託先選定基準・PoC評価 |
| [migration-plan.md](migration-plan.md) | 既存内製コードからAUTOSARへの移行計画 |

### 技術仕様

| ファイル | 内容 |
|---|---|
| [requirements-spec.md](requirements-spec.md) | AUTOSARを前提とした要求仕様の作り方 |
| [communication-spec.md](communication-spec.md) | CAN・Ethernet・J1939通信仕様の書き方 |
| [can-fd-tsn.md](can-fd-tsn.md) | CAN FD / Ethernet TSN 技術概念・建設機械への適用 |
| [toolchain.md](toolchain.md) | AUTOSARコード生成ツールチェーン |
| [autosar-modules.md](autosar-modules.md) | AUTOSARモジュール解説（SOME/IP・SoAD等） |
| [adaptive-autosar.md](adaptive-autosar.md) | Adaptive AUTOSAR主要モジュール（ara::com等）・2028年以降の検討準備 |
| [diagnostics.md](diagnostics.md) | UDS・診断通信（Dcm・Dem・J1939 DM1/DM2） |
| [j1939-detail.md](j1939-detail.md) | J1939プロトコル技術詳解（PGN・SPN・DBC・AUTOSAR連携） |
| [can-vs-j1939.md](can-vs-j1939.md) | 自動車CAN vs J1939 比較整理（フレーム構造・診断・AUTOSAR対応の違い） |
| [j1939-functional-safety.md](j1939-functional-safety.md) | J1939機能安全通信・アタッチメント対応（E2E保護・タイムアウト設計・J1939-75・アタッチメント別対応） |
| [bootloader.md](bootloader.md) | Bootloader / フラッシュプログラミング・OTA連携 |

### アーキテクチャ・技術概念

| ファイル | 内容 |
|---|---|
| [ee-architecture.md](ee-architecture.md) | E/Eアーキテクチャ変遷（分散型→ドメイン型→ゾーン型→集中型） |
| [vehicle-os.md](vehicle-os.md) | ビークルOS・オーケストレーション・CASE・ゾーンアーキ・セントラルECU |
| [incar.md](incar.md) | In-car領域（コックピット・HMI・車内ネットワーク・統合コックピットコントローラ） |
| [outcar.md](outcar.md) | Out-car領域（V2X・クラウド・フリート管理・遠隔操作・BIM/CIM連携） |
| [architecture.md](architecture.md) | 将来アーキテクチャ・セキュリティ・ゲートウェイ設計 |
| [hypervisor.md](hypervisor.md) | SoC Hypervisorによる統合アーキテクチャ |
| [cloud-connectivity.md](cloud-connectivity.md) | クラウド連携・OTA・IoT Core |
| [digital-twin.md](digital-twin.md) | デジタルツインの概念・建設機械への適用 |
| [electrification.md](electrification.md) | 建設機械の電動化・BMS・エネルギー管理・AUTOSARとの関係 |
| [autonomous-construction.md](autonomous-construction.md) | 自律施工・半自律化の技術概念・自動運転との比較・必要ソフトウェア技術 |
| [bim-cim.md](bim-cim.md) | BIM/CIM連携・Machine Guidance/Control・i-Construction |

### 規格対応

| ファイル | 内容 |
|---|---|
| [functional-safety.md](functional-safety.md) | ISO 25119のHARA・Safety Plan・証跡管理実施ガイド |
| [iso26262-vs-iso25119.md](iso26262-vs-iso25119.md) | ISO 26262 vs ISO 25119 の違い・委託先との共通言語 |
| [cybersecurity.md](cybersecurity.md) | ISO/SAE 21434のTARA・SecOC・Linuxセキュリティ実施ガイド |
| [regulations.md](regulations.md) | 建設機械の法規制・型式認証・排ガス規制・安全規則とソフトウェアへの影響 |

### 開発プロセス・品質

| ファイル | 内容 |
|---|---|
| [dev-process.md](dev-process.md) | CI/CD・アジャイル・TDD/ATDD・段階的整備 |
| [dev-process-standards.md](dev-process-standards.md) | ASPICE・Linux系プロセス標準の適用整理 |
| [verification-environment.md](verification-environment.md) | MIL/SIL/HIL検証環境・テスト自動化 |
| [hil-sil-setup.md](hil-sil-setup.md) | SIL/HIL環境構築具体例・ツール設定・CI統合・フォルトインジェクション |
| [automation.md](automation.md) | コード生成・テスト・トレーサビリティの自動化 |
| [shift-left.md](shift-left.md) | シフトレフトの考え方・具体的手法・AI活用 |
| [misra-c.md](misra-c.md) | MISRA-C準拠・コーディング標準・静的解析 要件定義 |
| [secoc.md](secoc.md) | SecOC（Secure Onboard Communication）実装ガイド・MAC生成・鍵管理 |
| [sw-requirements-template.md](sw-requirements-template.md) | 委託先向けソフトウェア要件仕様書（SRS）テンプレート |

### 知識整理・改善活動

| ファイル | 内容 |
|---|---|
| [process-improvement.md](process-improvement.md) | 開発プロセス改善の過去・現在・これから |
| [ai-utilization.md](ai-utilization.md) | 生成AI・AIエージェントの活用 |
| [ai-tools-landscape.md](ai-tools-landscape.md) | AIツール全体像（Claude Code・Copilot・ChatGPT等）と車載SW活用 |
| [dev-tools-landscape.md](dev-tools-landscape.md) | 車載SW開発ツール全体像（ALM・静的解析・単体テスト・HIL等） |
| [mbd.md](mbd.md) | MBD（モデルベース開発）概要・Simulink/SCADE・建設機械への適用 |
| [org-design.md](org-design.md) | 組織設計ベストプラクティス（チームトポロジー・コンウェイの法則） |
| [ota-sbom.md](ota-sbom.md) | OTA・SBOMの概念・規制・ツール・建設機械ロードマップ |

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

## フェーズ別アクションプラン

| フェーズ | 期間 | 主な取り組み |
|---|---|---|
| **Phase 1：土台整備** | 今〜2年 | 機能安全証跡・CI/CD・AUTOSAR PoC（Dcm+Dem）・フリート管理初版・SIL環境構築 |
| **Phase 2：移行完了** | 2〜5年 | Classic AUTOSAR本格稼働・クラウド連携・OTA（Linux系）・ISO 25119認証・HIL整備 |
| **Phase 3：SDV化** | 5〜10年 | 半自律化・Hypervisor統合検討・デジタルツイン・Adaptive AUTOSAR評価 |

---

## 関連規格

| 規格 | 内容 |
|---|---|
| ISO 25119 | 農業・建設機械の機能安全 |
| ISO 13849 | 機械安全（PLa〜PLe） |
| ISO/SAE 21434 | 自動車サイバーセキュリティ |
| ISO 14229（UDS） | 統合診断サービス |
| AUTOSAR Classic Platform | RTOS系ECU向けソフトウェア標準 |
| AUTOSAR Adaptive Platform | Linux系ECU向けソフトウェア標準（将来検討） |
| J1939 / J1939-22 | 建設・農業機械向けCAN上位プロトコル |
| SOME/IP | AUTOSAR標準のEthernetサービス通信プロトコル |
