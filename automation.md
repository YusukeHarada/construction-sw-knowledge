# AUTOSAR開発における自動化戦略

建設機械メーカーが外部委託でAUTOSAR準拠プラットフォームを構築し、自社担当者1〜2名で要求定義・受け入れ検証を担う前提での自動化方針。

---

## 1. 自動化できるもの・できないものの整理

| 領域 | 自動化可否 | 判断主体 |
|------|-----------|---------|
| ArXMLからのコード生成 | 可 | ツール |
| ビルド・静的解析 | 可 | ツール |
| SIL/通信テスト実行 | 可 | ツール |
| 要求IDとテストのトレース | 可（半自動） | ツール＋人 |
| 要求の妥当性確認 | 不可 | 自社担当者 |
| ArXML設計判断（ポート・インタフェース） | 不可 | 担当者＋委託先 |
| 安全目標の解釈・ASILアサイン | 不可 | 担当者（機能安全担当者） |

**原則:** ツールは「正しく実装されているか」を確認するが、「何が正しいか」は人が決める。

---

## 2. コード生成の自動化

### ArXML → RTE・BSWコンフィグコード

```yaml
# GitLab CI例
generate:
  stage: codegen
  script:
    - python tools/dbc2arxml.py input/network.dbc -o arxml/Com.arxml
    - vendor_tool/arxml2rte.sh arxml/ -o generated/
  artifacts:
    paths: [generated/]
```

- **DBC → ArXML変換:** `cantools` や社内スクリプトで自動化。CAN信号定義の変更がArXMLに即反映される
- **RTE生成:** Vector DaVinci / EB tresos のCLIを使いCI上で実行
- **ビルド:** CMake + Ninja構成、GitLab CIでステージ分割（generate → build → test）

---

## 3. テスト自動化

### GitLab CI YAMLサンプル（完全構成）

以下は、コード生成からSILテスト・CANoe通信テスト・静的解析・証跡レポートまでをカバーする実用的な`.gitlab-ci.yml`全体構成である。

```yaml
# .gitlab-ci.yml — AUTOSAR 建設機械プラットフォーム自動化パイプライン

stages:
  - codegen
  - build
  - lint
  - test
  - canoe_test
  - report

variables:
  ARXML_DIR: "arxml/"
  GENERATED_DIR: "generated/"
  BUILD_DIR: "build/"
  RESULTS_DIR: "results/"
  CANOE_CFG: "canoe/hydraulic_test.cfg"

# ────── 1. コード生成 ──────
codegen:
  stage: codegen
  image: registry.example.com/autosar/builder:latest
  script:
    - python tools/dbc2arxml.py input/network.dbc -o ${ARXML_DIR}Com.arxml
    - davinci_gen --project project.dvprj --output ${GENERATED_DIR}
  artifacts:
    paths: [${GENERATED_DIR}]
    expire_in: 1 week
  rules:
    - changes:
        - "arxml/**"
        - "input/*.dbc"

# ────── 2. ビルド（SIL用ホストビルド） ──────
build:sil:
  stage: build
  image: registry.example.com/autosar/builder:latest
  script:
    - cmake -S . -B ${BUILD_DIR} -DCMAKE_BUILD_TYPE=Debug -DTARGET=SIL -G Ninja
    - cmake --build ${BUILD_DIR} --parallel $(nproc)
  artifacts:
    paths: [${BUILD_DIR}]
  needs: [codegen]

# ────── 3. MISRA-C静的解析 ──────
lint:misra:
  stage: lint
  image: registry.example.com/autosar/pclintp:latest
  script:
    - mkdir -p ${RESULTS_DIR}
    - pclp64 +ffn -u misra_c2012.lnt $(find src/ -name "*.c") \
        > ${RESULTS_DIR}misra_report.txt 2>&1 || true
    - python tools/parse_misra.py ${RESULTS_DIR}misra_report.txt \
        --format gitlab-codequality \
        --output ${RESULTS_DIR}misra_codequality.json
    # Errorが1件でもあればパイプラインを失敗させる
    - |
      ERR=$(grep -c "^Error" ${RESULTS_DIR}misra_report.txt || echo 0)
      echo "MISRA-C Error: ${ERR} 件"
      test "${ERR}" -eq 0
  artifacts:
    when: always
    paths:
      - ${RESULTS_DIR}misra_report.txt
    reports:
      codequality: ${RESULTS_DIR}misra_codequality.json
  needs: [build:sil]

# ────── 4. Unit SILテスト（Google Test） ──────
test:unit:
  stage: test
  image: registry.example.com/autosar/builder:latest
  script:
    - mkdir -p ${RESULTS_DIR}
    - cmake --build ${BUILD_DIR} --target unit_tests
    - ${BUILD_DIR}unit_tests \
        --gtest_output=xml:${RESULTS_DIR}unit_test.xml \
        --gtest_color=yes
    # カバレッジ計測（gcov）
    - gcovr --root . --xml-pretty -o ${RESULTS_DIR}coverage.xml
  artifacts:
    when: always
    reports:
      junit: ${RESULTS_DIR}unit_test.xml
      coverage_report:
        coverage_format: cobertura
        path: ${RESULTS_DIR}coverage.xml
  coverage: '/TOTAL.*\s+(\d+%)$/'
  needs: [build:sil]

# ────── 5. Component SILテスト（VEOS） ──────
test:sil:
  stage: test
  image: registry.example.com/autosar/veos:latest
  script:
    - mkdir -p ${RESULTS_DIR}
    - veos-run \
        --config config/sil_config.xml \
        --testcases tests/component/ \
        --timeout 300 \
        --output ${RESULTS_DIR}sil_raw.xml
    - python tools/veos2junit.py ${RESULTS_DIR}sil_raw.xml \
        -o ${RESULTS_DIR}sil_test.xml
  artifacts:
    when: always
    reports:
      junit: ${RESULTS_DIR}sil_test.xml
  needs: [build:sil]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_TAG'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ────── 6. CANoe通信テスト（Windowsランナー使用） ──────
test:canoe:
  stage: canoe_test
  tags:
    - windows    # CANoeはWindowsのみ対応のため専用ランナーを使用
  script:
    - mkdir -Force results | Out-Null
    # CANoe CLIでテスト実行（PowerShell構文）
    - |
      $proc = Start-Process -FilePath "CANoe64.exe" `
        -ArgumentList "/cfg ${CANOE_CFG} /run /test HydraulicTestModule /quit /exitcode" `
        -Wait -PassThru
      if ($proc.ExitCode -ne 0) {
        Write-Error "CANoe tests failed with exit code $($proc.ExitCode)"
        exit 1
      }
    # CANoeが出力したXMLをJUnit形式に変換
    - python tools\canoe2junit.py results\canoe_output.xml -o results\canoe_test.xml
  artifacts:
    when: always
    paths:
      - results\canoe_test.xml
      - results\*.blf    # CANoe Traceログ（証跡として保存）
    reports:
      junit: results\canoe_test.xml
  needs: [test:unit]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_TAG'

# ────── 7. 証跡レポート生成 ──────
report:evidence:
  stage: report
  image: python:3.12-slim
  script:
    - pip install jinja2 weasyprint --quiet
    - python tools/generate_evidence.py \
        --unit   ${RESULTS_DIR}unit_test.xml \
        --sil    ${RESULTS_DIR}sil_test.xml \
        --misra  ${RESULTS_DIR}misra_report.txt \
        --commit ${CI_COMMIT_SHA} \
        --branch ${CI_COMMIT_REF_NAME} \
        --output ${RESULTS_DIR}evidence_report.pdf
  artifacts:
    paths:
      - ${RESULTS_DIR}evidence_report.pdf
    expire_in: 90 days
  needs: [test:unit, test:sil, lint:misra]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

### Robot FrameworkによるCAN通信テストサンプル

Robot FrameworkとCANoe Python API（Vector CANoe APIまたはcantools）を組み合わせたCAN通信テストの完全なサンプルを示す。

**ディレクトリ構成**

```
tests/
├── resources/
│   ├── can_keywords.resource    # CAN操作のキーワード定義
│   └── hydraulic_data.resource  # テストデータ定義
└── hydraulic_control_tests.robot
```

**`can_keywords.resource`（キーワード定義）**

```robot
*** Settings ***
Library    CANoeLibrary    # 社内実装のCANoe接続ライブラリ
Library    Collections

*** Keywords ***
CANoe接続を開始する
    [Arguments]    ${cfg_path}
    Open CANoe Configuration    ${cfg_path}
    Start Measurement
    Sleep    500ms    # 測定開始待ち

CANoe接続を終了する
    Stop Measurement
    Close CANoe Configuration

CAN信号を送信する
    [Arguments]    ${message}    &{signals}
    FOR    ${name}    ${value}    IN    &{signals}
        Set Signal Value    ${message}    ${name}    ${value}
    END
    Send Message    ${message}

CAN信号を受信して確認する
    [Arguments]    ${message}    ${signal}    ${expected}    ${tolerance}=0.5    ${timeout}=200ms
    ${start}=    Get Time    epoch
    WHILE    True
        ${actual}=    Get Signal Value    ${message}    ${signal}
        ${diff}=    Evaluate    abs(${actual} - ${expected})
        IF    ${diff} <= ${tolerance}
            Log    ${signal}: ${actual}（期待値: ${expected}±${tolerance}）— OK
            RETURN
        END
        ${elapsed}=    Evaluate    (Get Time("epoch") - ${start}) * 1000
        IF    ${elapsed} > 200    # 200ms タイムアウト
            Fail    ${signal} がタイムアウト内に期待値に達しなかった: actual=${actual}, expected=${expected}±${tolerance}
        END
        Sleep    5ms
    END

フォルト状態を注入する
    [Arguments]    ${fault_type}
    Set Signal Value    SystemStatus    FaultCode    ${fault_type}
    Send Message    SystemStatus
    Sleep    10ms

システムをリセットする
    Set Signal Value    SystemStatus    FaultCode    0
    Send Message    SystemStatus
    Sleep    50ms    # リセット後の安定待ち
```

**`hydraulic_control_tests.robot`（テストケース本体）**

```robot
*** Settings ***
Resource    resources/can_keywords.resource
Resource    resources/hydraulic_data.resource
Suite Setup       CANoe接続を開始する    ${CANOE_CFG}
Suite Teardown    CANoe接続を終了する

*** Variables ***
${CANOE_CFG}    ${CURDIR}/../canoe/hydraulic_test.cfg
${CAN_CYCLE}    10ms

*** Test Cases ***
# ────── 正常系テスト ──────

REQ-HYD-001_ブームリフト_正常応答確認
    [Documentation]    REQ-HYD-001: オペレータレバー入力50%に対して、
    ...    バルブ指令値が50±1%の範囲でHydraulicCommandメッセージに
    ...    乗ってくることを確認する
    [Tags]    正常系    油圧制御    REQ-HYD-001

    システムをリセットする
    CAN信号を送信する    OperatorInput    BoomLiftCmd=50.0    ArmCmd=0.0
    Sleep    ${CAN_CYCLE}
    CAN信号を受信して確認する
    ...    message=HydraulicCommand
    ...    signal=BoomValveCmd
    ...    expected=50.0
    ...    tolerance=1.0

REQ-HYD-002_ブームリフト_最大入力クランプ確認
    [Documentation]    REQ-HYD-002: 入力が100%を超えた場合でも
    ...    バルブ指令値が100%にクランプされることを確認する
    [Tags]    正常系    油圧制御    境界値    REQ-HYD-002

    システムをリセットする
    CAN信号を送信する    OperatorInput    BoomLiftCmd=110.0
    Sleep    ${CAN_CYCLE}
    CAN信号を受信して確認する
    ...    message=HydraulicCommand
    ...    signal=BoomValveCmd
    ...    expected=100.0
    ...    tolerance=0.5

REQ-HYD-003_入力ゼロ時にバルブ全閉確認
    [Documentation]    REQ-HYD-003: レバー入力が0%のとき、
    ...    バルブ指令値が0%（全閉）になることを確認する
    [Tags]    正常系    油圧制御    REQ-HYD-003

    CAN信号を送信する    OperatorInput    BoomLiftCmd=0.0
    Sleep    ${CAN_CYCLE}
    CAN信号を受信して確認する
    ...    message=HydraulicCommand
    ...    signal=BoomValveCmd
    ...    expected=0.0
    ...    tolerance=0.1

# ────── 異常系・フェールセーフテスト ──────

REQ-SAFE-001_フォルト時フェールセーフ動作確認
    [Documentation]    REQ-SAFE-001: SystemStatusがFault（FaultCode=1）の場合、
    ...    すべてのバルブ指令値が0%（安全状態）になることを確認する
    [Tags]    異常系    フェールセーフ    REQ-SAFE-001

    # まず正常入力を送る
    CAN信号を送信する    OperatorInput    BoomLiftCmd=50.0
    Sleep    ${CAN_CYCLE}
    # フォルト注入
    フォルト状態を注入する    1
    Sleep    ${CAN_CYCLE}
    # フェールセーフ確認
    CAN信号を受信して確認する
    ...    message=HydraulicCommand
    ...    signal=BoomValveCmd
    ...    expected=0.0
    ...    tolerance=0.1
    [Teardown]    システムをリセットする

REQ-SAFE-002_センサ断線フォルト時フェールセーフ確認
    [Documentation]    REQ-SAFE-002: 圧力センサ断線（FaultCode=2）時に
    ...    フェールセーフ移行することを確認する
    [Tags]    異常系    フェールセーフ    REQ-SAFE-002

    フォルト状態を注入する    2
    Sleep    ${CAN_CYCLE}
    CAN信号を受信して確認する
    ...    message=HydraulicCommand
    ...    signal=BoomValveCmd
    ...    expected=0.0
    ...    tolerance=0.1
    [Teardown]    システムをリセットする

# ────── タイミング検証テスト ──────

REQ-TIMING-001_CANメッセージ送信周期確認
    [Documentation]    REQ-TIMING-001: HydraulicCommandメッセージが
    ...    10±1msサイクルで送信されることを確認する
    [Tags]    タイミング    REQ-TIMING-001

    ${timestamps}=    Capture Message Timestamps    HydraulicCommand    count=10
    ${intervals}=    Calculate Intervals    ${timestamps}
    FOR    ${interval}    IN    @{intervals}
        Should Be True    ${interval} >= 9    メッセージ間隔が下限9ms未満: ${interval}ms
        Should Be True    ${interval} <= 11   メッセージ間隔が上限11msを超過: ${interval}ms
    END
```

---

### SILテスト（CIパイプライン組み込み）

```yaml
sil_test:
  stage: test
  script:
    - cmake --build build/ --target sil_test
    - ./build/sil_test --gtest_output=xml:results/sil.xml
  artifacts:
    reports:
      junit: results/sil.xml
```

### CANoe CLI（通信テスト）

```bash
# CANoe CLIで非対話実行
CANoe64.exe /cfg test.cfg /run /test TestModule1 /quit /exitcode
```

通信テストのパス・フェイルをCIのexit codeで判定し、GitLabのパイプラインに統合する。

### MISRA-C静的解析

- **Polyspace / Helix QAC:** CLIでCI実行し、違反レポートをアーティファクトとして保存
- 委託先に「CI通過を納品条件」として契約仕様に明記するのが効果的

---

## 4. 要求管理・トレーサビリティの自動化

```
要求ID（Polarion / GitLab Issue）
  ↓ 紐付け
ArXML要素（SoftwareComponent / Port）
  ↓ 紐付け
テストケース（Robot Framework / gtest）
  ↓ 自動収集
機能安全証跡レポート（PDF / HTML）
```

- **Polarion ALM:** 要求とテスト結果をAPI連携で自動紐付け
- **GitLab Issuesのみの場合:** ラベル・カスタムフィールドで要求IDを管理し、CI結果をIssueにコメント投稿するスクリプトで代替可能
- **証跡レポート:** CIの成果物（テスト結果XML・静的解析レポート）をまとめてPDFに変換するスクリプトを用意する

---

## 5. ArXML差分レビューの自動化

委託先から受け取ったArXMLを自動検証するPythonスクリプトの完全なサンプルを以下に示す。要求仕様（CSV）と照合することで、設計漏れ・誤配線を納品物受領時に自動検出できる。

```python
#!/usr/bin/env python3
"""
arxml_check.py — ArXML自動検証スクリプト
委託先から受領したArXMLに対して以下を検証する：
  1. 要求仕様（CSV）に定義されたSWCが全て存在するか
  2. 必須ポート（Provide/Require）が全て定義されているか
  3. ポートのデータ型が要求仕様と一致するか
  4. git diff との組み合わせで変更箇所を特定する
"""

import xml.etree.ElementTree as ET
import csv
import sys
import json
import subprocess
from pathlib import Path
from dataclasses import dataclass, field
from typing import Optional

# AUTOSARのXML名前空間定義
NS = {
    "ar": "http://autosar.org/schema/r4.0",
}

@dataclass
class PortRequirement:
    """要求仕様から読み込んだポート定義"""
    swc_name: str
    port_name: str
    port_kind: str      # "provide" or "require"
    data_type: str
    description: str = ""

@dataclass
class CheckResult:
    """検証結果"""
    passed: bool
    errors: list[str] = field(default_factory=list)
    warnings: list[str] = field(default_factory=list)

def load_port_requirements(csv_path: str) -> list[PortRequirement]:
    """
    要求仕様CSVを読み込む。
    フォーマット: swc_name, port_name, port_kind, data_type, description
    """
    requirements = []
    with open(csv_path, encoding="utf-8-sig") as f:
        reader = csv.DictReader(f)
        for row in reader:
            requirements.append(PortRequirement(
                swc_name=row["swc_name"].strip(),
                port_name=row["port_name"].strip(),
                port_kind=row["port_kind"].strip().lower(),
                data_type=row["data_type"].strip(),
                description=row.get("description", "").strip(),
            ))
    return requirements

def parse_arxml_ports(arxml_path: str) -> dict[str, dict]:
    """
    ArXMLからSWCとポート定義を解析して返す。
    戻り値: {swc_name: {"provide": {port_name: data_type}, "require": {...}}}
    """
    tree = ET.parse(arxml_path)
    root = tree.getroot()

    result: dict[str, dict] = {}

    # APPLICATION-SW-COMPONENT-TYPE を探す
    for swc in root.iter("{http://autosar.org/schema/r4.0}APPLICATION-SW-COMPONENT-TYPE"):
        short_name_el = swc.find("{http://autosar.org/schema/r4.0}SHORT-NAME")
        if short_name_el is None:
            continue
        swc_name = short_name_el.text or ""
        result[swc_name] = {"provide": {}, "require": {}}

        # P-PORT-PROTOTYPE（Provideポート）
        for p_port in swc.iter("{http://autosar.org/schema/r4.0}P-PORT-PROTOTYPE"):
            port_sn = p_port.find("{http://autosar.org/schema/r4.0}SHORT-NAME")
            if port_sn is None:
                continue
            port_name = port_sn.text or ""
            # データ型はPROVIDED-INTERFACE-TREF から取得
            iref = p_port.find(".//{http://autosar.org/schema/r4.0}PROVIDED-INTERFACE-TREF")
            data_type = iref.text.split("/")[-1] if iref is not None and iref.text else "unknown"
            result[swc_name]["provide"][port_name] = data_type

        # R-PORT-PROTOTYPE（Requireポート）
        for r_port in swc.iter("{http://autosar.org/schema/r4.0}R-PORT-PROTOTYPE"):
            port_sn = r_port.find("{http://autosar.org/schema/r4.0}SHORT-NAME")
            if port_sn is None:
                continue
            port_name = port_sn.text or ""
            iref = r_port.find(".//{http://autosar.org/schema/r4.0}REQUIRED-INTERFACE-TREF")
            data_type = iref.text.split("/")[-1] if iref is not None and iref.text else "unknown"
            result[swc_name]["require"][port_name] = data_type

    return result

def check_arxml_against_requirements(
    arxml_path: str,
    requirements_csv: str,
) -> CheckResult:
    """
    ArXMLと要求仕様CSVを照合して検証する。
    """
    result = CheckResult(passed=True)

    # 要求仕様読み込み
    try:
        requirements = load_port_requirements(requirements_csv)
    except Exception as e:
        result.errors.append(f"要求仕様CSVの読み込みに失敗: {e}")
        result.passed = False
        return result

    # ArXML解析
    try:
        arxml_ports = parse_arxml_ports(arxml_path)
    except Exception as e:
        result.errors.append(f"ArXMLの解析に失敗: {e}")
        result.passed = False
        return result

    # 照合
    for req in requirements:
        swc = req.swc_name

        # SWCの存在確認
        if swc not in arxml_ports:
            result.errors.append(
                f"[SWC不在] {swc} が ArXML に存在しない"
                f"（要求: {req.port_name}/{req.port_kind}）"
            )
            result.passed = False
            continue

        # ポートの存在確認
        port_dict = arxml_ports[swc].get(req.port_kind, {})
        if req.port_name not in port_dict:
            result.errors.append(
                f"[ポート不在] {swc}/{req.port_name}({req.port_kind}) が ArXML に存在しない"
            )
            result.passed = False
            continue

        # データ型の一致確認
        actual_type = port_dict[req.port_name]
        if actual_type != req.data_type:
            result.errors.append(
                f"[型不一致] {swc}/{req.port_name}: "
                f"要求={req.data_type}, ArXML={actual_type}"
            )
            result.passed = False

    return result

def get_changed_arxml_files(base_branch: str = "main") -> list[str]:
    """
    git diff を使って変更されたArXMLファイルの一覧を返す。
    """
    try:
        output = subprocess.check_output(
            ["git", "diff", "--name-only", f"origin/{base_branch}...HEAD"],
            text=True,
        )
        return [f for f in output.splitlines() if f.endswith(".arxml")]
    except subprocess.CalledProcessError:
        return []

def main():
    import argparse
    parser = argparse.ArgumentParser(description="ArXML自動検証スクリプト")
    parser.add_argument("--arxml",    required=True, help="検証対象のArXMLファイル")
    parser.add_argument("--requirements", required=True, help="要求仕様CSV")
    parser.add_argument("--output",   default=None, help="結果をJSONで出力するパス")
    parser.add_argument("--diff-only", action="store_true",
                        help="git diffで変更されたファイルのみ検証する")
    args = parser.parse_args()

    if args.diff_only:
        changed = get_changed_arxml_files()
        if not changed:
            print("ArXMLの変更なし。スキップします。")
            sys.exit(0)
        print(f"変更されたArXML: {changed}")
        arxml_target = changed[0]  # 複数ある場合は拡張する
    else:
        arxml_target = args.arxml

    result = check_arxml_against_requirements(arxml_target, args.requirements)

    # 結果表示
    if result.errors:
        print(f"\n[エラー] {len(result.errors)} 件")
        for err in result.errors:
            print(f"  ✗ {err}")
    if result.warnings:
        print(f"\n[警告] {len(result.warnings)} 件")
        for warn in result.warnings:
            print(f"  △ {warn}")

    if result.passed:
        print("\n✓ ArXML検証 PASSED")
    else:
        print("\n✗ ArXML検証 FAILED")

    # JSON出力（CI連携用）
    if args.output:
        with open(args.output, "w", encoding="utf-8") as f:
            json.dump({
                "passed": result.passed,
                "errors": result.errors,
                "warnings": result.warnings,
            }, f, ensure_ascii=False, indent=2)

    sys.exit(0 if result.passed else 1)

if __name__ == "__main__":
    main()
```

**GitLab CIへの組み込み例**

```yaml
# .gitlab-ci.yml に追加
lint:arxml:
  stage: lint
  image: python:3.12-slim
  script:
    - pip install lxml --quiet
    - python tools/arxml_check.py \
        --arxml arxml/HydraulicControl.arxml \
        --requirements requirements/port_requirements.csv \
        --output results/arxml_check.json \
        --diff-only
  artifacts:
    when: always
    paths:
      - results/arxml_check.json
  rules:
    - changes:
        - "arxml/**/*.arxml"
```

- **差分確認:** `git diff` でArXMLの変更箇所を可視化。構造的なdiffにはXML diffツール（`xmldiff`等）を活用
- **整合チェック:** 要求仕様（CSV/Excel）から期待するインタフェース一覧を生成し、ArXMLと突き合わせる

---

## 6. 建設機械向け自動化優先順位

### 今すぐ始める（コストゼロ〜低コスト）

1. GitLab CI設定ファイル（`.gitlab-ci.yml`）の作成 → ビルド自動化
2. `git diff` によるArXML変更の可視化
3. MISRA違反レポートのCI自動収集（既存ツールのCLI活用）
4. Robot Frameworkによる受け入れテストの雛形作成
5. `arxml_check.py`（上記）によるArXML差分検証の自動化

### 中期的に投資すべき（効果大）

1. **DBC → ArXML変換スクリプト:** 仕様変更のたびに手作業が発生するため、早期に自動化するとROIが高い
2. **トレーサビリティ自動生成:** 担当者1〜2名では手動管理が限界になるため、Polarion連携またはスクリプトによる証跡収集を優先
3. **SILテストのCI統合:** 委託先の実装品質を継続的に可視化できるため、受け入れ工数を大幅に削減できる
4. **Robot FrameworkのCANoe連携:** 上記サンプルを起点に、テストケースを主要機能（油圧制御・旋回制御・走行制御）ごとに拡充する
