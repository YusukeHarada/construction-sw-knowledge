# 通信仕様（CAN・Ethernet メッセージリスト）の書き方

**作成日：** 2026年6月16日  
**対象読者：** 自社技術担当者・委託先  
**資料種別：** 社内運用資料（社外秘）

---

## 概要

AUTOSARでは通信仕様はArXMLとして定義されるが、その入力として人間が読める形式の通信仕様書が必要である。本ドキュメントでは、AUTOSARのツールチェーンへのインプットとなる通信仕様の書き方を定義する。

---

## 1. 通信仕様書の全体構成

```
通信仕様書
├── ネットワーク構成
│   ├── CANバス構成（バスID・速度・接続ECU・バス用途）
│   └── Ethernetネットワーク構成（VLAN・IPアドレス体系・スイッチ有無）
├── CANメッセージリスト（バス別）
│   ├── メッセージ定義
│   ├── シグナル定義
│   └── CAN ID割り当てルール
├── J1939メッセージリスト（J1939バス使用時）
│   ├── PGN/SPN定義
│   └── J1939固有事項
├── Ethernetサービスリスト
│   ├── SOAPサービス定義（SOME/IP）
│   └── イベント定義
└── ネットワーク管理仕様
    ├── CANnm設定（ノードID・タイムアウト）
    └── バスオフ検出・回復動作
```

---

## 2. ネットワーク構成

### 2-1. CANバス構成

| バスID | バス名 | 通信速度 | プロトコル | 接続ECU | 用途 |
|---|---|---|---|---|---|
| CAN1 | ControlBus | 500kbps / CAN FD 2Mbps | AUTOSAR COM | VehicleCtrlECU, HydraulicECU | 制御系 |
| CAN2 | BodyBus | 250kbps | J1939 | BodyECU, EngineECU | エンジン・外部機器 |
| CAN3 | DiagBus | 500kbps | ISO 15765（UDS） | All ECU | 診断 |

**CAN ID割り当てルール：**
- J1939バス（CAN2）：29bit拡張ID固定（J1939規格に従う）
- 社内制御バス（CAN1）：11bit標準ID使用
- ID割り当て表は通信仕様担当者が管理し、重複・優先度競合がないことを確認すること

### 2-2. Ethernetネットワーク構成

| 項目 | 設定値 |
|---|---|
| 物理層 | 100BASE-T1（車載Ethernet）またはGigabit Ethernet |
| IPアドレス体系 | 192.168.X.X（要確定） |
| VLAN | VLAN ID：制御系=10、HMI系=20、クラウド通信=30（要確定） |
| スイッチ | 有無・型番（要確定） |

---

## 3. CAN通信仕様

### 3-1. メッセージリスト

| Message ID | メッセージ名 | DLC | 送信ECU | 送信周期 | バス |
|---|---|---|---|---|---|
| 0x100 | HydraulicCommand | 8 | VehicleCtrlECU | 10ms | CAN1 |
| 0x101 | HydraulicStatus | 8 | HydraulicECU | 10ms | CAN1 |
| 0x200 | OperatorInput | 8 | BodyECU | 20ms | CAN1 |
| 0x300 | SystemStatus | 4 | VehicleCtrlECU | 100ms | CAN1 |
| 0x7DF | DiagRequest | 8 | DiagTool | Event | CAN3 |

### 3-2. シグナル定義

**例：HydraulicCommand（0x100）**

バイトオーダーは**Intel（リトルエンディアン、@1）**に統一する。  
（Motorola＝ビッグエンディアンの場合は @0。DBCとテーブルで必ず一致させること）

| シグナル名 | 開始ビット | ビット長 | バイトオーダー | 型 | 係数 | オフセット | 最小値 | 最大値 | 単位 | 初期値 | タイムアウト | タイムアウト時フェールセーフ値 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| BoomLiftCmd | 0 | 16 | Intel | uint16 | 0.1 | -100.0 | -100.0 | 100.0 | % | 0 | 50ms | 0（安全値は機構設計確認要） |
| ArmLiftCmd | 16 | 16 | Intel | uint16 | 0.1 | -100.0 | -100.0 | 100.0 | % | 0 | 50ms | 0 |
| BucketCmd | 32 | 16 | Intel | uint16 | 0.1 | -100.0 | -100.0 | 100.0 | % | 0 | 50ms | 0 |
| ControlMode | 48 | 4 | Intel | uint8 | 1 | 0 | 0 | 15 | - | 0 | - | - |
| SafetyFlag | 52 | 2 | Intel | uint8 | 1 | 0 | 0 | 3 | - | 0 | - | - |
| Reserved | 54 | 10 | - | - | - | - | - | - | - | 0 | - | - |

**値の定義（Enum型シグナル）**

```
ControlMode:
  0x0 = Manual
  0x1 = Assisted
  0x2 = Auto
  0xF = Invalid（受信側はこの値を無効として扱い、フェールセーフ動作に移行する）

SafetyFlag:
  0x0 = Normal
  0x1 = Warning
  0x2 = Fault
  0x3 = Reserved
```

**タイムアウト・無効値について：**  
CAN受信タイムアウト（メッセージが一定時間来ない）を検出した場合の動作を必ず定義すること。制御側が「値が来ていない」と判断する方法を定義しないと、委託先が独自に実装してしまうリスクがある。

### 3-3. DBC形式での記述（ツール入力用）

**プロジェクト初期はDBCを一次ソースとし、AUTOSARツールへインポート後はArXMLをmasterとして管理する。DBCはArXMLから逆生成で同期させること。**

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

**注：** `@1` はIntelバイトオーダー（リトルエンディアン）、`@0` はMotorabaバイトオーダー（ビッグエンディアン）を示す。テーブルのバイトオーダー定義と必ず一致させること。

---

## 4. J1939通信仕様

### 4-1. 概要

建設機械では外部エンジンECU・アタッチメント機器との通信にJ1939が使われることが多い。J1939は29bit拡張IDを用い、PGN（Parameter Group Number）でメッセージ種別を、SPN（Suspect Parameter Number）でシグナルを識別する。

### 4-2. J1939メッセージリスト

| PGN | メッセージ名 | SA（送信ノードアドレス） | DA | 送信周期 | 用途 |
|---|---|---|---|---|---|
| 0xF004 (EEC1) | エンジン制御1 | EngineECU (0x00) | 全局 | 10ms | エンジン回転数・トルク |
| 0xFEF1 (CCVS) | 車速 | BodyECU (0x21) | 全局 | 100ms | 車速 |
| 0xFECA (DM1) | アクティブ診断コード | 各ECU | 全局 | Event + 1s | 故障コード通知 |

### 4-3. J1939 SPNリスト（例：EEC1）

| SPN | シグナル名 | 開始ビット | ビット長 | 係数 | オフセット | 単位 |
|---|---|---|---|---|---|---|
| 190 | EngineSpeed | 24 | 16 | 0.125 | 0 | rpm |
| 91 | AcceleratorPedalPosition | 8 | 8 | 0.4 | 0 | % |
| 92 | EnginePercentTorque | 16 | 8 | 1 | -125 | % |

### 4-4. J1939固有の注意事項

- **アドレスクレーム：** J1939ネットワーク上の各ノードはSA（Source Address）を動的に取得するアドレスクレーム処理が必要。AUTOSARの場合、J1939Nm（J1939 Network Management）モジュールが担当する
- **診断（DM1/DM2）：** 建設機械の外部診断ツールはJ1939 DM1（アクティブ診断コード）を前提とすることが多い。UDSのみでは対応できないケースがあるため、J1939診断が必要か事前に確認すること
- **ゲートウェイ処理：** エンジンECU等の外部サプライヤ製ECUとのJ1939通信は、ゲートウェイSWCでAUTOSARシグナルに変換して内部に取り込む構成にすること

---

## 5. Ethernet（SOME/IP）通信仕様

### 5-1. サービスリスト

| Service ID | サービス名 | 提供ECU | AUTOSAR種別 | プロトコル | 用途 |
|---|---|---|---|---|---|
| 0x0100 | PersonDetectionService | LinuxECU | Adaptive | UDP Multicast | 人検知結果の配信 |
| 0x0101 | CloudTelemetryService | LinuxECU | Adaptive | TCP | クラウドへのデータ送信 |
| 0x0200 | HMIControlService | HmiECU | Adaptive | TCP | HMI制御コマンド |

**注：** SOME/IPはClassic AUTOSARとAdaptive AUTOSARで実装方法が異なる。Classic側でSOME/IPを受信する場合（SomeIpTransformer経由）の仕様も別途定義すること。

### 5-2. サービス詳細定義

**例：PersonDetectionService（0x0100）**

```
Service ID         : 0x0100
Instance ID        : 0x0001
Major Version      : 1
Minor Version      : 0
プロトコル         : UDP Multicast
SD ポート          : 30490（SOME/IP-SD 標準ポート）
サービスデータポート: 50100（別途割り当て）
マルチキャストアドレス: 239.0.0.1

イベントグループ：
  EventGroup ID: 0x0001
  イベント一覧:
    Event ID: 0x8001
    イベント名: PersonDetected
    送信条件: 検知状態変化時（Event）+ 1000ms周期
    最大レイテンシ要件: 検知から通知まで最大XX ms（安全要求側と合意すること）
      ※1000ms周期では安全レイテンシ要件を満たさない可能性があるため必ず確認
    ペイロード:
      - DetectionStatus (uint8): 0=未検知, 1=検知, 2=高信頼度検知
      - DetectedArea (uint8): 検知エリアID（0〜7）
      - Confidence (uint8): 信頼度 0〜100%
      - Timestamp (uint32): 検知時刻（ms）
```

**注：** ArXMLの具体的な構造はツールチェーンのバージョンによって異なる。上記はイメージ例であり、正確な構造は委託先に確認すること。

---

## 6. ネットワーク管理（NM）仕様

### 6-1. CAN Network Management（CANnm）

| 設定項目 | 設定値 |
|---|---|
| ノードID | ECUごとに割り当て（0x01〜0xFE、要確定） |
| NMタイムアウト | XX ms（要確定） |
| バスオフ検出方式 | AUTOSARネイティブ（バスオフカウンタ方式） |
| バスオフ後の自動回復 | 最大3回、回復間隔：100ms |
| 3回回復後も復帰しない場合 | 上位レイヤー（システムマネージャ）へ通知しフェールセーフへ移行 |

### 6-2. スリープ・ウェイクアップ制御

建設機械ではエンジン停止後のECUスリープと、キースイッチONによるウェイクアップ処理が必要。

```
スリープシーケンス：
  エンジン停止信号受信 → XX秒後にNMスリープ要求送出 → 全ECU応答確認 → スリープ

ウェイクアップシーケンス：
  キースイッチON（またはWake-on-CAN） → NMウェイクアップ → 全ECU起動確認 → 通常動作
```

---

## 7. 通信仕様書作成・管理のルール

### ツール管理

通信仕様をExcelのみで管理すると、ビット割り付けミス・DBC変換ミス・整合性崩壊が起きやすい。

**推奨管理方法：**
- プロジェクト初期：DBCを一次ソースとして作成し、ツールへインポート
- AUTOSARツール導入後：ArXMLをmasterとし、DBCは逆生成で同期
- Excelは閲覧・共有用の派生物として扱う

### バージョン管理

- 通信仕様（DBC・ArXML）はgitで管理する
- メッセージ・シグナルの追加・変更はコミットログに理由を記載する
- 変更の承認者（自社担当者名）を決め、委託先への通知フローを確立する
- 量産に近づく時期（XX週前）以降の変更はCCB（変更管理委員会）の承認を要するルールを設ける

### バス負荷率の確認

変更のたびにバス負荷率を確認する。

```
バス負荷率 ≈ Σ（フレームビット長 × 送信頻度） ÷ バス帯域[bps]

目安：
  CAN（500kbps）：60%以下を維持
  CAN FD（2Mbps）：70%以下を目安
  Ethernet：物理帯域に余裕があっても SOME/IP SDの帯域消費・遅延に注意

確認ツール：Vector CANoeのBus Statistics機能が最も手軽
```
