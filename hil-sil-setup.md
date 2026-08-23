# SIL・HIL検証環境 構築手順書

対象読者：技術担当者  
関連ファイル：`verification-environment.md`（概念・全体像）、`dev-process.md`（CI/CD）、`functional-safety.md`（機能安全証跡）


> **段階区分の呼称について**：本書の「検証環境Step」は本書固有の段階区分であり、`roadmap.md` の全社ロードマップ「Phase 1/2/3」（今〜2年 / 2〜5年 / 5〜10年）とは別体系である。対応関係は `roadmap.md` 1章を参照。

---

## 1. SIL環境の構築手順

### 1.1 SILとは

SIL（Software-in-the-Loop）は、実機ECUハードウェアを使用せず、PC上でAUTOSAR BSW・RTE・SWCをシミュレーション実行することでソフトウェアの動作を検証する環境。  
実機が調達できない開発初期段階や、CI/CDパイプラインへの自動テスト組み込みに適している。

### 1.2 使用ツール例

| ツール | 用途 | 備考 |
|---|---|---|
| Vector CANoe + SIL kit | 仮想CAN/Ethernet環境でのECUシミュレーション | ライセンス費用要確定 |
| Vector vVIRTUALtarget | Classic AUTOSARのBSWをPC上で実行 | EB treosと組み合わせ可 |
| MATLAB/Simulink | SWCモデルの実行・MILとの比較検証 | MathWorks社 |
| AUTOSAR SIL kit（オープンソース） | ベクター社提供のOSSシミュレーション基盤 | 無償・GitHubで公開中 |

### 1.3 SIL環境の構成図

```
┌─────────────────────────────────────────────┐
│                開発PC（Linux / Windows）      │
│                                              │
│  ┌───────────────┐    ┌───────────────────┐ │
│  │ BSW + RTE     │    │ SWC（制御ロジック）│ │
│  │（vVIRTUALtarget│◄──►│（Simulinkモデル   │ │
│  │  or SIL kit） │    │  or C コード）    │ │
│  └──────┬────────┘    └───────────────────┘ │
│         │                                    │
│  ┌──────▼────────────────────────────────┐   │
│  │ 仮想CAN bus / 仮想Ethernet（SIL kit） │   │
│  └──────┬────────────────────────────────┘   │
│         │                                    │
│  ┌──────▼────────┐    ┌───────────────────┐ │
│  │ CANoe（仮想）  │    │ テストスクリプト  │ │
│  │ 信号モニタ・   │    │（Python / CAPL）  │ │
│  │ ログ取得       │    │                   │ │
│  └───────────────┘    └───────────────────┘ │
└─────────────────────────────────────────────┘
```

### 1.4 GitLab CI/CDへのSILテスト組み込み

Dockerコンテナ内にSIL実行環境を構築し、プッシュのたびに自動実行する。

```yaml
# .gitlab-ci.yml（SILテスト例）
stages:
  - build
  - sil-test

build-autosar:
  stage: build
  image: autosar-builder:latest   # ビルドツールチェーン入りイメージ（要整備）
  script:
    - ./scripts/build_bsw.sh
    - ./scripts/generate_rte.sh
  artifacts:
    paths:
      - output/sil_binary/

sil-regression:
  stage: sil-test
  image: silkit-runner:latest     # SIL kit + Pythonテストランナー（要整備）
  dependencies:
    - build-autosar
  script:
    - python3 tests/sil/run_tests.py --config tests/sil/config.yaml
    - python3 tests/sil/check_results.py --report output/sil_report.xml
  artifacts:
    reports:
      junit: output/sil_report.xml
    paths:
      - output/sil_logs/
  allow_failure: false
```

**注意事項：**
- CANoe等の商用ツールはライセンスサーバが必要なため、Dockerコンテナでの完全自動化には制約あり（要確定）
- AUTOSAR SIL kit（OSS）を使用することで、ライセンスなしのCI実行が可能になる可能性がある

### 1.5 SILで検証できること・できないこと

| 検証対象 | SIL可否 | 備考 |
|---|---|---|
| BSW・RTE・SWCのソフトウェアロジック | ○ | 主用途 |
| CAN/Ethernet通信プロトコルの動作 | ○ | 仮想バスで検証 |
| タイミング・リアルタイム性能 | △ | PCのスケジューラ依存、精度に限界あり |
| ハードウェア固有動作（ADC精度・割込み遅延） | × | HIL必須 |
| 電気的特性（電圧・電流） | × | HIL必須 |
| 実機センサ・アクチュエータとの接続 | × | HIL必須 |

### 1.6 V-ECU・FMIによるSIL→HIL資産再利用

vVIRTUALtargetのようなツールが提供する仮想ECU（V-ECU）は、実際のECUと同一のBSWコンフィグレーションで動作する仮想実行環境であり、実機を用意する前の段階からクラウドサーバー上を含む任意の環境で機能検証ができる。ハードウェア依存を排除することで、ECUハードウェア量産前にソフトウェアを検証でき、開発サイクルの短縮とコスト削減につながるとされる。

SIL段階で作成したテストケース・シナリオは、**FMI（Functional Mock-up Interface）規格準拠のV-ECU**であれば、HIL段階でそのまま再利用できるとされている。FMIは異なるシミュレーションツール間でモデルを交換するための標準インターフェースであり、V-ECUをFMU（Functional Mock-up Unit）としてパッケージ化することで、SIL用のテストハーネスをHILシミュレータでも流用できる。

テスト環境と仕様が別管理になりがちな運用では、こうした資産再利用の構造自体が存在しないため、SIL→HIL間でテストケースを都度作り直すことになりやすい。V-ECU/FMIベースの構成を検討する際は、SIL段階のテスト資産（テストケース・シナリオ・期待値）がHIL段階でどこまで再利用可能か、導入前にツールベンダーへ確認することを推奨する（要確定：具体的な再利用範囲はツール・バージョンに依存）。

**注意**：仮想化はツールが生成・提供する範囲が対象であり、実マイコンの割り込みタイミングやペリフェラル固有の挙動までは正確に再現されない。V-ECUでの検証はあくまでSIL段階の代替であり、最終的な実機検証（HIL・実車）は別途必要である。

---

## 2. HIL環境の構築手順

### 2.1 HILとは

HIL（Hardware-in-the-Loop）は、実際のECUハードウェアをHILシミュレータに接続し、実機と等価な電気的入出力（センサ信号・アクチュエータ駆動）を与えながら検証する環境。SILでは確認できないハードウェア依存動作・リアルタイム性能を検証する。

### 2.2 必要なハードウェア例

| 機器 | 用途 | 製品例（要確定） |
|---|---|---|
| HILシミュレータ本体 | リアルタイムモデル実行・IO制御 | dSPACE SCALEXIO、NI PXI |
| CANインターフェース | CAN/CAN FD信号の入出力 | Vector VN1630A等 |
| アナログ・デジタルIOボード | センサ信号模擬・アクチュエータ負荷 | HILシミュレータに内蔵の場合あり |
| 電源シミュレータ | バッテリ電圧変動テスト | Regatron等（要確定） |
| テスト管理PC | CANoe・テストスクリプト実行 | Windows PC |

### 2.3 HIL構成図

```
┌──────────────────┐         ┌────────────────────────────────┐
│   テスト管理PC    │         │       HILシミュレータ           │
│                  │◄────────►  ┌──────────────────────────┐  │
│ ・CANoe          │  Ethernet│  │ リアルタイムモデル         │  │
│ ・テストスクリプト│  / USB   │  │（油圧プラント・車体モデル）│  │
│ ・結果ログ管理    │         │  │（Simulink / AMESim）       │  │
└──────────────────┘         │  └────────────┬─────────────┘  │
                             │               │ アナログ/デジタルIO │
                             └───────────────┼────────────────┘
                                             │ ハーネス接続
                             ┌───────────────▼────────────────┐
                             │          実機ECU                │
                             │  ・Classic AUTOSAR BSW/RTE/SWC │
                             │  ・CAN/CAN FDコネクタ          │
                             │  ・センサ入力・アクチュエータ出力│
                             └────────────────────────────────┘
```

### 2.4 油圧シミュレーションモデルの例

建設機械固有の非線形プラントモデルをリアルタイムで実行する。

```
モデル構成例（Simulink / AMESim）：
  ・エンジン回転数モデル（燃料噴射 → トルク → 回転数）
  ・油圧ポンプモデル（回転数 → 吐出圧・流量、非線形特性含む）
  ・油圧シリンダモデル（流量 → シリンダ速度・位置、負荷変動あり）
  ・電動アクチュエータモデル（EV化対応）（要確定）
  ・作業機負荷モデル（土砂掘削抵抗等）

非線形要素の考慮ポイント：
  ・油圧応答遅延（配管容積・油温変化による粘度変動）
  ・ポンプ吐出特性の飽和
  ・シリンダのエンドクッション特性
```

### 2.5 CAN信号・センサ・アクチュエータの接続

```yaml
# CAN信号マッピング例（要確定・実車両の設計に合わせる）
can_channels:
  - id: CH1
    bus: CAN1
    baudrate: 500kbps
    messages:
      - name: EngineSpeed_Req
        id: 0x18FF0001    # J1939フォーマット（PGN要確定）
        direction: TX     # HILからECUへ送信（センサ値模擬）
      - name: PumpPressure_Resp
        id: 0x18FF0002
        direction: RX     # ECUからHILへ受信（制御指令）

analog_io:
  - name: ThrottleSensor
    type: voltage_input
    range: 0.5V - 4.5V   # 要確定
    mapping: ECU_PIN_A3
  - name: HydraulicPressureSensor
    type: current_input
    range: 4mA - 20mA    # 要確定
    mapping: ECU_PIN_A7
```

### 2.6 テスト自動化

```python
# Python制御例（CANoe COM API または SIL kitを使用）
import canoe_automation as cae  # ツール依存のライブラリ（要確定）

def test_pump_pressure_control():
    """油圧ポンプ圧力制御テスト"""
    # テスト初期化
    cae.set_env_var("EngineSpeed", 1500)   # rpm
    cae.set_env_var("HydOilTemp", 50)       # ℃（要確定）
    cae.wait(2.0)  # 安定待ち

    # 操作入力
    cae.set_env_var("ThrottlePedal", 0.8)  # 80%踏み込み

    # 応答確認（タイムアウト：要確定）
    result = cae.wait_for_condition(
        lambda: cae.get_signal("PumpPressure") > 25.0,  # MPa（要確定）
        timeout=3.0
    )
    assert result, "ポンプ圧力が規定値に達しなかった"

    # DTCが発生していないことを確認
    assert cae.get_env_var("ActiveDTC_Count") == 0
```

---

## 3. テストケース設計

### テストレベルと実施環境の対応

| テストレベル | 実施環境 | 対象 | 管理ツール例 |
|---|---|---|---|
| 単体テスト（SWCレベル） | SIL | 個々のSWCの入出力ロジック | GoogleTest / Unity |
| 統合テスト（ECU内） | SIL または HIL | BSW+RTE+全SWCの協調動作 | CANoe Test Feature Set |
| システムテスト（複数ECU間） | HIL | CAN通信・プラント応答・J1939シーケンス | CANoe / Python |
| 機能安全テスト（フォルト） | HIL | エラー検出・フォールバック動作 | CANoe + 専用フォルト注入 |

### テストケース管理ツールとの連携

- テストケースはPolarion ALM / JAMA（要確定）で管理し、要求仕様とトレースリンクを張る
- テスト実行結果（JUnit XML）をGitLab CIからPolarionへ自動インポート（要確定）
- ISO 25119証跡として「テストケースID ↔ 要求ID ↔ 実行結果」のトレーサビリティを維持する

---

## 4. フォルトインジェクションテスト

### 4.1 実施項目と方法

| フォルト種別 | 実施方法 | 確認内容 |
|---|---|---|
| CAN通信断 | CANoeでバス断線シミュレーション | タイムアウト検出・Dem DTCへの記録 |
| CANエラーフレーム注入 | CANoe Error Frame Generatorを使用 | エラーカウンタ動作・バスオフ回復 |
| センサ値異常（上限・下限超え） | HILのアナログIO電圧を意図的に逸脱させる | レンジ外検出・フォールバック制御 |
| センサ断線（オープン） | HILのIOを開放状態にする | 断線検出・安全状態移行 |
| 電源電圧低下 | 電源シミュレータでアンダー電圧を印加 | 低電圧検出・BSW動作継続確認 |
| 電源電圧過昇 | 電源シミュレータでオーバー電圧を印加 | 保護動作・Dem記録 |

### 4.2 Demモジュールとの連携確認

```
フォルト注入 → ECU内でDem_ReportErrorStatus()呼び出し
           → DTC記録・フリーズフレームデータ保存
           → フォールバック動作（制御値のクランプ等）への移行
           → フォルト解除後の回復シーケンス確認
```

---

## 5. 建設機械固有の考慮事項

### 5.1 油圧・電動アクチュエータのプラントモデル

- 油圧システムの応答遅延は数百ms〜数秒単位であり、一般的な車載ECUテストよりも長い制定時間が必要
- プラントモデルは稼働作業（掘削・旋回・走行）ごとに負荷条件が大きく異なる
- AMESim等の1D流体シミュレーションツールをSimulinkと連携させることで、精度の高い油圧特性を再現可能（要確定）

### 5.2 J1939通信のHILシミュレーション

- 建設機械はJ1939プロトコルを多用するため、CANoeにJ1939シンボルDBを読み込ませてSPN/PGN単位でのテストを実施
- 複数ECU（エンジンECU・油圧コントローラ・表示器等）をHIL上で模擬し、実機同等の通信シーケンスを再現する

### 5.3 現場環境テストの代替シミュレーション手法

| 試験 | HIL代替手法 | 限界 |
|---|---|---|
| 高温環境（エンジン室内） | HIL上でセンサ値・油温を高温相当に設定 | 部品の熱特性変化は模擬不可 |
| 低温環境（寒冷地起動） | 油粘度変化をプラントモデルパラメータで反映 | 実機の起動電流特性は模擬不可 |
| 振動・衝撃 | 制御アルゴリズムへの影響はSILで確認 | ハーネス・コネクタへの物理的影響は実機試験必要 |

---

## 6. 段階的な環境整備計画

```
検証環境Step 1（〜2027年）：SIL環境の整備
  ・AUTOSAR SIL kit（OSS）または vVIRTUALtargetを導入
  ・基本的なBSW・SWCのSILテストをGitLab CIに組み込み
  ・SWCレベルの単体テスト自動化（GoogleTest等）
  ・委託先にSIL環境の整備・テスト実施を要件として含める

検証環境Step 2（〜2029年）：HIL環境の構築
  ・HILシミュレータ（dSPACE SCALEXIO等）の選定・調達（要確定）
  ・油圧プラントモデルの整備（Simulink / AMESim）
  ・統合テスト・フォルトインジェクションテストをHILで実施
  ・Polarion等によるテストケース管理の本格運用開始
  ・ISO 25119証跡としての活用開始

検証環境Step 3（2030年以降）：デジタルツイン連携
  ・実機フィールドデータをプラントモデルにフィードバック
  ・仮想施工シナリオでの長期耐久テスト（`digital-twin.md`参照）
  ・HIL環境とデジタルツインの統合によるソフトウェア更新前のOTA検証
```

---

## 7. 委託先への要件

### 7.1 責任分担

| 項目 | 自社 | 委託先 |
|---|---|---|
| HILシミュレータ機材の調達 | ○（予算確保） | △（提案・仕様策定を支援） |
| 油圧プラントモデルの作成 | △（仕様提供・確認） | ○（作成・検証） |
| SIL環境のGitLab CI組み込み | △（CI基盤管理） | ○（SILランナー設定） |
| テストケースの設計・実施 | △（レビュー・承認） | ○（設計・実施） |
| テスト証跡の提出 | ×（受け取り） | ○（作成・提出） |

### 7.2 テスト証跡の形式と提出タイミング

委託先は以下の証跡を提出する義務を契約に明記すること。

- **SILテスト結果レポート**：JUnit XML形式またはPDF。各マイルストーン完了時
- **HILテストレポート**：テストケースID・実施日・合否・波形ログ。フェーズ完了時
- **フォルトインジェクション記録**：注入内容・ECU応答・DTC記録の対応表。システムテスト完了時
- **カバレッジレポート**：コードカバレッジ（MC/DC要求ありの場合は必須）。定期提出（要確定）

### 7.3 ISO 25119との整合

- SIL/HILテストはISO 25119の「ソフトウェア統合テスト」「ハードウェア・ソフトウェア統合テスト」の証跡として活用する
- テストケースは`functional-safety.md`で定義したHARAの安全要求にトレースする
- フォルトインジェクションテストはSafety Mechanismの有効性検証として機能安全計画（Safety Plan）に組み込む

---

## 参考：関連ドキュメント

| ドキュメント | 参照目的 |
|---|---|
| `verification-environment.md` | MIL/SIL/HILの概念・全体像 |
| `dev-process.md` | GitLab CI/CDパイプライン設計 |
| `functional-safety.md` | ISO 25119 Safety Plan・証跡管理 |
| `autosar-modules.md` | BSW・Dem・Dcmモジュール仕様 |
| `diagnostics.md` | DTC・UDS診断の詳細 |
| `digital-twin.md` | 検証環境Step 3のデジタルツイン連携構想 |
| `migration-plan.md` | 委託先との役割分担・マイルストーン |
