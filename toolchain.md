# AUTOSARコード生成ツールチェーン

**作成日：** 2026年6月16日  
**対象読者：** 自社技術担当者・委託先評価担当  
**資料種別：** 社内運用資料（社外秘）

---

## 概要

通信仕様（DBC・ArXML）をインプットとして、AUTOSARのBSWコンフィグおよびコードを生成するツールチェーンの全体像を整理する。自社開発ではなく外部委託を前提とするが、ツールの役割を理解することで成果物のレビューと委託先の評価が可能になる。

---

## 1. ツールチェーン全体像

```
【インプット】
  通信仕様書（DBC / ArXML / Fibex）
  要求仕様書（SWC定義・ポート定義）
  ECU仕様（マイコン・OS・BSW構成）
       ↓
【Step 1】システム記述ツール
  AUTOSARシステム全体のArXML作成
  （SWC定義・ネットワーク定義・ECUマッピング）
       ↓
【Step 2】BSWコンフィグツール
  BSW各モジュールのパラメータをArXMLで設定
  （COM・CANIF・PDUR・OS・RTE等）
  + MCAL（Microcontroller Abstraction Layer）コンフィグ
    ※マイコンベンダー提供（Renesas Smart Configurator・NXP AUTOSAR MCAL等）
      またはツールベンダー提供（Vector MCAL）
       ↓
【Step 3】コード生成ツール
  ArXMLからCソースコードを自動生成
  （RTE・BSWコンフィグコード）
       ↓
【Step 4】静的解析・MISRA-Cチェック
  機能安全対応のためMISRA-C:2012準拠チェックが必要
  （Polyspace / PC-lint Plus / Helix QAC等）
       ↓
【Step 5】コンパイル・リンク
  生成コード＋アプリSWC（手書き）→ ECUバイナリ
       ↓
【Step 6】テスト・検証
  CANoe / テスト自動化ツール（TPT・EXAM等）
       ↓
【アウトプット】
  ECU実行バイナリ + テストレポート + 設計ドキュメント + 機能安全成果物
```

---

## 2. 主要ツールと役割

### 2-1. Vectorグループ（最も普及しているツールチェーン）

| ツール名 | 役割 | 入力 | 出力 |
|---|---|---|---|
| **DaVinci Developer** | SWC設計ツール | 要求仕様 | SWC定義ArXML |
| **DaVinci Configurator Pro** | BSWコンフィグ＋コード生成（統合環境） | SWC ArXML・DBC | BSWコンフィグArXML・Cコード |
| **CANdb++** | CAN DB管理ツール | 手入力 | DBC |
| **CANoe** | 通信シミュレーション・テスト | DBC・バイナリ | テスト結果 |

**備考：** DaVinci Generatorは現行バージョンではDaVinci Configurator Proに統合されている（旧バージョンでは分離していた）。委託先に使用バージョンを確認すること。

**特長：** 実績が最も多く、委託先候補のほとんどが対応している。ライセンスコストは高め。

### 2-2. Elektrobitグループ（EB tresos）

| ツール名 | 役割 | 備考 |
|---|---|---|
| **EB tresos Studio** | BSWコンフィグ＋コード生成 | Eclipseベース、オープン性が高い |
| **EB tresos AutoCore** | BSW本体 | AUTOSAR Classic対応 |

**特長：** DaVinciよりオープンなアーキテクチャ。EB（Elektrobit）は車載ソフトウェア専門企業。

### 2-3. ETAS（Boschグループ）

| ツール名 | 役割 |
|---|---|
| **ETAS ISOLAR-A** | AUTOSAR SWC開発環境 |
| **ETAS ISOLAR-EVE** | 仮想ECU（バイナリなしでテスト可能） |
| **RTA-OS** | AUTOSAR OS |
| **RTA-BSW** | BSWスタック |

**特長：** Bosch系のプロジェクトで多く使われる。仮想ECU環境が充実しており、HIL未整備の場合の代替確認に活用できる。

### 2-4. ORIENTAIS（東軟グループ・アジア系委託先で多い）

| ツール名 | 役割 |
|---|---|
| **ORIENTAIS Configurator** | BSWコンフィグ＋コード生成 |
| **ORIENTAIS OS** | AUTOSAR OS |

**特長：** コストが比較的低い。中国・アジア系の委託先で使われることが多い。国内実績は少ないため、採用する場合は機能安全対応の実績を特に慎重に確認すること。

---

## 3. 通信仕様からコード生成までの流れ（具体例）

### Step 1：DBC → ArXML変換

CANメッセージリスト（DBC）をAUTOSARの通信記述（ArXML）に変換する。

```
ツール：Vector DaVinci Configurator Pro または CANdb++ + スクリプト

入力：HydraulicCommand.dbc（メッセージ・シグナル定義）

出力（ArXML）：
  <I-PDU>
    <SHORT-NAME>HydraulicCommand_IPDU</SHORT-NAME>
    <LENGTH>8</LENGTH>
  </I-PDU>
  <I-SIGNAL>
    <SHORT-NAME>BoomLiftCmd</SHORT-NAME>
    <START-POSITION>0</START-POSITION>
    <LENGTH>16</LENGTH>
  </I-SIGNAL>
```

### Step 2：BSWコンフィグ

COMモジュール・PDURモジュール・CANIFモジュール・MCALの設定をArXMLで行う。

```
DaVinci Configurator Proでの設定例：

COMモジュール：
  - HydraulicCommand_IPDU をTxPDUとして登録
  - 送信周期：10ms
  - BoomLiftCmd シグナルをCOM Signalとして登録

CANIFモジュール：
  - CAN ID 0x100 を HydraulicCommand_IPDU にマッピング
  - 対象CANコントローラ：CAN1

PDURモジュール：
  - COMからCANIFへのルーティングを定義

MCALモジュール（MCAL CanDriver）：
  - CANコントローラの物理設定（クロック・ボーレート・ピン）
  - マイコンベンダー提供ツールで設定することが多い
```

### Step 3：RTE生成

SWCのポート定義からRTE（Runtime Environment）のCコードを自動生成する。

```c
/* 自動生成されるRTEのAPIイメージ */

/* HydraulicControlSWC が呼び出すAPI */
Std_ReturnType Rte_Read_OperatorInput_BoomLiftCmd(uint16 *data);
Std_ReturnType Rte_Write_ValveCommand_BoomLiftCmd(uint16 data);

/* 開発者が実装するRunnable（手書き部分）*/
void HydraulicControl_10ms(void)
{
    uint16 operatorInput;
    uint16 valveCmd;

    Rte_Read_OperatorInput_BoomLiftCmd(&operatorInput);

    /* ここにアプリケーションロジックを実装 */
    valveCmd = ConvertToValveCommand(operatorInput);

    Rte_Write_ValveCommand_BoomLiftCmd(valveCmd);
}
```

**重要：** RTE・BSWのコードは自動生成。開発者が書くのはSWC内部のアプリケーションロジックのみ。

---

## 4. Adaptive AUTOSAR（Linux系）のツールチェーン

Linux系（HMI・ゲートウェイ・人検知等）にはAdaptive AUTOSARが対応する。

**注：** Adaptive AUTOSARは自動車業界でも普及途上であり、Classic AUTOSARと比較してツール・実績・ノウハウが少ない。採用判断は慎重に行うこと（後述の「Adaptive AUTOSARの現状」を参照）。

| ツール・フレームワーク | 役割 | 備考 |
|---|---|---|
| **AUTOSAR Adaptive Platform（AP）** | ミドルウェア基盤 | SOME/IP・ARA APIを提供 |
| **Vector MICROSAR.AP** | Adaptive AUTOSAR実装 | 商用 |
| **ETAS RTA-AP** | Adaptive AUTOSAR実装 | 商用 |
| **vsomeip（COVESA OSS）** | SOME/IP実装 | オープンソース。サポートなしのため機能安全用途には不向き |
| **Manifest Editor** | サービス・実行マニフェスト定義ツール | ArXML形式で設定（JSON形式は一部商用ツールの内部表現） |

Adaptive AUTOSARではアプリはC++14/17で実装し、ara::com APIを通じてSOME/IPサービスにアクセスする。クロスコンパイル環境（例：Yocto + arm-poky-linux-gnueabi）の整備も委託先に求めること。

### Adaptive AUTOSARの現状

自動車業界でもAdaptive Platform（AP）の実用化は道半ばである。以下の課題が指摘されている。

- **ツール・ノウハウが未成熟：** Classic AUTOSARと比較して対応ベンダー・事例が少ない
- **機能安全対応が難しい：** Linux上での機能安全認証は複雑であり、ASIL-Cを超えるレベルへの対応は限定的
- **規格改訂が速い：** R22-11・R23-11等のリリースが続いており、ツールバージョンの追従が必要

**現実的な判断：** Linux系（HMI・人検知・クラウド通信）は機能安全要件が低い場合が多いため、Adaptive AUTOSARにこだわらず「Linuxベースのミドルウェア（非AUTOSAR）」で構築するアプローチも十分合理的。SOME/IPはAdaptive AP非採用でも使用できる。

---

## 5. 自社担当者が最低限把握すべきこと

外部委託で開発してもらう場合でも、以下は自社側で理解しておく必要がある。

| 理解レベル | 内容 |
|---|---|
| **必須** | ArXMLの構造（何がどこに定義されているか読める・差分を確認できる） |
| **必須** | SWC・ポート・Runnableの概念 |
| **必須** | 通信設定（CANIF・COM・PDUR）の役割と設定項目 |
| **必須** | CANoeまたは代替ツールを使ったCAN通信の確認方法 |
| **必須** | J1939のPGN・SPN・SAの基本概念（建設機械での外部I/Fに必須） |
| **推奨** | DaVinci Configurator Proの基本操作 |
| **推奨** | 静的解析ツール（Polyspace等）のレポートの読み方 |
| **任意** | コード生成の仕組みの詳細 |

**習得方法：** PoC期間中に委託先から教えてもらうことを契約に含めるのが最も効率的。「成果物のレビュー方法をレクチャーすること」を発注仕様に明記する。

**ArXMLのレビュー環境：** 自社ではビューアーレベルのツールで十分。DaVinciの無償Viewer、またはNotepad++のXPath検索機能を活用する。数千行のArXMLをテキストで全部読む必要はない。

---

## 6. ツールライセンスの費用感

自社でツールを購入する必要はない。委託先がツールを保有していることを選定条件にし、ライセンスコストを委託先に持たせること。自社に必要なのは成果物（ArXML・ソースコード）を確認するためのビューアーレベルのツールで十分。

| ツール | 費用感 | 推奨 |
|---|---|---|
| Vector DaVinci Configurator Pro | 高コスト（年間ライセンス） | 委託先が保有 |
| Vector CANoe | 高コスト（モジュール構成で変動） | 委託先が保有。受け入れ検証でのみ要確認 |
| EB tresos Studio | 中〜高コスト | 委託先が保有 |
| ETAS ISOLAR | 中コスト | 委託先が保有 |
| 静的解析ツール（Polyspace等） | 中〜高コスト | 委託先が実施・レポート納品 |
