# Adaptive AUTOSAR 主要モジュール

2028年以降の次世代建設機械に向けた検討準備として、Adaptive AUTOSARの主要コンセプト・モジュール・Classic AUTOSARとの違いをまとめる。

---

## 1. Adaptive AUTOSARとは

Adaptive AUTOSAR（正式名称：AUTOSAR Adaptive Platform）は、**高性能SoC上のLinux/QNX等POSIXベースOSで動作するサービス指向ミドルウェアプラットフォーム**。

Classic AUTOSARが「安全クリティカルな制御系向け」の静的・リアルタイムな基盤であるのに対し、Adaptive AUTOSARは「HPC上の動的・コネクテッド・高機能な領域」向けに設計されている。

```
Classic AUTOSAR                    Adaptive AUTOSAR
────────────────────               ─────────────────────────────
マイコン（RH850・TC3xx等）          SoC（A72/A55コア等）
RTOS（OSEK/OS）                   Linux / QNX / Integrity
静的設計・ECU統合時に確定           実行時のサービス動的起動・停止
コンポーネント間の通信は静的         SOA（SOME/IP等）で動的接続
ドメイン型アーキテクチャ向け         ゾーン型・集中型向け
```

---

## 2. Classic AUTOSARとの比較

| 比較軸 | Classic AUTOSAR | Adaptive AUTOSAR |
|---|---|---|
| 動作環境 | マイコン（シングルコア〜）+ RTOS | SoC（マルチコア）+ POSIX OS |
| アーキテクチャ | 静的（設定はコード生成時に確定） | 動的（実行時にアプリを起動・停止可能） |
| 通信 | COM（PDU・Signal） | ara::com（SOME/IP・DDS） |
| アプリ形式 | SWC（Software Component） | AA（Adaptive Application） |
| 実行ファイル | 1つのECUに焼く一体型 | 個別プロセス（Unix fork/exec） |
| 更新 | Bootloader経由で全体更新 | OTA で個別アプリ更新可能 |
| 機能安全 | ISO 26262/ISO 25119 実績豊富 | 対応規格整備中（ISO 26262-6 最新版で対応） |
| 主な用途 | エンジン・ブレーキ・油圧制御 | ADAS・OTA・クラウド連携・HMI |

---

## 3. 主要な ara::* API（Adaptive Applicationのインタフェース）

Adaptive AUTOSARでは「ara」（AUTOSAR Runtime for Adaptive Applications）というC++ APIセットを通じて各機能にアクセスする。

### 3.1 ara::com（通信）

Adaptive AUTOSARの核心。サービス指向通信を実現するAPI。

```cpp
// サービス提供側（Provider）
ara::com::ServiceIdentifierType serviceId = ...;
auto skeleton = std::make_unique<RadarServiceSkeleton>(serviceId);
skeleton->OfferService();

// 速度データをPublish（Event）
skeleton->FrontObject.Send(frontObject);

// サービス利用側（Consumer）
auto proxy = RadarServiceProxy::StartFindService(...);
proxy->FrontObject.Subscribe(ara::com::EventCacheUpdatePolicy::kLastN, 1);
proxy->FrontObject.GetNewSamples([](auto sample) {
    // サンプルデータ処理
});
```

**主な機能**：
- **Event**：周期データの非同期発行・購読（Publish/Subscribe）
- **Method**：リクエスト・レスポンス型の呼び出し（RPC）
- **Field**：状態値の取得・設定・通知の組み合わせ

**通信バインディング（実装）**：
- SOME/IP（デフォルト）：CANからEthernetへの移行で最もよく使われる
- DDS（Data Distribution Service）：高頻度センサデータ向け
- 共有メモリ：同一SoC内の高速通信向け

### 3.2 ara::exec（実行管理）

Adaptive Applicationのライフサイクル（起動・停止・状態遷移）を管理する。

```
ExecutionManager（EM）
    │
    ├── Adaptive Application A（プロセスとして起動）
    │       State: Running / Terminating / ...
    ├── Adaptive Application B
    └── Adaptive Application C
```

- State Machineの定義（Init/Running/ShutDown等）はマニフェスト（JSON/XML）で記述
- 依存関係のあるアプリの起動順序をEMが管理

### 3.3 ara::diag（診断）

UDS診断をAdaptive AUTOSAR上で実現するAPI。Classic AUTOSARのDCMに相当。

- DiagnosticEventManager（DEM相当）：DTC管理・環境データ記録
- UDS Service Handler：$27 SecurityAccess・$31 RoutineControl等
- DoIPとの統合：EthernetベースのDoIP対応

### 3.4 ara::phm（Platform Health Management）

アプリケーションの死活監視・ウォッチドッグ機能。

```
SupervisedEntity（監視対象アプリ）
    │ ── Checkpoint通知 ──▶ HealthManagement
    │                           │
    │                       タイムアウト検出時:
    │                           ├── アプリ再起動
    │                           ├── ECUリセット
    │                           └── フォールバック動作
```

### 3.5 ara::log（ログ）

標準化されたログ出力API。DLT（Diagnostic Log and Trace）フォーマットに準拠。開発・デバッグ時の解析効率を向上させる。

```cpp
auto logger = ara::log::CreateLogger("APPL", "Application logger");
logger.LogInfo() << "Service started: " << serviceId;
```

### 3.6 ara::crypto（暗号）

セキュリティ機能のAPI。TLS・署名・MAC生成・鍵管理のインタフェースを提供。HSMとの連携。

### 3.7 ara::nm（ネットワーク管理）

ネットワーク状態（起動・スリープ・シャットダウン）の管理。Classic AUTOSARのNM（Network Management）に対応する機能をAdaptive向けに提供。

### 3.8 ara::iam（Identity and Access Management）

Adaptive Application間のアクセス制御。どのアプリがどのサービス・リソースにアクセスできるかを定義するセキュリティ機能。

---

## 4. Adaptive AUTOSARのサービス定義

### 4.1 サービスインタフェース定義

Adaptive AUTOSARのサービスはFRSI（Functional Requirements of a Service Interface）に基づくArXMLで定義する。開発ツールが.arxmlからC++スタブ・スケルトンを自動生成する。

```
┌─────────────────────────────────────────┐
│ FRSI定義（ArXML）                        │
│  Service: RadarService                   │
│  Events: FrontObject, RearObject         │
│  Methods: Calibrate()                    │
│  Fields: UpdateRate                      │
└─────────────────────────────────────────┘
          │ ツールによるコード生成
          ▼
┌──────────────────┐  ┌──────────────────┐
│ RadarSkeleton.hpp │  │ RadarProxy.hpp   │
│（サービス提供側）  │  │（サービス利用側）  │
└──────────────────┘  └──────────────────┘
```

### 4.2 サービスマニフェスト

各Adaptive Applicationは**マニフェストファイル**（SOME/IPマッピング・起動設定・依存関係等）を持つ。ExecutionManagerはマニフェストを読んでアプリを管理する。

---

## 5. 開発ツール

| ツール | ベンダー | 役割 |
|---|---|---|
| DaVinci Adaptive Developer | Vector Informatik | ArXML編集・コード生成（Adaptive向け） |
| EB tresos Adaptive Studio | Elektrobit | Adaptive AUTOSAR設定・生成 |
| AUTOSAR Builder | Eclipse ベース | AUTOSAR設計・検証 |
| CANoe / Scope | Vector | SOME/IP通信解析・テスト |

---

## 6. Adaptive AUTOSARのデプロイメントモデル

### 6.1 単一SoC構成

```
SoC（例：Qualcomm SA8295）
├── Linux (Adaptive AUTOSAR)
│   ├── OTA Manager（AA）
│   ├── Cloud Connector（AA）
│   └── ADAS Application（AA）
└── Safety Island（Classic AUTOSAR / RTOS）
    ├── Brake Control
    └── Steering Control
```

### 6.2 マルチSoC構成

建設機械の現実的な2028年以降の構成候補：

```
Classic AUTOSAR ECU（RTOS）          Linux/Adaptive AUTOSAR SoC
  油圧制御・走行制御                    OTA・クラウド・HMI・予知保全
        │                                        │
        └──────── CAN FD / Ethernet ─────────────┘
                  （ゲートウェイ連携）
```

---

## 7. 建設機械への適用展望（2028年以降）

### 7.1 適用が有望な領域

| 機能 | Adaptive AUTOSARが活きる理由 |
|---|---|
| OTAアップデート管理 | アプリ単位の個別更新、ロールバック管理 |
| クラウド連携・テレマティクス | MQTT/HTTPS通信、動的接続先変更 |
| 人検知・物体認識 | NN推論エンジンをAAとして動作させる |
| 予知保全・データ収集 | センサデータのSOA配信・クラウド転送 |
| リモート診断（SOVD） | HTTP/REST・JSON形式の診断サービス |

### 7.2 現時点での準備事項

- SOME/IPの理解を深める（Classic AUTOSARでもEthernetを導入する際に必要）
- ara::comのインタフェース設計思想（Event/Method/Field）に慣れる
- POSIX/Linuxスキルを組織内で育成しておく
- 将来のAdaptive移行を見越してサービスインタフェースをArXMLで定義する習慣を作る

---

## 8. まとめ

| 項目 | 内容 |
|---|---|
| 動作環境 | SoC + Linux/QNX（POSIXベース） |
| 主要API | ara::com・ara::exec・ara::diag・ara::phm等 |
| 通信方式 | SOME/IP（デフォルト）・DDS・共有メモリ |
| 設計単位 | Adaptive Application（個別プロセス） |
| 建設機械への適用 | 2028年以降。OTA・クラウド・人検知領域が先行候補 |
| 今から準備すること | SOME/IP理解・Linuxスキル育成・サービス設計の習慣化 |

Adaptive AUTOSARはClassic AUTOSARの代替ではなく補完関係。両者を組み合わせた**マルチプラットフォーム構成**が2030年代の建設機械の現実的な姿と考える。
