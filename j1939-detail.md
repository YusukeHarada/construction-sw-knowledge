# J1939プロトコル技術詳解

建設機械向けJ1939プロトコルのPGN・SPN定義、DBCファイル作成、AUTOSARとの連携を技術担当者・委託先向けに解説する。仕様書の書き方は `communication-spec.md` を参照。本資料はJ1939プロトコル自体の技術解説を目的とする。

---

## 1. J1939概要

### OSI参照モデルでの位置付け

| OSI層 | 機能 | J1939対応仕様 |
|---|---|---|
| 物理層（L1） | 電気的特性・コネクタ | ISO 11898-2（CAN High Speed） |
| データリンク層（L2） | フレーム構造・アービトレーション | ISO 11898-1（CAN仕様）＋J1939-21 |
| ネットワーク層（L3） | アドレス管理・トランスポート | J1939-21（TP）・J1939-81（NAM） |
| アプリケーション層（L7） | PGN・SPN定義 | J1939-71（Vehicle Application Layer） |

J1939は**SAE（Society of Automotive Engineers）**が策定した商用車・農業機械・建設機械向けのCAN上位層プロトコルである。物理層はISO 11898-2（250 kbps標準、500 kbps・1 Mbps対応機器も存在）を使用する。

### 建設機械での使われ方

- **ISOBUS（ISO 11783）**：農業機械向けでJ1939をベースにトラクタ＋作業機の接続を定義。建設機械のアタッチメント連携にも転用される場合がある。
- **J1939-75（Application Layer for Earthmoving Machines）**：掘削機・ブルドーザ・クレーン等向けのPGN拡張定義。作業機圧力・バケット角度・積載量等の専用SPNが規定されている。
- エンジン・トランスミッション・ポンプ・各種コントローラ間の通信バスとして使用。1ネットワークに最大30ノードを接続可能（物理的制約による）。

---

## 2. フレーム構造

### 29bit拡張CANフレーム

J1939は必ず**29bitの拡張ID**（CAN 2.0B）を使用する。28bit目から0bit目の割り当ては以下の通り。

```
 28    26 25 24 23        16 15         8 7          0
┌────────┬──┬──┬───────────┬─────────────┬────────────┐
│Priority│ R│DP│ PDU Format │ PDU Specific│Source Addr │
│ 3bit   │1b│1b│   8bit     │    8bit     │   8bit     │
└────────┴──┴──┴───────────┴─────────────┴────────────┘
```

| フィールド | ビット数 | 説明 |
|---|---|---|
| Priority | 3bit | 送信優先度（0が最高、7が最低）。制御系は3〜5が多い |
| Reserved（R） | 1bit | 将来拡張用。通常0 |
| Data Page（DP） | 1bit | PGNページ拡張。0＝ページ0（標準）、1＝ページ1（拡張） |
| PDU Format（PF） | 8bit | PDUタイプ判定に使用（0x00〜0xEF：PDU1、0xF0〜0xFF：PDU2） |
| PDU Specific（PS） | 8bit | PDU1では宛先アドレス、PDU2ではグループ拡張 |
| Source Address（SA） | 8bit | 送信元アドレス（0x00〜0xFD：通常使用、0xFE：Anonymous、0xFF：Global） |

### PGNの計算方法

PGN = （DP × 65536）＋（PF × 256）＋（PDU2の場合はPS、PDU1の場合は0）

**PDU1（PF ≤ 0xEF）**：PSは宛先アドレスのため、PGNにはPS=0x00を代入して計算する。特定アドレス宛ての送信が可能。

**PDU2（PF ≥ 0xF0）**：PSはGroup Extension（GE）として扱い、PGNに含まれる。ブロードキャスト専用。

### 具体的なCANフレーム例

エンジン回転数（EEC1、PGN 0xF004 = 61444）の送信例：

```
29bit ID（16進）: 0CF00400
内訳：
  Priority    = 3  (0b011)
  R           = 0
  DP          = 0
  PDU Format  = 0xF0 (240) → PDU2
  PDU Specific= 0x04 (GE=4)
  Source Addr = 0x00 (ECM)

PGN = 0xF004 = 61444 (EEC1)
データフィールド（8バイト）:
  Byte 0: Engine Control Mode（SPN 899）
  Byte 1: Driver Demand Engine - Percent Torque（SPN 512）
  Byte 2: Actual Engine - Percent Torque（SPN 513）
  Byte 3-4: Engine Speed（SPN 190）LSB先（リトルエンディアン）
  Byte 5: Source Address of Controlling Device（SPN 1483）
  Byte 6: Engine Starter Mode（SPN 1675）
  Byte 7: Engine Demand Engine - Percent Torque（SPN 2432）
```

---

## 3. PGN（Parameter Group Number）

### 代表的なPGN一覧

| PGN（10進） | 略称 | 説明 | 送信周期（目安） |
|---|---|---|---|
| 61444 | EEC1 | Electronic Engine Controller 1（エンジン回転数・トルク） | 10ms |
| 61445 | EEC2 | Electronic Engine Controller 2（アクセル位置等） | 50ms |
| 65265 | CCVS | Cruise Control / Vehicle Speed（車速・クルーズ状態） | 100ms |
| 61442 | ETC1 | Electronic Transmission Controller 1（シフト位置等） | 10ms |
| 65257 | LFC | Fuel Consumption（燃料消費量） | 1000ms |
| 65262 | ET1 | Engine Temperature 1（冷却水温・燃料温度） | 1000ms |
| 65263 | EFL | Engine Fluid Level（油圧・油温） | 1000ms |
| 65110 | — | J1939-75：掘削機アーム角度等（要確定） | 要確定 |

### 建設機械固有PGNの例（J1939-75）

- 作業機シリンダ圧力（SPN 要確定）：バケットシリンダ・アームシリンダの圧力値
- 機体傾斜角：前後・左右の傾き（転倒防止制御に使用）
- 積載量：バケット内土砂のペイロード推定値

カスタムPGNを定義する場合は、PGN 0xFF00〜0xFFFF（Proprietary B）またはPDU1の相手先指定形式（Proprietary A）を使用する。

### データのバイト順・スケーリング

J1939のデータ表現ルール：
- **バイト順**：1バイト目がLSB（リトルエンディアン）が基本。ただしSPN定義書に従うこと。
- **スケーリング**：`実値 = 生データ × Resolution + Offset`
- **無効値**：0xFE = エラー表示、0xFF（1バイト）または 0xFFFF（2バイト）= 未実装

**エンジン回転数（SPN 190）変換例：**
```
Resolution  = 0.125 rpm/bit
Offset      = 0 rpm
Range       = 0 〜 8031.875 rpm
データ値 0x1900 (LSB=0x19=25, MSB=0x00=0) → 生データ = 25
実値 = 25 × 0.125 = 3.125 rpm  ← 実際の生データは要確定
```

---

## 4. SPN（Suspect Parameter Number）

### SPNとPGNの関係

1つのPGNは固定長8バイトのデータフィールドを持ち、その中に複数のSPNが詰め込まれる。SPNは個々のパラメータ定義を指す。

```
PGN 61444 (EEC1)
├── SPN 899  : Engine Control Mode          (Byte 0, bit 0-3)
├── SPN 512  : Driver Demand Engine Torque  (Byte 1, 全8bit)
├── SPN 513  : Actual Engine Torque         (Byte 2, 全8bit)
├── SPN 190  : Engine Speed                 (Byte 3-4, 全16bit)
└── ...
```

### SPNデータタイプ

| タイプ | 説明 | 例 |
|---|---|---|
| Measured | 測定値（数値）。ResolutionとOffsetで実値変換 | 回転数・温度・圧力 |
| Status | 状態値（列挙）。固定値が状態を表す | シフト位置・オン/オフ |
| Counter | カウンタ（累積値）。エラー回数・燃料消費量合計 | トリップ燃費 |
| Textual | テキスト表示用（通常8bit、ASCII） | 警告メッセージ等 |

---

## 5. DBCファイル

### DBCファイルの基本構造

DBC（Database CAN）はVector社が策定したCANネットワーク記述フォーマット。CANoe・CANdb++・CANalyzer等のツールで標準的に使用される。

```dbc
VERSION ""

NS_ :

BS_:

BU_: ECM TCM VCM

BO_ 2566844416 EEC1: 8 ECM
 SG_ EngineSpeed : 24|16@1+ (0.125,0) [0|8031.875] "rpm" VCM
 SG_ ActualEngineTorque : 16|8@1+ (1,-125) [-125|125] "%" VCM
 SG_ DriverDemandTorque : 8|8@1+ (1,-125) [-125|125] "%" VCM
 SG_ EngineControlMode : 0|4@1+ (1,0) [0|15] "" VCM

VAL_ 2566844416 EngineControlMode
 0 "Low Idle Governor"
 1 "Accelerator Pedal"
 3 "Remote Accelerator"
 ;
```

### J1939をDBCで表現する際の注意点

1. **29bit IDの表現**：DBCの`BO_`行のメッセージIDは10進表記。J1939の29bit拡張フレームを示すためIDに`0x80000000`を加算する慣例がある（ツール依存）。
   - 例：PGN 61444（0xF004）、SA=0x00、Priority=3 → CAN ID = `0x0CF00400` = `217056256`
   - DBCには `2566844416`（= `217056256 + 0x80000000`）と記載する場合が多い（要確定・ツール確認）

2. **SG_のビット定義**：`SG_ 名前 : 開始bit | ビット長 @ バイト順 符号 (resolution, offset) [min|max] "単位" 受信ノード`
   - `@1` = リトルエンディアン（Intel byte order）
   - `@0` = ビッグエンディアン（Motorola byte order）

3. **PGN・SPNのコメント記述**：DBCのコメント機能（`CM_`）でSPN番号を明示しておくと、後のトレーサビリティが向上する。

### ArXMLとの変換・連携

AUTOSARのネットワーク記述はArXMLで行うが、DBCとの相互変換ツールが各社から提供されている。

| ツール | 用途 |
|---|---|
| Vector CANdb++ | DBC作成・編集 |
| Vector DaVinci Developer | DBC→ArXML変換・COM設定 |
| ETAS ISOLAR-A | ArXML編集・J1939設定 |
| EB tresos Studio | ArXML編集・J1939 Stack設定 |

変換時の確認ポイント：
- SPN名・解像度・オフセットがArXML側に正しく引き継がれているか
- PGN→I-PDU のマッピングが正しいか
- 送信周期（CycleTime）が反映されているか

---

## 6. 診断との関係（DM1/DM2）

### DM1（Diagnostic Message 1）の構造

DM1はアクティブ（現在発生中）の故障コードを通知するPGN（PGN 65226、0xFECA）。

```
DM1フレームのデータフィールド（マルチパケットの場合あり）:
  Byte 0    : Lamp Status（MIL/RSL/AWL/PL点灯状態）
  Byte 1    : Lamp Status continued
  Byte 2-3  : SPN（SPNの下位19bit）＋FMI（5bit）＋CM(1bit)＋OC(7bit)
  Byte 4-5  : 2番目の故障コード（同じ構造）
  ...（最大21個まで 1パケットで、それ以降はTransport Protocolを使用）
```

### DM2（以前の故障コード）

- PGN 65227（0xFECB）
- 過去に発生し現在は解消された故障コードを保持
- 構造はDM1と同じ
- エンジン停止・バッテリ遮断後もフラッシュに保存される

### FMI（Failure Mode Identifier）

| FMI | 意味 |
|---|---|
| 0 | 高電圧・高データ値（Above Normal） |
| 1 | 低電圧・低データ値（Below Normal） |
| 2 | データが不安定・間欠的 |
| 3 | 電圧高（ショート to バッテリ） |
| 4 | 電圧低（ショート to グランド） |
| 5 | 電流低・開路 |
| 6 | 電流高・ショート to グランド |
| 7 | メカニカル故障 |
| 8 | 周期・波形異常 |
| 9 | 更新レート異常 |
| 10 | 異常な変化率 |
| 11 | 原因特定不可 |
| 12 | 不正デバイスまたはコンポーネント |
| 13 | キャリブレーション外 |
| 14 | 特殊指示 |
| 19 | ネットワーク通信エラー（受信タイムアウト等） |
| 31 | 条件あり（SAE J1939-73で定義） |

---

## 7. AUTOSARとの連携

### COM/PDUとPGN/SPNのマッピング

Classic AUTOSARのCOM（Communication）スタックは、I-PDU（Interaction Layer PDU）単位でデータを管理する。J1939の1PGN = 1I-PDUとして扱うのが基本。

```
J1939層                   AUTOSAR COM層
─────────────────         ─────────────────────────
PGN（例: EEC1）     →    I-PDU（例: EEC1_Tx）
  └── SPN 190      →      Signal（例: EngineSpeed）
  └── SPN 513      →      Signal（例: ActualEngineTorque）
```

設定項目（ArXML）：
- `J1939Tp`：J1939トランスポートプロトコルの設定
- `CanNm`：J1939ネットワーク管理との整合（アドレスクレーム等）
- `ComConfig`：I-PDUの送信周期・タイムアウト監視設定

### J1939Tp（Transport Protocol）

8バイトを超えるデータ（DM1の多故障コード等）はJ1939 TPを使用して分割送信する。

| フレームタイプ | PGN | 説明 |
|---|---|---|
| TP.CM (Connection Management) | 60416（0xEC00） | 接続開始・終了の制御 |
| TP.DT (Data Transfer) | 60160（0xEB00） | 実データの分割送信 |

AUTOSARのJ1939Tpモジュールがこの分割・再構成を自動処理するため、上位のCOMスタックからはシームレスに扱える。

---

## 8. アドレスクレーム（J1939-81 NAM）

J1939ではCANのIDがメッセージを識別するのではなく、各ECUが**8bitのSource Address（SA）**を動的に取得して通信に参加する仕組みがある。このSA取得プロセスがアドレスクレームである。

### 8.1 NAME（64bit識別子）

各ECUはROMに書き込まれた64bitの**NAME**を持つ。NAMEはアドレス競合時の優先順位決定と、ECUの種別識別に使われる。

```
NAME（8バイト）
Byte 7（MSB）                                    Byte 0（LSB）
┌─┬─────┬────────┬─┬────────┬──────┬──────────┬──────────────────────┐
│A│ IG  │  VS  │R│  VF  │ FI  │  FC        │   MFC        │ IDN  │
│1│ 3bit│ 4bit │1│ 4bit │5bit │   8bit     │   11bit      │21bit │
└─┴─────┴────────┴─┴────────┴──────┴──────────┴──────────────────────┘
```

| フィールド | ビット | 説明 |
|---|---|---|
| Identity Number（IDN） | 21bit | メーカーが割り当てるシリアル番号。ECU個体を一意に識別 |
| Manufacturer Code（MFC） | 11bit | SAEが割り当てるメーカーコード |
| ECU Instance（EI） | 4bit | 同一機能を持つECUが複数ある場合の識別（0が主） |
| Function Instance（FI） | 5bit | 同一Functionを持つECUの識別（0が主） |
| Function（FC） | 8bit | ECUの機能種別（エンジン=0、変速機=3等。SAE規定） |
| Vehicle System（VS） | 4bit | 車両システム種別（建設機械系等） |
| Industry Group（IG） | 3bit | 産業グループ（0=グローバル・1=道路車両・2=農業・3=建設等） |
| Arbitrary Address Capable（A） | 1bit | 1=競合時に別SAへ切り替え可能、0=固定SAのみ |

NAMEの数値が**小さいほど優先度が高い**（競合時に勝つ）。

### 8.2 アドレスクレームのシーケンス

```
ECU起動
  │
  ▼
① Claim Address（PGN 0xEE00）を送信
   ID: Priority=6, PGN=0xEE00, SA=自分が使いたいアドレス
   Data: 自分のNAME（8バイト）
  │
  ▼
② 250ms 待機しながらバスを監視
  │
  ├─── 競合なし（他のECUから同じSAのClaimが来ない）
  │      → そのSAを使って通常通信開始
  │
  └─── 競合あり（別ECUが同じSAをClaimしてきた）
         │
         ├─ 自分のNAME < 相手のNAME（自分が優先）
         │    → 再度 Claim Address を送信してSAを確保
         │
         └─ 自分のNAME > 相手のNAME（相手が優先）
              │
              ├─ Arbitrary Address Capable=1
              │    → Cannot Claim（PGN 0xEE00, SA=0xFE）を送信 → 別SAで再試行
              │
              └─ Arbitrary Address Capable=0
                   → 通信を諦める（エラー処理）
```

### 8.3 アドレスクレームに関わるPGN

| PGN | 10進 | 名称 | 用途 |
|---|---|---|---|
| 0xEE00 | 60928 | Address Claimed | SA取得の宣言・確保。SA=0xFEで送ると「取得失敗」 |
| 0xEA00 | 59904 | Request（リクエストPGN） | 相手ECUに特定PGNの送信を要求する汎用メッセージ |

アドレスクレームの流れで使われるのは主に0xEE00（宣言・確保）と、それを要求する0xEA00の組み合わせ。

### 8.4 建設機械での実態

- **エンジン・変速機等の市販ECU**：メーカが自社NAMEとデフォルトSAを設定して出荷。通常は固定SAで問題ない。
- **アタッチメント交換時**：アタッチメント側コントローラが起動するたびにアドレスクレームが走る。別のアタッチメントと同じSAを持っていると競合が起きる。
- **AUTOSARでの扱い**：J1939Nmモジュールが自動でアドレスクレームを処理する。ArXMLでNAMEとデフォルトSAを設定する。

---

## 9. リクエストPGN（PGN 0xEA00）

### 9.1 概要

J1939において、特定のPGNを相手ECUに「今すぐ送ってほしい」と要求するための仕組みがリクエストPGNである。

- **PGN**: 0xEA00（59904）、PDU1形式（宛先指定可能）
- **PS（PDU Specific）**: 要求先ECUのSA（0xFFで全ECUへのグローバル要求）
- **データ長**: 3バイト（要求するPGN番号をリトルエンディアンで格納）

```
リクエストフレーム例：DM2（過去故障）を SA=0x00（ECM）に要求する

CAN ID（29bit）:
  Priority=6, DP=0, PF=0xEA, PS=0x00（宛先=ECM）, SA=0xXX（自分）

データ（3バイト）:
  Byte 0: 0xCB  ← PGN 65227（0xFECB）の下位バイト
  Byte 1: 0xFE  ← 中位バイト
  Byte 2: 0x00  ← 上位バイト（DP=0）
```

### 9.2 リクエストPGNが必要な場面

J1939のPGNには**自律送信型**（周期送信）と**要求応答型**（Request時のみ送信）の2種類がある。

| 種別 | 動作 | 代表例 |
|---|---|---|
| 自律送信型（Cyclic） | ECUが一定周期で送り続ける | DM1（アクティブ故障）・EEC1（エンジン回転数） |
| 要求応答型（On-Request） | リクエストPGNを受けてから応答する | DM2（過去故障）・DM5（故障カウント）・ソフトウェアID等 |

**よく使うリクエスト先PGN**：

| 要求するPGN | 名称 | 説明 |
|---|---|---|
| 0xEE00（60928） | Address Claimed | ネットワーク起動時にバス上の全ECUのNAMEを収集する |
| 0xFECB（65227） | DM2 | 過去故障コードの読み出し（DM3で消去要求、DM2で確認） |
| 0xFEDA（65242） | Software Identification | ECUのソフトウェアバージョン取得 |
| 0xFEEB（65259） | Component Identification | ECUの部品番号・シリアル番号取得 |
| 0xFEEC（65260） | Vehicle Identification | VIN等の車両識別情報取得 |

### 9.3 グローバルリクエスト vs ピアツーピアリクエスト

```
グローバルリクエスト（PS=0xFF）
  → バス上のすべてのECUが要求されたPGNを持っていれば応答する
  → ネットワーク起動時のAddress Claimed収集等に使用

ピアツーピアリクエスト（PS=宛先SA）
  → 指定したECUのみが応答する
  → 特定のエンジンECUのDM2を読み出す等に使用
```

グローバルリクエストは多数のECUが一斉に応答するとバスが一時的に混雑するため、起動シーケンス以外では用途を絞って使う。

### 9.4 自動車CANとの比較

| | 自動車CAN | J1939 リクエストPGN |
|---|---|---|
| 診断情報の取得 | UDS 0x19サービス（ツールから要求） | DM2へのリクエストPGN（0xEA00）で要求 |
| ネットワーク参加ECUの確認 | CANNMのノード管理（別仕組み） | Address Claimed（0xEE00）をリクエスト |
| 汎用データ取得 | UDS 0x22（ReadDataById） | 要求応答型PGNにリクエストPGN送信 |

---

## 10. J1939 over EthernetとSOME/IPの使い分け

### 10.1 「意味定義の共通化」と「ワイヤフォーマットの一致」は別問題

建設機械のEthernet通信では、UDPのペイロードをJ1939と同じ構成にすることで、CAN側とEthernet側で同一のPGN/SPNシグナルカタログを使い回す設計が取られることがある。J1939はCAN 2.0Bをベースとした上位層プロトコルで、建機・農機・林業機械など幅広いオフハイウェイ車両で採用されており、全メーカーに共通する言語を提供する業界標準である。CAN/J1939はリアルタイム制御には適しているが現代のデータ需要に対応する容量を欠いており、Ethernetはリモートアクセス・ビッグデータ解析・IoT/IT基盤との統合を可能にすることでこのギャップを埋める役割を担うとされる。「J1939 over Ethernet」という形でPGNベースの構造をEthernetに拡張する手法は業界で認識されているパターンである。

CAN側とEthernet側で同一の意味定義（PGN/SPNのシグナルカタログ）を使い回すこと自体は、6章で整理したAUTOSARとの連携とも整合し、通信仕様の統一という観点で価値が高い。

一方で、ペイロードの意味定義（PGN/SPNのカタログ）を共通化することと、**ワイヤフォーマット（パケット構造そのもの）を一致させること**は別問題である。J1939はCANフレームの8バイト制約に由来する分割・再結合の仕組み（2章のTransport Protocol、マルチパケット、Connection Managementメッセージ）を持つが、Ethernetにはこの制約がない。UDPペイロードの構造をCAN由来の制約ごと引き継ぐと、Ethernetが本来持つ広い帯域を活かしきれない可能性がある。SPN単位の意味定義は共通化しつつ、Ethernet側はより効率的なペイロードレイアウト（J1939のパケット分割を模倣しない構造）を採用する余地がある。

### 10.2 SOME/IPという選択肢

SOME/IP（Scalable service-Oriented MiddlewarE over IP）は、AUTOSARコンソーシアムが標準化した、Ethernet上でUDP/TCPを用いてECU間のサービス指向通信を行うためのミドルウェアである。ECU間の通信用途ごとに「サービス」が定義され、これを提供する側（サーバ）と利用する側（クライアント）に分かれて相互にやり取りする。主な用途はRemote Procedure Call（RPC、メソッド呼び出し）、イベントのpublish/subscribe、フィールドの3種類であり、SOME/IP-SD（Service Discovery）によって、どのサービスがネットワーク上に存在するかを実行時に動的に検出できる。ペイロードが1パケットに収まらない場合はSOME/IP-TPによる分割が用いられる。SOME/IPは主にAdaptive AUTOSAR（ara::com）に対応するが、Classic AUTOSARからも利用可能な仕様が整備されており、Classic/Adaptive間の橋渡しにも使われる。詳細は `autosar-modules.md` 3章・7章を参照。

SOME/IPの利点は、機能を独立した「サービス」として扱うことで、後からリソース配置を変更する場合（あるECUの機能を別ECUに移す等）にも柔軟に対応できる点、稼働中の動的なサービス発見・接続が可能な点にある。一方で、サービスディスカバリやサブスクリプションの仕組みは、J1939のような固定的な周期配信と比べてオーバーヘッドが大きく、単純な周期信号の配信にはむしろ不向きである。

### 10.3 建設機械文脈での使い分け

以上を踏まえると、既存のPGN/SPNで表現できる周期的な制御系信号についてはJ1939ベースの統一を維持する妥当性が高く、これはAUTOSAR連携（6章）とも整合する。一方、今後追加される高帯域・非周期的なデータ（自動運転・遠隔運転関連のセンサ情報、カメラ映像、遠隔操作の状態通知等）は、動的なサービス発見やバージョニングが必要になる可能性が高く、そうした領域でSOME/IP（またはROS 2のDDS）を検討する方が適している。

J1939構造への無理な統一にこだわらず、既存の周期信号はJ1939ベースの統一を維持しつつ、新規の高帯域・イベント駆動データはSOME/IP等を別立てで採用する、という使い分けが妥当な整理になる。SOME/IPスタック（ara::com等）の新規導入は、周期信号の置き換えとして単独で先行させるものではなく、対象データの性質（周期的か非周期か、帯域要求、動的な接続変更の要否）を踏まえて個別に投資判断すべきである。

---

## 11. 委託先との仕様協議ポイント

### DBCファイルの提供・管理方法

- 自社が保有するDBCファイルをバージョン管理（Git）し、委託先に提供する
- DBCに不明なSPNが含まれる場合、J1939-71・J1939-75の原典を委託先が参照できるよう確認する（有償規格のため購入要否を確認）
- カスタムPGN・SPNは別ファイル（例: `custom_pgn.dbc`）で管理し、標準PGN定義と混在させない

### カスタムPGNの定義方法と命名規則

建設機械固有の信号はProprietaryPGNを使用する。

| 範囲 | 名称 | 用途 |
|---|---|---|
| PGN 0xFF00〜0xFFFF | Proprietary B (PPB) | ブロードキャスト。会社固有の制御データ |
| PGN 0xEF00（PDU1） | Proprietary A (PPA) | ピアツーピア。特定ECU向け制御コマンド |

命名規則（例）：
```
SPN命名: [会社略称]_[機能]_[パラメータ名]
例: ABC_HYD_BucketPressure（ABC社・油圧系・バケット圧力）

PGN命名: [会社略称]_[サブシステム]_[グループ名]
例: ABC_HYD_WorkingEquipment1
```

### 協議時の確認事項チェックリスト

- [ ] 標準J1939-71・J1939-75のPGN/SPNで対応できない信号の洗い出し
- [ ] カスタムPGNのID割り当て（社内で一元管理する担当者を決める）
- [ ] DBCファイルの更新フロー（誰が変更し、誰がレビューするか）
- [ ] ArXML生成ツール（DaVinci・EB tresos等）でのDBC取り込み手順確認（要確定）
- [ ] TP使用が必要なメッセージの特定（8バイト超えのDM1等）
- [ ] 送信周期・タイムアウト値の定義（制御系は厳密な周期管理が必要）

---

## 参考規格・資料

| 規格 | 内容 |
|---|---|
| SAE J1939-21 | データリンク層 |
| SAE J1939-71 | Vehicle Application Layer（標準PGN/SPN定義） |
| SAE J1939-73 | Application Layer - Diagnostics（DM1/DM2/FMI定義） |
| SAE J1939-75 | Application Layer for Earthmoving Machines |
| SAE J1939-81 | Network Management（アドレスクレーム） |
| ISO 11898-1/2 | CAN物理層・データリンク層 |
| ISO 11783 | ISOBUS（農業機械・建設機械アタッチメント） |

> **注意**：本資料中の具体的な数値（PGN番号・SPN番号・ビット位置・スケール値等）は参考値である。実装時は上記SAE規格原典および委託先ECUメーカの仕様書で必ず確認すること（要確定箇所は個別に明示している）。
