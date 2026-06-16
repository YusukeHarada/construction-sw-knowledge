# AUTOSARコード生成ツールチェーン

**作成日：** 2026年6月16日

---

## 概要

通信仕様（DBC・ArXML）をインプットとして、AUTOSARのBSWコンフィグおよびコードを生成するツールチェーンの全体像を整理する。自社開発ではなく外部委託を前提とするが、ツールの役割を理解することで成果物のレビューと委託先の評価が可能になる。

---

## 1. ツールチェーン全体像

```
【インプット】
  通信仕様書（DBC / ARXML / Fibex）
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
       ↓
【Step 3】コード生成ツール
  ArXMLからCソースコードを自動生成
  （RTE・BSWコンフィグコード）
       ↓
【Step 4】コンパイル・リンク
  生成コード＋アプリSWC（手書き）→ ECUバイナリ
       ↓
【アウトプット】
  ECU実行バイナリ + テストレポート + 設計ドキュメント
```

---

## 2. 主要ツールと役割

### 2-1. Vectorグループ（最も普及しているツールチェーン）

| ツール名 | 役割 | 入力 | 出力 |
|---|---|---|---|
| **DaVinci Developer** | SWC設計ツール | 要求仕様 | SWC定義ArXML |
| **DaVinci Configurator Pro** | BSWコンフィグツール | SWC ArXML・DBC | BSWコンフィグArXML |
| **DaVinci Generator** | コード生成ツール | コンフィグArXML | Cソースコード |
| **CANdb++** | CAN DB管理ツール | 手入力 | DBC |
| **CANoe** | 通信シミュレーション・テスト | DBC・バイナリ | テスト結果 |

**特長：** 実績が最も多く、委託先候補のほとんどが対応している。ライセンスコストは高め。

### 2-2. Elektrobitグループ（EB tresos）

| ツール名 | 役割 | 備考 |
|---|---|---|
| **EB tresos Studio** | BSWコンフィグ＋コード生成 | Eclipseベース、オープン性が高い |
| **EB tresos AutoCore** | BSW本体 | AUTOSAR Classic対応 |

**特長：** DaVinciよりオープンなアーキテクチャ。EB（Elektrobit）はContinentalグループ。

### 2-3. ETAS（Boschグループ）

| ツール名 | 役割 |
|---|---|
| **ETAS ISOLAR-A** | AUTOSAR SWC開発環境 |
| **ETAS ISOLAR-EVE** | 仮想ECU（バイナリなしでテスト可能） |
| **RTA-OS** | AUTOSAR OS |
| **RTA-BSW** | BSWスタック |

**特長：** Bosch系のプロジェクトで多く使われる。仮想ECU環境が充実。

### 2-4. ORIENTAIS（東軟グループ・アジア系委託先で多い）

| ツール名 | 役割 |
|---|---|
| **ORIENTAIS Configurator** | BSWコンフィグ＋コード生成 |
| **ORIENTAIS OS** | AUTOSAR OS |

**特長：** コストが比較的低い。中国・アジア系の委託先で使われることが多い。日本国内での実績は少ない。

---

## 3. 通信仕様からコード生成までの流れ（具体例）

### Step 1：DBC → ArXML変換

CANメッセージリスト（DBC）をAUTOSARの通信記述（ArXML）に変換する。

```
ツール：Vector DaVinci Configurator Pro または CANdb++ + スクリプト

入力：
  HydraulicCommand.dbc（メッセージ・シグナル定義）

出力（ArXML）：
  <I-PDU>
    <SHORT-NAME>HydraulicCommand_IPDU</SHORT-NAME>
    <LENGTH>8</LENGTH>
    <I-SIGNAL-I-PDU-GROUP-REF>...</I-SIGNAL-I-PDU-GROUP-REF>
  </I-PDU>
  <I-SIGNAL>
    <SHORT-NAME>BoomLiftCmd</SHORT-NAME>
    <DATA-TYPE-REF>...</DATA-TYPE-REF>
    <START-POSITION>0</START-POSITION>
    <LENGTH>16</LENGTH>
  </I-SIGNAL>
```

### Step 2：BSWコンフィグ

COMモジュール・PDURモジュール等の設定をArXMLで行う。

```
DaVinci Configurator Proでの設定例：

COMモジュール：
  - HydraulicCommand_IPDU をTxPDUとして登録
  - 送信周期：10ms
  - BoomLiftCmd シグナルをCOM Signal として登録

CANIFモジュール：
  - CAN ID 0x100 を HydraulicCommand_IPDU にマッピング
  - 対象CANコントローラ：CAN1

PDURモジュール：
  - COMからCANIFへのルーティングを定義
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

| ツール・フレームワーク | 役割 | 備考 |
|---|---|---|
| **AUTOSAR Adaptive Platform（AP）** | ミドルウェア基盤 | SOME/IP・ARA APIを提供 |
| **Vector MICROSAR.AP** | Adaptive AUTOSAR実装 | 商用 |
| **ETAS RTA-AP** | Adaptive AUTOSAR実装 | 商用 |
| **OpenAA** | オープンソース実装 | コスト削減可能だがサポートなし |
| **Manifest Editor** | サービス・実行定義ツール | JSON/ArXMLで設定 |

Adaptive AUTOSARでは、通信はSOME/IP経由でサービスとして定義され、araCOM APIを通じてアクセスする。

---

## 5. 自社担当者が最低限把握すべきこと

外部委託で開発してもらう場合でも、以下は自社側で理解しておく必要がある。

| 理解レベル | 内容 |
|---|---|
| **必須** | ArXMLの構造（何がどこに定義されているか読める） |
| **必須** | SWC・ポート・Runnableの概念 |
| **必須** | 通信設定（CANIF・COM・PDUR）の役割と設定項目 |
| **推奨** | DaVinci Configurator Proの基本操作 |
| **推奨** | CANoeを使ったCAN通信の確認方法 |
| **任意** | コード生成の仕組みの詳細 |

**PoC期間中に委託先から教えてもらうことを契約に含める**のが、最も効率的な習得方法である。

---

## 6. ツールライセンスの費用感

参考として（市場価格は変動するため必ず見積もりを取ること）。

| ツール | 費用感 | 備考 |
|---|---|---|
| Vector DaVinci Configurator Pro | 高コスト | 年間ライセンス、機能単位で追加 |
| Vector CANoe | 高コスト | モジュール構成で変動 |
| EB tresos Studio | 中〜高コスト | BSWとセット購入が基本 |
| ETAS ISOLAR | 中コスト | ETASのBSWとセットが多い |

**自社でツールを購入する必要はない。** 外部委託先がツールを持っていることを選定条件にすることで、ライセンスコストを委託先に持たせることができる。自社に必要なのは、成果物（ArXML・ソースコード）を確認するためのビューアーレベルのツールで十分。
