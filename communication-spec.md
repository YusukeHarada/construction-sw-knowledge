# 通信仕様（CAN・Ethernet メッセージリスト）の書き方

**作成日：** 2026年6月16日

---

## 概要

AUTOSARでは通信仕様はArXMLとして定義されるが、その入力として人間が読める形式の通信仕様書が必要である。本ドキュメントでは、AUTOSARのツールチェーンへのインプットとなる通信仕様の書き方を定義する。

---

## 1. 通信仕様書の全体構成

```
通信仕様書
├── ネットワーク構成図
├── CANメッセージリスト（バス別）
│   ├── メッセージ定義
│   └── シグナル定義
├── Ethernetサービスリスト
│   ├── SOAPサービス定義（SOME/IP）
│   └── イベント定義
└── ネットワーク管理仕様
    ├── NM（Network Management）設定
    └── バスオフ・スリープ制御
```

---

## 2. CAN通信仕様

### 2-1. ネットワーク構成

まず各CANバスの役割・接続ECUを定義する。

| バスID | バス名 | 通信速度 | 接続ECU | 用途 |
|---|---|---|---|---|
| CAN1 | ControlBus | 500kbps | VehicleCtrlECU, HydraulicECU | 制御系 |
| CAN2 | BodyBus | 250kbps | BodyECU, DisplayECU | ボディ系 |
| CAN3 | DiagBus | 500kbps | All ECU | 診断 |

### 2-2. メッセージリスト

```
フォーマット定義：
- Message ID：CAN ID（標準11bit or 拡張29bit）
- DLC：データ長（バイト）
- 送信ECU：送信ノード
- 送信周期：定期送信の場合は周期（ms）、イベント送信は「Event」
- バス：どのCANバスを流れるか
```

| Message ID | メッセージ名 | DLC | 送信ECU | 送信周期 | バス |
|---|---|---|---|---|---|
| 0x100 | HydraulicCommand | 8 | VehicleCtrlECU | 10ms | CAN1 |
| 0x101 | HydraulicStatus | 8 | HydraulicECU | 10ms | CAN1 |
| 0x200 | OperatorInput | 8 | BodyECU | 20ms | CAN1 |
| 0x300 | SystemStatus | 4 | VehicleCtrlECU | 100ms | CAN1 |
| 0x7DF | DiagRequest | 8 | DiagTool | Event | CAN3 |

### 2-3. シグナル定義

メッセージの中のビット割り付けを定義する。

**例：HydraulicCommand（0x100）**

| シグナル名 | 開始ビット | ビット長 | バイトオーダー | 型 | 係数 | オフセット | 最小値 | 最大値 | 単位 | 初期値 |
|---|---|---|---|---|---|---|---|---|---|---|
| BoomLiftCmd | 0 | 16 | Motorola | uint16 | 0.1 | -100.0 | -100.0 | 100.0 | % | 0 |
| ArmLiftCmd | 16 | 16 | Motorola | uint16 | 0.1 | -100.0 | -100.0 | 100.0 | % | 0 |
| BucketCmd | 32 | 16 | Motorola | uint16 | 0.1 | -100.0 | -100.0 | 100.0 | % | 0 |
| ControlMode | 48 | 4 | Motorola | uint8 | 1 | 0 | 0 | 15 | - | 0 |
| SafetyFlag | 52 | 2 | Motorola | uint8 | 1 | 0 | 0 | 3 | - | 0 |
| Reserved | 54 | 10 | - | - | - | - | - | - | - | 0 |

**値の定義（Enum型シグナル）**

```
ControlMode:
  0x0 = Manual
  0x1 = Assisted
  0x2 = Auto
  0xF = Invalid

SafetyFlag:
  0x0 = Normal
  0x1 = Warning
  0x2 = Fault
  0x3 = Reserved
```

### 2-4. DBC形式での記述（ツール入力用）

CANdb++・CANoe・Vector DaVinci等のツールへの入力として、DBCファイル形式でも定義する。

```dbc
VERSION ""

NS_ :

BS_:

BU_: VehicleCtrlECU HydraulicECU BodyECU

BO_ 256 HydraulicCommand: 8 VehicleCtrlECU
 SG_ BoomLiftCmd : 0|16@1+ (0.1,-100) [-100|100] "%" HydraulicECU
 SG_ ArmLiftCmd : 16|16@1+ (0.1,-100) [-100|100] "%" HydraulicECU
 SG_ BucketCmd : 32|16@1+ (0.1,-100) [-100|100] "%" HydraulicECU
 SG_ ControlMode : 48|4@1+ (1,0) [0|15] "" HydraulicECU
 SG_ SafetyFlag : 52|2@1+ (1,0) [0|3] "" HydraulicECU

BO_ 257 HydraulicStatus: 8 HydraulicECU
 SG_ BoomActualPos : 0|16@1+ (0.1,-100) [-100|100] "%" VehicleCtrlECU
 SG_ ArmActualPos : 16|16@1+ (0.1,-100) [-100|100] "%" VehicleCtrlECU
 SG_ HydraulicTemp : 32|8@1+ (1,-40) [-40|215] "degC" VehicleCtrlECU
 SG_ HydraulicPressure : 40|16@1+ (0.1,0) [0|6553] "kPa" VehicleCtrlECU

BA_DEF_ BO_ "GenMsgCycleTime" INT 0 10000;
BA_ "GenMsgCycleTime" BO_ 256 10;
BA_ "GenMsgCycleTime" BO_ 257 10;
```

---

## 3. Ethernet（SOME/IP）通信仕様

### 3-1. サービスリスト

| Service ID | サービス名 | 提供ECU | プロトコル | 用途 |
|---|---|---|---|---|
| 0x0100 | PersonDetectionService | LinuxECU | UDP | 人検知結果の配信 |
| 0x0101 | CloudTelemetryService | LinuxECU | TCP | クラウドへのデータ送信 |
| 0x0200 | HMIControlService | HmiECU | TCP | HMI制御コマンド |

### 3-2. サービス詳細定義

**例：PersonDetectionService（0x0100）**

```
Service ID : 0x0100
Instance ID: 0x0001
Major Version: 1
Minor Version: 0
プロトコル  : UDP Multicast
ポート番号  : 30490

イベントグループ：
  EventGroup ID: 0x0001
  イベント一覧:
    Event ID: 0x8001
    イベント名: PersonDetected
    送信条件: 検知状態変化時（Event）+ 1000ms周期
    ペイロード:
      - DetectionStatus (uint8): 0=未検知, 1=検知, 2=高信頼度検知
      - DetectedArea (uint8): 検知エリアID（0〜7）
      - Confidence (uint8): 信頼度 0〜100%
      - Timestamp (uint32): 検知時刻（ms）
```

### 3-3. ARXML用サービス定義（SDインスタンス）

```xml
<!-- AUTOSAR SOME/IPの設定例（委託先へのインプットイメージ） -->
<SOMEIP-SERVICE-INSTANCE-TO-MACHINE-MAPPING>
  <SERVICE-INSTANCE-IREF>
    <SERVICE-INTERFACE-ID>PersonDetectionService</SERVICE-INTERFACE-ID>
    <INSTANCE-IDENTIFIER>1</INSTANCE-IDENTIFIER>
  </SERVICE-INSTANCE-IREF>
  <MAJOR-VERSION>1</MAJOR-VERSION>
  <MINOR-VERSION>0</MINOR-VERSION>
  <UDP-PORT>30490</UDP-PORT>
  <MULTICAST-ADDRESS>239.0.0.1</MULTICAST-ADDRESS>
</SOMEIP-SERVICE-INSTANCE-TO-MACHINE-MAPPING>
```

---

## 4. J1939固有の注意事項

建設機械ではCANの上位プロトコルとしてJ1939が使われることが多い。J1939を使う場合は以下の定義が追加で必要になる。

| 定義項目 | 内容 |
|---|---|
| PGN（Parameter Group Number） | メッセージの種類を示す識別子 |
| SPN（Suspect Parameter Number） | シグナルの識別子（J1939の標準シグナル番号） |
| SA（Source Address） | 送信ノードのアドレス（0x00〜0xFE） |
| DA（Destination Address） | 宛先アドレス（P2P通信の場合） |

標準PGN・SPNはJ1939-71等の仕様書で定義されており、自社独自拡張（Proprietary A/B PGN）を使う場合はその旨を明記する。

---

## 5. 通信仕様書作成・管理のルール

### ツール管理を推奨する理由

通信仕様をExcelで管理すると、以下の問題が起きやすい。

- ビット割り付けのミスが目視では発見しにくい
- DBCやArXMLへの変換ミスが発生する
- 複数人が編集すると整合性が崩れる

**推奨：** DBCまたはArXMLを一次ソースとし、Excelは閲覧・共有用の派生物として扱う。

### バージョン管理

- 通信仕様はgitで管理する
- メッセージ・シグナルの追加・変更は必ずコミットログに理由を記載する
- バス全体のバランス（バス負荷率）を変更のたびに確認する

**バス負荷率の目安：**
- CAN：60%以下を維持（それ以上は遅延・エラーフレームのリスク）
- Ethernet：物理的には余裕があるが、SOME/IP SDの帯域消費に注意
