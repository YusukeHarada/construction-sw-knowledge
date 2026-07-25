# SecOC（Secure Onboard Communication）実装ガイド

対象読者：技術担当者・委託先  
関連ファイル：`autosar-modules.md`、`cybersecurity.md`、`communication-spec.md`

---

## 1. SecOCとは

### 1.1 概要と目的

SecOC（Secure Onboard Communication）は、Classic AUTOSARが定義する車載ネットワーク通信のセキュリティモジュールである。CAN バスは元来、受信ノードが送信元を検証する仕組みを持たないため、以下の攻撃が現実的な脅威となる。

- **なりすまし（Spoofing）**：攻撃者が正規ECUのCAN IDで偽のフレームを送出する
- **改ざん（Tampering）**：物理アクセスを得た攻撃者がフレームデータを変更する
- **リプレイ攻撃（Replay）**：過去の正規フレームを記録して再送する

SecOCは、PDUにMAC（Message Authentication Code）とフレッシュネスバリューを付加することで、これらの攻撃を検知する。

### 1.2 AUTOSARスタック内での位置付け

```
┌─────────────────────────────────────────────┐
│           Application / RTE                 │
├─────────────────────────────────────────────┤
│    Com（通信サービス層）                       │
├─────────────────────────────────────────────┤
│    SecOC ◀──── CSM（Crypto Service Manager） │
│                     │                       │
│              Crypto Stack（SHE/HSM）          │
├─────────────────────────────────────────────┤
│    PduR（PDUルーター）                         │
├─────────────────────────────────────────────┤
│    CanIf / CanTP                            │
├─────────────────────────────────────────────┤
│    CAN ドライバ                               │
└─────────────────────────────────────────────┘
```

SecOCは PduR と Com の間に位置し、送信時にMAC付加・受信時にMAC検証を行う。暗号演算は CSM を通じて Crypto Stack（HSM含む）に委譲する。

### 1.3 適用が必要なシナリオ

建設機械において SecOC の適用優先度が高い信号カテゴリは以下の通り。

| シナリオ | 理由 |
|---|---|
| アクチュエータ指令（油圧バルブ・走行モータ等） | 改ざん時の機体暴走リスク |
| 機能安全関連信号（ASIL B以上相当） | ISO 25119 の悪意ある操作対策 |
| スタート・ストップ制御 | 不正始動の防止 |
| ブレーキ・安全ロック解除 | 人身事故直結 |
| ゲートウェイ経由の外部入力 | 攻撃経路となりやすい |

---

## 2. MACの仕組み

### 2.1 CMAC（AUTOSAR推奨アルゴリズム）

AUTOSARはMAC生成アルゴリズムとして **AES-128-CMAC** を推奨している。CMACはブロック暗号（AES）をベースにしたMAC方式であり、SHEやHSMに広く実装されている。

```
MAC = CMAC(AES-128, 秘密鍵, データペイロード || フレッシュネスバリュー)
```

HMAC-SHA256 も選択肢だが、CAN の帯域・演算コストの観点から小規模ECUではCMACが現実的。

### 2.2 フレッシュネスバリュー（Freshness Value）

フレッシュネスバリューはリプレイ攻撃を防ぐための単調増加カウンタである。

```
フレッシュネスバリュー（例：64bit）
├── Trip Counter（電源ON/OFFの回数）   : 上位16bit
├── Message Counter（PDU送信回数）      : 下位48bit
└── ※実際のビット配分は要確定
```

**カウンタ管理の注意点**

- 電源断をまたいでもカウンタ値が逆行しないよう、NVM（不揮発性メモリ）に保存する
- Trip Counter はECU起動時にインクリメントし、Message Counter はPDU送信毎にインクリメントする
- カウンタのオーバーフロー時の動作（拒否・リセット手順）を仕様で明示する
- FreshnessValueProvider（FVP）モジュールがカウンタの配布と同期を管理する

### 2.3 トランケーション（MAC値の短縮）

フル長のCMACは128bit（16バイト）だが、CANフレームは最大8バイト（CAN FDは最大64バイト）のため、MAC値をトランケーションして搭載する。

| CAN種別 | フレームサイズ | SecOCオーバーヘッドの現実 |
|---|---|---|
| CAN 2.0 | 最大8バイト | 24bit MAC + フレッシュネス下位数bit が現実的な上限 |
| CAN FD | 最大64バイト | 64bit MAC + 充分なフレッシュネスバリューを搭載可能 |

トランケーションにより衝突確率は上がるが、フレッシュネスバリューとの組み合わせにより実用的な安全性は維持される（AUTOSAR仕様内でトランケーション24bitが例示されている）。

### 2.4 CANフレームへの適用例（CAN 2.0、8バイト）

```
送信前（元のPDU）:
┌────────────────────────────────────────┐
│ CAN ID │ DLC │ データ（最大8バイト）      │
└────────────────────────────────────────┘

SecOC適用後（SecuredIPdu）:
┌──────────────────────────────────────────────────────────────────┐
│ CAN ID │ DLC │ 有効ペイロード（4B）│ FV下位（1B）│ MAC（3B/24bit）│
└──────────────────────────────────────────────────────────────────┘
※バイト配分は設計により異なる。要確定。
```

有効ペイロードが削減される点に注意し、J1939信号のPGN設計段階からSecOC適用を考慮する必要がある。

---

## 3. 鍵管理

### 3.1 対称鍵の配布

SecOCはノード間で共通の対称鍵（AES-128）を使用する。鍵配布の一般的な方針は以下の通り。

```
製造・出荷フロー:
  1. 安全な環境（鍵管理サーバ）で鍵を生成
  2. 製造ラインで HSM の鍵スロットに書き込み（SHE KeyUpdate プロトコル等）
  3. 鍵はHSM外に平文で出さない（以後、HSM内に封印）
  4. 同一通信グループのECU群に同一鍵を配布
```

### 3.2 HSMとの連携原則

**鍵をHSM外に出さない** ことがセキュリティの根幹である。

```
┌──────────────────────────────┐
│  Application Core             │
│  SecOC → CSM → Crypto Driver  │
│                │               │
│          ┌─────▼──────┐        │
│          │    HSM      │        │
│          │  ┌───────┐  │        │
│          │  │ 鍵A   │  │        │
│          │  │ 鍵B   │  │        │
│          │  └───────┘  │        │
│          │ CMAC演算のみ │        │
│          │ 結果を返す  │        │
│          └────────────┘        │
└──────────────────────────────┘
```

- アプリケーションコアは鍵を直接参照できない
- HSMはデータを受け取り、内部で演算し、MACのみを返す
- HSM選定は委託先ハードウェアとの適合確認が必要（要確定：SHE / HSM+ 等）

### 3.3 鍵更新（キーロールオーバー）

稼働後の鍵漏洩リスクに備え、鍵更新の仕組みを設計段階で定義する。

| 方式 | 概要 | 建設機械での現実性 |
|---|---|---|
| 整備時書込 | ディーラーツール接続時に新鍵を配布 | 現実的。既存の診断インフラ活用 |
| OTA経由 | クラウドから暗号化した鍵を配信 | 2028年以降のOTA整備後に検討 |
| 工場回収 | 機体を工場に戻して書換 | 緊急時のフォールバックとして保持 |

---

## 4. AUTOSARのSecOC設定手順（概要）

AUTOSARツール（DaVinci Developer / EB tresos 等）での設定の流れを示す。

### 4.1 SecuredIPduの定義

1. **ISignalIPdu** として元のPDUを定義（Com モジュール側）
2. **SecOCSecuredIPdu** を作成し、元のIPduを参照させる
3. SecuredIPduに以下を設定する：
   - `SecOCDataLength`：有効ペイロードのビット長
   - `SecOCFreshnessValueLength`：フレッシュネスバリュー**全体**のビット長（例：64bit）
   - `SecOCFreshnessValueTxLength`：フレームに**送信する**FVビット長（例：4〜8bit、下位ビット）
   - `SecOCAuthInfoTxLength`：MACトランケーション長（例：24bit（CAN 2.0）/ 64bit（CAN FD））

   ※`SecOCFreshnessValueLength` と `SecOCFreshnessValueTxLength` は混同しやすい。前者はFVMが管理する全体長、後者は帯域節約のためフレームに載せる下位ビット長である。

**ArXML記述例**

油圧制御コマンド（CAN ID: 0x201）にSecOCを適用する場合のArXML断片。値はシステム設計に合わせて確定すること。

```xml
<!-- SecOC Protected I-PDU の定義 -->
<SECURED-I-PDU>
  <SHORT-NAME>HydCtrl_Cmd_Secured</SHORT-NAME>
  <CONTAINED-I-PDU-PROPS>
    <CONTAINED-PROPS>
      <SHORT-NAME>HydCtrl_Cmd</SHORT-NAME>
    </CONTAINED-PROPS>
  </CONTAINED-I-PDU-PROPS>
  <!-- Freshness Value 設定 -->
  <FRESHNESS-VALUE-LENGTH>40</FRESHNESS-VALUE-LENGTH>
  <FRESHNESS-VALUE-TX-LENGTH>8</FRESHNESS-VALUE-TX-LENGTH>
  <!-- MAC 設定 -->
  <AUTH-INFO-TX-LENGTH>24</AUTH-INFO-TX-LENGTH>
  <AUTH-ALGORITHM-REF DEST="CRYPTO-ALGO-CONFIG">
    /AutosarPackages/CryptoConfig/AES128_CMAC
  </AUTH-ALGORITHM-REF>
  <!-- 検証失敗時の動作 -->
  <MESSAGE-LINK-POSITION>0</MESSAGE-LINK-POSITION>
  <RECEPTION-OVERFLOW-HANDLING>REJECT</RECEPTION-OVERFLOW-HANDLING>
</SECURED-I-PDU>
```

**主要パラメータの推奨値**

| パラメータ名 | 意味 | 推奨値 |
|---|---|---|
| `SecOCFreshnessValueLength` | FV全体のビット長（FVMが管理） | 40ビット以上（例：64ビット） |
| `SecOCFreshnessValueTxLength` | フレームに送信するFVビット長 | 4〜8ビット（下位ビット） |
| `SecOCAuthInfoTxLength` | MACトランケーション長 | 24〜48ビット（CAN 2.0）／ 64ビット（CAN FD） |
| `SecOCMacVerifyAttempts` | MAC検証失敗を許容する回数 | 3回（要確定） |
| `SecOCReceptionOverflowStrategy` | 受信バッファあふれ時の動作 | `REJECT`（破棄） |

### 4.2 FreshnessValueProviderの設定

```
SecOCFreshnessValueProvider
├── FreshnessValueId    : PDUごとに一意のID
├── FreshnessValueLength: FV全体のビット長（例：64bit）
└── 実装：FvM（Freshness Value Manager）または独自実装
```

FvMモジュールがある場合はそれを使用する。ない場合はカスタムFVP実装を委託先に要求する。

### 4.3 CsmJobの設定（Crypto Service Manager）

```
CsmJob（CMAC生成用）
├── CsmJobType        : MAC_GENERATE
├── CsmJobPrimitive   : CMAC_AES128
├── CsmKeyId          : 使用する鍵スロットID（HSM内）
└── CsmJobResultRef   : 結果格納先

CsmJob（CMAC検証用）
├── CsmJobType        : MAC_VERIFY
├── CsmJobPrimitive   : CMAC_AES128
└── CsmKeyId          : 受信側の同一鍵スロットID
```

### 4.4 生成コードとHSMドライバの結合

```
生成物:
  SecOC_Cfg.c / SecOC_Cfg.h      ← AUTOSARツールが生成
  Csm_Cfg.c / Csm_Cfg.h          ← AUTOSARツールが生成
  Crypto_76_HaeHsm.c（例）       ← HSMベンダーが提供（要確定）

結合ポイント:
  CsmJobが → Crypto Driverを呼び出し → HSM APIに転送
  HSMベンダーのCrypto Driver実装がAUTOSAR Crypto Driver仕様に準拠していること
```

HSMドライバはMCUベンダーまたはHSMソリューションベンダーが提供する。委託先がサポート可能か事前確認が必要。

---

## 5. 建設機械への適用判断

### 5.1 J1939信号への適用優先度

| 信号グループ | 適用優先度 | 理由 |
|---|---|---|
| 走行・旋回モータ指令 | **高** | 機体暴走・転倒リスク |
| ブレーキ・安全ロック | **高** | 人身事故直結 |
| エンジン始動・停止 | **高** | 不正始動防止 |
| アーム・バケット速度指令 | **中** | 作業員への危険 |
| センサフィードバック（油圧・温度） | **低** | 情報漏洩リスクは低い |
| ダイアグノーシス（DM1等） | **低** | 既存の診断保護で対処 |

### 5.2 CAN FDとの組み合わせ

CAN FD（最大64バイト）を採用することで SecOC オーバーヘッドの問題が大幅に緩和される。

```
CAN 2.0（8バイト）: 有効ペイロードが4バイト程度に制限 → 既存信号の再設計が必要
CAN FD（64バイト）: 64bitFV + 64bitMAC を搭載しても 48バイトの有効ペイロード確保可能
```

CAN FD移行と SecOC 導入をセットで計画することを推奨する。

### 5.3 現時点（2026年）での現実的な導入ステップ

```
フェーズ1（2026〜2027年）：基盤構築
  - 委託先と共に適用PDU一覧・鍵管理方針を合意
  - HSM搭載MCUの選定（要確定）
  - SecOC・CSM・Crypto Driverの初期設定・PoC
  - 優先度「高」の信号のみSecOC適用

フェーズ2（2027〜2028年）：拡大適用
  - CAN FD移行に合わせて適用範囲拡大
  - 鍵更新フロー（整備時書込）の整備
  - HIL環境でのMACエラーインジェクションテスト実施

フェーズ3（2029年以降）：OTA鍵配信
  - クラウド連携OTA整備後に鍵ロールオーバー自動化を検討
```

---

## 6. 委託先への要件定義

### 6.1 SOWへの記載事項

```
【SecOC要件（SOW記載例）】

1. 適用PDU一覧
   - 対象PDU名・CAN ID・信号内容を一覧化（別紙参照）
   - ※一覧は要確定。PoC開始前にJointで作成

2. 使用アルゴリズム
   - MACアルゴリズム：AES-128-CMAC
   - フレッシュネスバリュー長：64bit（全体）、下位Nbitをフレームに搭載（要確定）
   - MACトランケーション長：24bit（CAN 2.0）/ 64bit（CAN FD）（要確定）

3. 鍵管理方針
   - 鍵はHSM内に格納し、平文でのECU外部出力を禁止する
   - 鍵書込フロー（製造ライン・整備時）の手順書を納品物に含める
   - 鍵スロット設計（グループ鍵か個体鍵か）を仕様書に明記する

4. 使用ツールチェーン
   - AUTOSARコンフィグレータ（DaVinci / EB tresos、要確定）
   - Crypto Driverは委託先が提供するか確認（MCUベンダー提供品の場合はその旨記載）
```

### 6.2 HSM選定と委託先サポート確認

委託先に以下を事前確認する。

| 確認項目 | 確認内容 |
|---|---|
| 対応MCU | 採用予定MCUにHSM/SHEが搭載されているか（要確定） |
| Crypto Driver | AUTOSAR Crypto Driver仕様に準拠したドライバを提供できるか |
| SecOCモジュール | 採用AUTOSARベンダー（Vector / ETAS 等）のSecOC実装実績があるか |
| 鍵書込ツール | 製造ライン向け鍵プロビジョニングツールを持っているか |
| FvM対応 | FreshnessValueManagerの実装・設定経験があるか |

### 6.3 検証方法

```
検証ケース例：

[TC-SECOC-001] 正常系MAC検証
  手順：正規ECUから SecuredIPdu を送信
  期待：受信ECUでMAC検証成功・信号値がアプリに渡る

[TC-SECOC-002] MACエラー検出
  手順：Vectorツール（CANoe等）でMACビットを意図的に反転して送信
  期待：SecOCがエラーを検出し、アプリへの信号渡しをブロックする
        Demにセキュリティイベントが記録される

[TC-SECOC-003] リプレイ攻撃検出
  手順：正規フレームを記録後、再送（フレッシュネスバリューが古い）
  期待：受信ECUがリプレイを検出・拒否する

[TC-SECOC-004] フレッシュネスバリュー同期
  手順：受信ECUの電源ON/OFFを繰り返した後に通信を再開
  期待：Trip Counterが正しく同期され、正常通信が再開する
```

ツール：CANoe（Vector Informatik）のSecurityテスト機能を推奨（要確定）。

---

## 参考：関連AUTOSARモジュール

| モジュール | 役割 |
|---|---|
| SecOC | MAC付加・検証、Secured PDU管理 |
| PduR | PDUルーティング（SecOC↔CanIf） |
| Com | 信号パック・アンパック |
| CSM（Crypto Service Manager） | 暗号サービスの抽象化層 |
| Crypto Driver | HSM/SHEとのIF |
| FvM（Freshness Value Manager） | フレッシュネスバリューの配布・同期 |
| Dem | SecOCエラーの診断イベント記録 |

---

*数値・製品名（MCU型番・HSM仕様・ツールバージョン等）はいずれも要確定。実装時には委託先および各ベンダーと仕様をすり合わせること。*
