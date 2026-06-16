# AUTOSARコード生成ツールチェーン

**作成日：** 2026年6月16日  
**対象読者：** 自社技術担当者・委託先評価担当  
**資料種別：** 社内運用資料（社外秘）

---

## 概要

通信仕様（DBC・ArXML）をインプットとして、AUTOSARのBSWコンフィグおよびコードを生成するツールチェーンの全体像を整理する。自社開発ではなく外部委託を前提とするが、ツールの役割を理解することで成果物のレビューと委託先の評価が可能になる。

**なぜ自社担当者がツールチェーンを理解する必要があるのか：**  
「委託先に任せる」だけでは、設定ミスや設計の手抜きを発注者側で検出できない。たとえば「COMモジュールのDeadline Monitoringが未設定なのにテストをパスしている」「MISRAチェックをかけずにコードを納品している」などの問題を見抜くには、ツールチェーンの各ステップで何がどのように設定・生成されるかを知っている必要がある。本書はそのための最低限の知識を提供する。

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

**ステップ間の成果物の関係：**

```
DBC ──────────► ArXML（通信定義）─────────────────────┐
（Step1）         （Step1でインポート）                  │
                                                       ▼
要求仕様書 ──► SWC ArXML ─────────────────► BSWコンフィグArXML
（手動作成）   （DaVinci Developerで作成）    （DaVinci Configurator Proで設定）
                                                       │
                                                       ▼
                                              Cソースコード（自動生成）
                                                       │
                                              ┌────────┴─────────┐
                                              │                  │
                                       RTE_xxx.c          Com_cfg.c 等
                                       （RTEコード）       （BSW設定コード）
                                              │
                                              ▼
                              手書きSWC（ConvertToValveCommand等）と合体
                                              │
                                              ▼
                                         ECUバイナリ
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

**DaVinci Configurator Pro（通称DaVinci CP）の役割イメージ：**
```
         DaVinci Configurator Pro
         ┌────────────────────────────────────────┐
         │                                        │
DBC ───► │ Import  → COM・CANIF・PDUR設定画面   │ ──► ArXML（BSWコンフィグ）
ArXML──► │         → パラメータをGUIで設定       │ ──► Cコード（自動生成）
         │         → 設定の整合性を自動チェック  │
         │         → コード生成ボタンを押す      │
         └────────────────────────────────────────┘
  ※ Excelで手動設定するのではなく、DBCをインポートすれば
    COM/CANIF/PDURの設定の大部分が自動で埋まる
```

### 2-2. Elektrobitグループ（EB tresos）

| ツール名 | 役割 | 備考 |
|---|---|---|
| **EB tresos Studio** | BSWコンフィグ＋コード生成 | Eclipseベース、オープン性が高い |
| **EB tresos AutoCore** | BSW本体 | AUTOSAR Classic対応 |

**特長：** DaVinciよりオープンなアーキテクチャ。EB（Elektrobit）は車載ソフトウェア専門企業。Tier1メーカーからの採用実績が多く、ツールそのものの機能安全認定（ISO 26262 tool confidence）を取得している。

### 2-3. ETAS（Boschグループ）

| ツール名 | 役割 |
|---|---|
| **ETAS ISOLAR-A** | AUTOSAR SWC開発環境 |
| **ETAS ISOLAR-EVE** | 仮想ECU（バイナリなしでテスト可能） |
| **RTA-OS** | AUTOSAR OS |
| **RTA-BSW** | BSWスタック |

**特長：** Bosch系のプロジェクトで多く使われる。仮想ECU環境（ISOLAR-EVE）が充実しており、HIL（Hardware-in-the-Loop）設備が未整備の開発初期段階でも通信シナリオの検証ができる。

### 2-4. ORIENTAIS（東軟グループ・アジア系委託先で多い）

| ツール名 | 役割 |
|---|---|
| **ORIENTAIS Configurator** | BSWコンフィグ＋コード生成 |
| **ORIENTAIS OS** | AUTOSAR OS |

**特長：** コストが比較的低い。中国・アジア系の委託先で使われることが多い。国内実績は少ないため、採用する場合は機能安全対応の実績を特に慎重に確認すること。

---

## 3. 通信仕様からコード生成までの流れ（具体的な操作フロー）

ここでは最も普及しているVector DaVinci Configurator Proを例に、DBCインポートからコード生成までの具体的な手順を示す。委託先の作業内容を把握し、レビュー時の確認ポイントとして活用すること。

### Step 1：DBCのインポートとArXML変換

CANメッセージリスト（DBC）をAUTOSARの通信記述（ArXML）に変換する。

**操作手順（DaVinci Configurator Pro）：**
```
1. DaVinci Configurator Proを起動し、プロジェクトを開く
2. 「File」→「Import」→「CAN DB（.dbc）」を選択
3. HydraulicCommand.dbc を指定してインポート
4. インポート完了後、「Communication」ビューで以下を確認する：
   - IPDUが正しく作成されているか（HydraulicCommand_IPDU 等）
   - シグナルの開始ビット・ビット長・係数・オフセットが一致しているか
   - 送信ECUと受信ECUのマッピングが正しいか
5. 「File」→「Save」でプロジェクトを保存（ArXML形式）
```

**インポート結果として生成されるArXMLの例：**
```xml
<I-PDU>
  <SHORT-NAME>HydraulicCommand_IPDU</SHORT-NAME>
  <LENGTH>8</LENGTH>   <!-- DLC=8バイト -->
</I-PDU>
<I-SIGNAL>
  <SHORT-NAME>BoomLiftCmd</SHORT-NAME>
  <START-POSITION>0</START-POSITION>   <!-- 開始ビット=0 -->
  <LENGTH>16</LENGTH>                  <!-- ビット長=16 -->
  <INIT-VALUE>0</INIT-VALUE>           <!-- 初期値 -->
  <DATA-TRANSFORMATION-REF>           <!-- 係数・オフセットへの参照 -->
    /Package/BoomLiftCmd_Compumethod
  </DATA-TRANSFORMATION-REF>
</I-SIGNAL>
```

**レビュー時の確認ポイント：**
- DBCの開始ビット番号とArXMLのSTART-POSITIONが一致しているか
- Intelバイトオーダーの場合、ArXMLの`BYTE-ORDER`が`OPAQUE`ではなく`LITTLE-ENDIAN`になっているか
- 係数・オフセット（Compumethod）が正しく生成されているか

### Step 2：BSWコンフィグ（COM・CANIF・PDURの設定）

COMモジュール・PDURモジュール・CANIFモジュール・MCALの設定をArXMLで行う。

**DaVinci Configurator Proでの設定手順：**

```
─── COMモジュール設定 ───────────────────────────────────────
1. 「BSW Explorer」→「COM」→「ComConfig」を開く
2. 「ComIPdu」で HydraulicCommand_IPDU を選択
3. 以下のパラメータを設定：
   - ComIPduDirection : SEND（送信側ECUの場合）/ RECEIVE（受信側）
   - ComIPduSignalProcessing : DEFERRED（割り込みではなくタスク周期で処理）
   - ComTxModeMode : PERIODIC（周期送信）
   - ComTxModePeriod : 0.01（10ms = 0.01秒）
4. 「ComSignal」で BoomLiftCmd シグナルを選択し確認：
   - ComBitPosition : 0
   - ComBitSize : 16
   - ComSignalType : UINT16
   - ComSignalInitValue : 0
5. Deadline Monitoring（受信タイムアウト）の設定：
   - ComRxIPduSignalProcessing → Enabled
   - ComFirstTimeout : 50ms（初回タイムアウト）
   - ComTimeout : 50ms（以降のタイムアウト）
   - ComTimeoutNotification : Com_BoomLiftCmd_TimeoutCallback

─── CANIFモジュール設定 ─────────────────────────────────────
1. 「BSW Explorer」→「CanIf」→「CanIfInitCfg」を開く
2. 「CanIfTxPduCfg」で新規PDUを追加：
   - CanIfTxPduCanId : 0x100（CAN ID）
   - CanIfTxPduCanIdType : STANDARD_CAN（11bit）
   - CanIfTxPduRef : HydraulicCommand_IPDU（COMのIPDUへの参照）
3. 物理CANコントローラとのマッピング：
   - CanIfCtrlRef : CanController_0（CAN1に対応するコントローラ）

─── PDURモジュール設定 ──────────────────────────────────────
1. 「BSW Explorer」→「PduR」→「PduRRoutingTables」を開く
2. ルーティングパスを追加：
   送信の場合：COM → CANIF
     PduRSrcPduRef : HydraulicCommand_IPDU（COM側）
     PduRDestPduRef : HydraulicCommand_IPDU（CANIF側）
   ※ DaVinci CPでは多くの場合、DBCインポート時に自動生成される

─── MCAL（CanDriver）設定 ──────────────────────────────────
　 ※ MCALはマイコンベンダーのツール（Renesas Smart Configurator等）で設定することが多い
1. CANコントローラのクロック設定（マイコン依存）
2. ボーレート設定：500kbps / CAN FD 2Mbps
3. 送受信バッファ数の設定
4. 割り込み優先度の設定
   → 設定完了後、CANドライバのCコードをエクスポートしてDaVinci CPプロジェクトへ取り込む
```

**委託先の成果物で確認すべき項目：**
- すべてのメッセージにDeadline Monitoringが設定されているか
- CAN IDとPDU IDのマッピングに漏れがないか
- 送信周期がDBC（`GenMsgCycleTime`属性）と一致しているか

### Step 3：SWCの設計とRTE生成

SWCのポート定義からRTE（Runtime Environment）のCコードを自動生成する。

**DaVinci DeveloperでのSWC定義手順（概要）：**

```
1. DaVinci Developerを起動し、新規SWCを作成（例：HydraulicControlSWC）
2. 「Port」タブでポートを定義：
   - R-Port（受信ポート）: OperatorInput（BoomLiftCmdを受け取るポート）
     DataElement: BoomLiftCmd (uint16)
   - P-Port（送信ポート）: ValveCommand（バルブ指令を送るポート）
     DataElement: BoomLiftCmd (uint16)
3. 「Runnable」タブでRunnableを定義：
   - HydraulicControl_10ms（10ms周期タスクで実行）
   - トリガ：TimingEvent / 10ms
4. ポートとRunnableのアクセス定義を設定
5. 「File」→「Export ArXML」でSWC定義をエクスポート
```

**RTE生成手順（DaVinci Configurator Pro）：**

```
1. SWC定義ArXMLをDaVinci CPプロジェクトへインポート
2. 「Generate」→「RTE」を選択
3. 自動生成されるファイル：
   - Rte.c / Rte.h（RTEのメイン実装）
   - Rte_HydraulicControlSWC.h（SWC固有のAPIヘッダ）
4. 生成後のAPIを確認：
   Rte_Read_OperatorInput_BoomLiftCmd(uint16 *data);   ← 受信API
   Rte_Write_ValveCommand_BoomLiftCmd(uint16 data);    ← 送信API
```

**開発者が実装するRunnable（手書き部分）：**

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
    Std_ReturnType status;

    /* RTEのAPIでシグナルを受信（CANの詳細は意識しない）*/
    status = Rte_Read_OperatorInput_BoomLiftCmd(&operatorInput);

    if (status == RTE_E_COM_STOPPED) {
        /* COM Deadline Monitoring でタイムアウト検出 → フェールセーフ */
        valveCmd = FAILSAFE_VALUE;
    } else {
        /* ここにアプリケーションロジックを実装 */
        valveCmd = ConvertToValveCommand(operatorInput);
    }

    /* RTEのAPIでシグナルを送信（CANへの送出はRTE/COMが担当）*/
    Rte_Write_ValveCommand_BoomLiftCmd(valveCmd);
}
```

**重要：** RTE・BSWのコードは自動生成。開発者が書くのはSWC内部のアプリケーションロジックのみ。開発者が自動生成コードを手修正した場合、次回のコード生成で上書きされて消える。手修正が必要な場合はArXMLを修正して再生成すること。

### Step 4：静的解析・MISRA-Cチェック

```
対象コード：
  ・自動生成コード（Rte.c, Com_cfg.c 等）
  ・手書きSWCコード（HydraulicControlSWC.c 等）

使用ツール（委託先のいずれかが使うことを確認）：
  ・Polyspace Bug Finder / Code Prover（MathWorks）
  ・Helix QAC（Perforce）
  ・PC-lint Plus（Gimpel）

MISRA-C:2012の代表的なルール（委託先のレポートで確認するもの）：
  ・Rule 15.5：1つの関数に return文は1つのみ
  ・Rule 10.1：演算時の暗黙的型変換を禁止
  ・Rule 14.3：制御式が定数にならないこと（デッドコードの禁止）
  ・Advisory ルールとMandatoryルールの違いを理解する
    → Mandatory違反は原則として修正必須
    → Advisory違反は逸脱理由書（Deviation）の提出で対応可能

レポートの見方：
  ・「Findings Summary」で違反件数とカテゴリを確認
  ・Mandatory違反が0件であることを確認
  ・Advisory逸脱件数が多い場合は理由が妥当かレビューすること
```

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

**Adaptive AUTOSARのコード例（サービス消費側）：**
```cpp
// PersonDetectionServiceのプロキシ（自動生成）を使ったアプリのイメージ
#include "ara/com/sample/person_detection_proxy.h"

void PersonDetectionApp::Init() {
    // サービスを発見・接続
    auto handles = ara::com::sample::PersonDetectionProxy::FindService(
        ara::com::InstanceIdentifier("0x0001"));

    if (!handles.empty()) {
        proxy_ = std::make_unique<PersonDetectionProxy>(handles[0]);

        // イベントの購読（Classic AUTOSARのSDと同等）
        proxy_->PersonDetected.Subscribe(10);  // 最大10件キューイング
        proxy_->PersonDetected.SetReceiveHandler(
            [this]() { OnPersonDetected(); });
    }
}

void PersonDetectionApp::OnPersonDetected() {
    auto samples = proxy_->PersonDetected.GetNewSamples();
    for (auto& sample : samples) {
        if (sample->DetectionStatus >= 1) {
            TriggerSafetyAlarm(sample->DetectedArea);
        }
    }
}
```

### Adaptive AUTOSARの現状

自動車業界でもAdaptive Platform（AP）の実用化は道半ばである。以下の課題が指摘されている。

- **ツール・ノウハウが未成熟：** Classic AUTOSARと比較して対応ベンダー・事例が少ない
- **機能安全対応が難しい：** Linux上での機能安全認証は複雑であり、ASIL-Cを超えるレベルへの対応は限定的
- **規格改訂が速い：** R22-11・R23-11等のリリースが続いており、ツールバージョンの追従が必要

**現実的な判断：** Linux系（HMI・人検知・クラウド通信）は機能安全要件が低い場合が多いため、Adaptive AUTOSARにこだわらず「Linuxベースのミドルウェア（非AUTOSAR）」で構築するアプローチも十分合理的。SOME/IPはAdaptive AP非採用でも使用できる。

---

## 5. 自社担当者が最低限把握すべきこと

外部委託で開発してもらう場合でも、以下は自社側で理解しておく必要がある。

| 理解レベル | 内容 | 具体的な習得方法 |
|---|---|---|
| **必須** | ArXMLの構造（何がどこに定義されているか読める・差分を確認できる） | PoC期間中に委託先に成果物のウォークスルーを依頼する。Notepad++のXPath検索で特定のSHORT-NAMEを検索する練習をすること |
| **必須** | SWC・ポート・Runnableの概念 | AUTOSAR公式のIntroductory Overviewドキュメント（autosar.org）を読む。委託先にサンプルSWCを1つ作ってもらい、RteのAPIが生成されるまでを見学する |
| **必須** | 通信設定（CANIF・COM・PDUR）の役割と設定項目 | 本ドキュメントとautosar-modules.mdを読んだうえで、委託先にDaVinci CPの画面を共有してもらい、COM設定画面を一緒に確認する |
| **必須** | CANoeまたは代替ツールを使ったCAN通信の確認方法 | VectorのCANoe Demoライセンス（無償・機能制限あり）で操作を試す。DBCを読み込んでシグナルモニターを表示する操作を習得する |
| **必須** | J1939のPGN・SPN・SAの基本概念（建設機械での外部I/Fに必須） | SAE J1939-21・J1939-71の要点を委託先に解説してもらう（1〜2時間のレクチャーを発注仕様に含める）。自社で保有するエンジンECUのマニュアルのPGN一覧と照合して理解を深める |
| **推奨** | DaVinci Configurator Proの基本操作 | 委託先にデモ環境を借りてDBCインポート→COM設定→コード生成を自分で1回実施する。Vectorの有償トレーニング（1〜2日コース）も有効 |
| **推奨** | 静的解析ツール（Polyspace等）のレポートの読み方 | 委託先から納品されたPolyspaceレポートを開き、Findings Summaryの読み方を委託先に説明してもらう。Mandatory違反件数0を納品条件にして、レポートを確認するフローを確立する |
| **任意** | コード生成の仕組みの詳細 | 自動生成コードの全体を把握する必要はないが、Rte.hのAPIシグネチャとSWCの実装コードの対応を読めるようにすること |

**習得の優先順位と時間感：**
```
PoC開始前（0〜1ヶ月）：
  → 本ドキュメントとautosar-modules.mdを熟読（2〜3時間）
  → AUTOSARの入門書1冊（例：「AUTOSARによる次世代ECUソフトウェア設計」）を読む

PoC開始後（1〜3ヶ月）：
  → 委託先にウォークスルーを依頼（週1回1時間のレビューセッションを設ける）
  → CANoe Demoで操作練習（半日）
  → J1939レクチャーを受ける（2時間）

量産設計フェーズ前（3〜6ヶ月後）：
  → Vectorトレーニングまたは委託先トレーニングに参加（1〜2日）
  → Polyspaceレポートを自分でレビューできる状態にする
```

**習得をPoC契約に含める文言の例：**

```
【発注仕様への記載例】
・成果物（ArXML・コード）のレビュー方法を発注者担当者に対してレクチャーすること
  （PoC期間中に最低2回、各2時間以上のセッションを設けること）
・使用ツール（DaVinci Configurator Pro等）のデモ操作環境を
  発注者担当者が閲覧できる状態にすること
・J1939 PGN/SPN定義について、本プロジェクト固有の実装部分を
  1時間以上かけて説明すること
```

**ArXMLのレビュー環境：** 自社ではビューアーレベルのツールで十分。DaVinciの無償Viewer、またはNotepad++のXPath検索機能を活用する。数千行のArXMLをテキストで全部読む必要はない。以下の検索パターンを覚えるだけで主要な確認が可能：

```
確認したい内容 → 検索キーワードの例
──────────────────────────────────────────────────────────
COMのシグナル一覧    → "ComSignal"
タイムアウト設定     → "ComTimeout"
CANIF の CAN ID設定  → "CanIfTxPduCanId"
PDURのルーティング   → "PduRRoutingPath"
SWCのポート定義      → "P-PORT-PROTOTYPE" / "R-PORT-PROTOTYPE"
Runnableの周期設定   → "TimingEvent"
```

---

## 6. ツールライセンスの費用感

自社でツールを購入する必要はない。委託先がツールを保有していることを選定条件にし、ライセンスコストを委託先に持たせること。自社に必要なのは成果物（ArXML・ソースコード）を確認するためのビューアーレベルのツールで十分。

| ツール | 費用感 | 推奨 | 自社に必要か |
|---|---|---|---|
| Vector DaVinci Configurator Pro | 高コスト（年間ライセンス） | 委託先が保有 | 不要（ビューアーで十分） |
| Vector CANoe | 高コスト（モジュール構成で変動） | 委託先が保有。受け入れ検証でのみ要確認 | Demoライセンスで操作確認は可能 |
| EB tresos Studio | 中〜高コスト | 委託先が保有 | 不要 |
| ETAS ISOLAR | 中コスト | 委託先が保有 | 不要 |
| 静的解析ツール（Polyspace等） | 中〜高コスト | 委託先が実施・レポート納品 | 不要（レポートのPDF確認のみ） |

**委託先選定時のツール確認チェックリスト：**
```
□ DaVinci Configurator Pro（または同等ツール）の正規ライセンスを保有しているか
□ CANoeの正規ライセンスを保有しているか（モジュール構成も確認）
□ MISRAチェックツールの正規ライセンスを保有し、機能安全対応実績があるか
□ 使用ツールのバージョンが最新または1世代前以内か
  （古いバージョンのArXMLはインポートできない場合がある）
□ Adaptive AUTOSAR対応が必要な場合、MICROSAR.APまたはRTA-APの実績があるか
```
