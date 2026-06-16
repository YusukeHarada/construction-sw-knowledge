# 開発基盤・プロセス整備ガイド

**作成日：** 2026年6月16日  
**対象読者：** 開発部門担当者・マネージャー  
**資料種別：** 社内検討用（社外秘）

---

## 概要

AUTOSAR準拠プラットフォームの外部委託移行と並行して、自社の開発基盤・プロセスを整備する必要がある。本ドキュメントはCI/CD・DevOps・アジャイル・TDD/ATDDの組み込み・車載開発への適用方針を整理する。

---

## 1. 現状の課題と方向性

### 現状

- 仕様書・設計書なし
- 単体テストなし
- CI/CDなし
- ソフトウェアプロセスなし
- 機能安全の証跡なし

### 目指す方向性

「一度に全部変える」のは不可能。**リグレッションを防ぐ最小テストの整備とCI化を最初に行い、アジャイルやATDDはその後に段階導入する**。

---

## 2. CI/CD

### AUTOSAR開発でCI/CDを導入する際の課題

| 課題 | 内容 |
|---|---|
| ハードウェア依存 | ECU固有のフラッシュ書き込みが必要なHILテストはCIパイプラインへの組み込みが難しい |
| ビルド時間 | Classic AUTOSARはRTE生成・BSW設定がArXMLベースで複雑なため数十分かかることがある |
| テスト自動化の範囲 | HIL環境がなければ実行できるテストの種類が限られる |

### SIL（Software-in-the-Loop）を使ったCI/CDアプローチ

HIL環境がなくてもCIで自動テストできる構成がSILである。ECUソフトウェアをLinuxホストやDockerコンテナ上でコンパイルし、仮想環境でテストを実行する。

```
┌────────────────────────────────────────────┐
│ CIパイプライン（GitLab CI / Jenkins）       │
│                                            │
│  git push → ビルド → SILテスト → レポート  │
│                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │Unit SIL  │  │Component │  │System    │ │
│  │Google    │  │SIL       │  │SIL       │ │
│  │Test/CMock│  │仮想ECU   │  │複数仮想  │ │
│  │          │  │通信検証  │  │ECU結合   │ │
│  └──────────┘  └──────────┘  └──────────┘ │
└────────────────────────────────────────────┘
         ↓ 合格後
┌────────────────────────────────────────────┐
│ HILテスト（実機確認。CI外で定期実施）       │
└────────────────────────────────────────────┘
```

**3段階のSILテスト構成：**

| レベル | テスト内容 | ツール |
|---|---|---|
| **Unit SIL** | RTOS依存を抽象化してビジネスロジック単体テスト | Google Test / CMock / Unity |
| **Component SIL** | 仮想ECU環境で通信・タイミング検証 | dSPACE VEOS / CANoe .NET |
| **System SIL** | 複数仮想ECUをネットワーク結合した機能シナリオ検証 | AUTOSAR Adaptive Simulator |

### GitLab CI パイプライン YAMLサンプル（全ステージ構成）

以下はClassic AUTOSARのビルドからSILテスト・静的解析・証跡生成までをカバーする`.gitlab-ci.yml`の実用的なサンプルである。

```yaml
# .gitlab-ci.yml
# Classic AUTOSAR CI パイプライン（建設機械プラットフォーム向け）

stages:
  - codegen    # ArXML → RTE/BSWコード生成
  - build      # クロスコンパイル + ホストビルド（SIL用）
  - lint       # MISRA-C静的解析
  - test       # Unit SIL + Component SIL
  - report     # 証跡レポート生成

variables:
  ARXML_DIR: "arxml/"
  GENERATED_DIR: "generated/"
  BUILD_DIR: "build/"
  RESULTS_DIR: "results/"

# ────────────── ステージ1: コード生成 ──────────────
codegen:
  stage: codegen
  image: registry.example.com/autosar/builder:latest  # 社内Dockerイメージ
  script:
    # DBC → ComArXML変換
    - python tools/dbc2arxml.py input/network.dbc -o ${ARXML_DIR}Com.arxml
    # RTE・BSW生成（Vector DaVinci CLIの例）
    - davinci_gen --project project.dvprj --output ${GENERATED_DIR}
    - echo "コード生成完了: $(ls ${GENERATED_DIR} | wc -l) ファイル"
  artifacts:
    paths:
      - ${GENERATED_DIR}
    expire_in: 1 week
  rules:
    - changes:
        - "arxml/**/*"
        - "input/*.dbc"

# ────────────── ステージ2: ビルド ──────────────
build:sil:
  stage: build
  image: registry.example.com/autosar/builder:latest
  script:
    - cmake -S . -B ${BUILD_DIR} -DCMAKE_BUILD_TYPE=Debug -DTARGET=SIL
    - cmake --build ${BUILD_DIR} --parallel $(nproc)
  artifacts:
    paths:
      - ${BUILD_DIR}
  needs: [codegen]

build:target:
  stage: build
  image: registry.example.com/autosar/cross-gcc:latest  # クロスコンパイラ入りイメージ
  script:
    - cmake -S . -B build_target/ -DCMAKE_TOOLCHAIN_FILE=cmake/arm-none-eabi.cmake
    - cmake --build build_target/ --parallel $(nproc)
  artifacts:
    paths:
      - build_target/*.elf
  needs: [codegen]

# ────────────── ステージ3: 静的解析 ──────────────
lint:misra:
  stage: lint
  image: registry.example.com/autosar/pclintp:latest
  script:
    - mkdir -p ${RESULTS_DIR}
    # PC-lint Plus でMISRA-C:2012チェック
    - pclp64 +ffn -u misra_c2012.lnt src/**/*.c > ${RESULTS_DIR}misra_report.txt 2>&1 || true
    # 違反件数をカウント（Error/Warning別）
    - python tools/parse_misra.py ${RESULTS_DIR}misra_report.txt --format gitlab > ${RESULTS_DIR}misra_gl.json
    - |
      ERROR_COUNT=$(grep -c "^Error" ${RESULTS_DIR}misra_report.txt || echo 0)
      echo "MISRA Error: ${ERROR_COUNT} 件"
      if [ "${ERROR_COUNT}" -gt "0" ]; then exit 1; fi
  artifacts:
    when: always
    paths:
      - ${RESULTS_DIR}misra_report.txt
    reports:
      codequality: ${RESULTS_DIR}misra_gl.json
  needs: [build:sil]

# ────────────── ステージ4: テスト ──────────────
test:unit:
  stage: test
  image: registry.example.com/autosar/builder:latest
  script:
    - mkdir -p ${RESULTS_DIR}
    - ${BUILD_DIR}unit_tests --gtest_output=xml:${RESULTS_DIR}unit_test.xml
  artifacts:
    when: always
    reports:
      junit: ${RESULTS_DIR}unit_test.xml
  needs: [build:sil]

test:sil:
  stage: test
  image: registry.example.com/autosar/veos:latest  # dSPACE VEOS入りイメージ
  script:
    - mkdir -p ${RESULTS_DIR}
    # VEOS上でコンポーネントSILテストを実行
    - veos-run --config config/sil_config.xml \
               --testcases tests/component/ \
               --output ${RESULTS_DIR}sil_raw.xml
    # JUnit形式に変換
    - python tools/veos2junit.py ${RESULTS_DIR}sil_raw.xml -o ${RESULTS_DIR}sil_test.xml
  artifacts:
    when: always
    reports:
      junit: ${RESULTS_DIR}sil_test.xml
  needs: [build:sil]
  # SILテストは時間がかかるためmainブランチとtagのみ実行
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_TAG'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ────────────── ステージ5: 証跡レポート ──────────────
report:evidence:
  stage: report
  image: python:3.12-slim
  script:
    - pip install jinja2 weasyprint --quiet
    - python tools/generate_evidence.py \
        --unit-result ${RESULTS_DIR}unit_test.xml \
        --sil-result  ${RESULTS_DIR}sil_test.xml \
        --misra-report ${RESULTS_DIR}misra_report.txt \
        --commit ${CI_COMMIT_SHA} \
        --output ${RESULTS_DIR}evidence_report.pdf
  artifacts:
    paths:
      - ${RESULTS_DIR}evidence_report.pdf
    expire_in: 90 days
  needs: [test:unit, test:sil, lint:misra]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

### SILのDockerコンテナ構成：具体的な手順

Unit SILをDockerコンテナで動かすための手順を以下に示す。前提としてLinuxホスト（Ubuntu 22.04）とDocker Engine 24以降を想定する。

**手順1：Dockerfileの作成**

```dockerfile
# Dockerfile.sil
FROM ubuntu:22.04

# ビルド・テストに必要なパッケージ
RUN apt-get update && apt-get install -y \
    gcc g++ cmake ninja-build \
    libgtest-dev libgmock-dev \
    python3 python3-pip \
    git curl \
    && rm -rf /var/lib/apt/lists/*

# Google Testをビルドしてインストール
RUN cd /usr/src/googletest && \
    cmake -B build -G Ninja && \
    cmake --build build && \
    cmake --install build

# CMockのインストール（CのモックフレームワークはC言語SWCのテストに使う）
RUN pip3 install cmock

WORKDIR /workspace
```

**手順2：ローカルでのビルド・テスト確認**

```bash
# イメージをビルド
docker build -f Dockerfile.sil -t autosar/sil-builder:latest .

# ソースをマウントしてテスト実行
docker run --rm \
  -v $(pwd):/workspace \
  autosar/sil-builder:latest \
  bash -c "cmake -S . -B build -DTARGET=SIL && cmake --build build && ./build/unit_tests"
```

**手順3：社内レジストリへのプッシュ（GitLab Container Registry使用例）**

```bash
# GitLabのレジストリにログイン
docker login registry.example.com

# タグ付けしてプッシュ
docker tag autosar/sil-builder:latest registry.example.com/autosar/builder:latest
docker push registry.example.com/autosar/builder:latest
```

**手順4：.gitlab-ci.ymlから参照**

上記YAMLサンプルの`image:`フィールドで`registry.example.com/autosar/builder:latest`を指定することで、CI実行のたびに同じ環境でビルド・テストが再現される。

### 推奨ツールスタック

| 用途 | ツール | 理由 |
|---|---|---|
| ビルド（Classic AUTOSAR） | CMake + Ninja | 標準的・高速 |
| ビルド（Linux系） | Yocto | Linuxディストリビューション管理の標準 |
| CI基盤 | GitLab CI | オンプレ対応・統合性が高い |
| ユニットテスト | Google Test / CMock | 実績多・C対応 |
| ATDDテスト | Robot Framework + CANoe Python API | 自然言語に近い記述でドメイン専門家も参加可能 |
| トレーサビリティ | Polarion ALM または GitLab Issues | 機能安全証跡管理に必要 |

---

## 3. 機能安全とアジャイルの関係

### 「アジャイルは機能安全と相容れない」は誤解

ISO 25119・ISO 26262はプロセスを規定するが**アジャイルを禁止していない**。ただしV字モデルとアジャイルの橋渡しには工夫が必要。

### Safety Backlog の考え方

通常のプロダクトバックログに加えて「Safety Backlog」を設ける。

```
通常Backlog: 機能要求・改善・バグ修正
Safety Backlog: HARA更新、安全要求トレーサビリティ更新、
               FMEA見直し、テスト証跡更新、FSAチェックリスト消化
```

スプリントレビューに「安全証跡の確認」を組み込み、各イテレーションでISO 25119適合証跡を蓄積していく。

### ISO 25119（建設機械）とアジャイルの相性

ISO 25119はISO 26262（自動車）より要件が柔軟なため、アジャイル適用のハードルは相対的に低い。ただし以下の原則は守る必要がある。

- HARAは最初のスプリント前に実施しておく（後付けは不可）
- 安全要求のトレーサビリティは毎スプリント更新する
- リリース前に独立したFSA（機能安全監査）を実施する

---

## 4. スクラム・TDD/ATDDの組み込み開発への適用

### スクラムの適用

| 項目 | 一般的なスクラム | 組み込み・機能安全向けの調整 |
|---|---|---|
| スプリント期間 | 1〜2週間 | 3〜4週間（HW確認・証跡作成の時間が必要） |
| Definition of Done | 動くソフトウェア | 動くソフトウェア＋MISRA-C違反0＋トレーサビリティ更新 |
| スプリントレビュー | デモ | デモ＋安全証跡レビュー |
| ベロシティ | ストーリーポイント | Safety Backlogコストを含めた計測 |

### TDD（テスト駆動開発）の適用

組み込みTDDの鉄則は**「ハードウェア依存を最小化したビジネスロジックから始める」**こと。

```c
/* ハードウェア層はHALで隔離し、モック化する */

/* HALインターフェース（自社定義）*/
typedef struct {
    uint16_t (*ReadSensor)(void);
    void (*WriteActuator)(uint16_t value);
} HalInterface;

/* SWCはHALを通じてのみHWにアクセス */
void HydraulicControl_10ms(const HalInterface *hal) {
    uint16_t input = hal->ReadSensor();
    uint16_t cmd = ConvertToValveCommand(input);
    hal->WriteActuator(cmd);
}

/* テスト時はモックHALを注入 */
TEST(HydraulicControl, NormalOperation) {
    MockHal mockHal = { MockReadSensor, MockWriteActuator };
    HydraulicControl_10ms(&mockHal);
    ASSERT_EQ(MockWriteActuator_LastValue(), EXPECTED_CMD);
}
```

### ATDD（受け入れテスト駆動開発）とRobot Framework

Robot Frameworkを使うと、テスト仕様を自然言語に近い形で記述できる。機械設計者・安全担当者が読めるテスト仕様書を兼ねる。

```robot
*** Test Cases ***
油圧制御_正常動作確認
    [Documentation]    オペレータ入力に対してバルブ指令値が正しく出力されることを確認
    CAN信号を送信する    message=OperatorInput    BoomLiftCmd=50.0
    10ms待つ
    CAN信号を受信する    message=HydraulicCommand
    バルブ指令値を確認する    BoomLiftCmd=50.0    tolerance=1.0

油圧制御_フェールセーフ確認
    [Documentation]    SystemStatusがFaultの場合バルブ指令値が安全値になることを確認
    CAN信号を送信する    message=SystemStatus    Status=Fault
    10ms待つ
    CAN信号を受信する    message=HydraulicCommand
    バルブ指令値を確認する    BoomLiftCmd=0.0
```

---

## 5. 建設機械での「今から始める」推奨フェーズ

### フェーズ0（0〜3ヶ月）：土台の整備

**担当：** 自社ソフトウェア担当者1名（リード）＋委託先との調整窓口  
**主な作業ツール：** GitLab、CMake、Docker Desktop  
**想定工数：** 担当者1名 × 3ヶ月（並行業務あり）

これがなければ何も始まらない。以下を完了させることで、フェーズ1以降の自動化・テスト導入の土台が整う。

- [ ] Gitリポジトリを一元化する（現状がバラバラなら統合）  
  → GitLab CE（無償）をオンプレに立てる。委託先アカウントも同一リポジトリで管理。
- [ ] ビルドスクリプトを整備する（CMake/Ninjaで再現可能なビルド）  
  → 手元のPCで `cmake -S . -B build && cmake --build build` だけでビルドできる状態にする。
- [ ] 最小CIパイプラインを構築する（push → ビルド確認だけでもよい）  
  → `.gitlab-ci.yml` に `build` ステージだけ定義し、コンパイルエラーをCIで検知できる状態にする。所要時間：1〜2日。
- [ ] コードレビュープロセスを導入する（PRベースのレビュー）  
  → GitLabのMerge Request機能を使い、mainブランチへの直接pushを禁止するブランチ保護ルールを設定する。

**フェーズ0完了の目安：** mainへのpushが自動でビルドされ、結果がGitLabのパイプライン画面で確認できる状態。

---

### フェーズ1（3〜6ヶ月）：テストの整備

**担当：** ソフトウェア担当者1〜2名  
**主な作業ツール：** Google Test、CMock、Docker、GitLab CI  
**想定工数：** 担当者1名 × 3ヶ月（新規テスト記述 + 既存コードのHAL分離に時間がかかる）

- [ ] HALを既存コードに後付けで導入する（テスト可能な構造に変える）  
  → I/O直打ちの箇所を`HalInterface`構造体経由に置き換える。一度に全部やらず、テストを書きたいSWCから着手する。
- [ ] Unit SIL環境を構築する（Google Test + CMockをDockerで動かす）  
  → 前述の `Dockerfile.sil` を使い、Dockerコンテナでテストが実行できる状態にする。所要時間：2〜3日。
- [ ] 既存コードのリバースユニットテストを書く（まず現状動作を固定する）  
  → 「今の動作が正しい」として先にテストを書き、リファクタリング時のリグレッション検知に使う。
- [ ] CIにユニットテストを組み込む  
  → `.gitlab-ci.yml` に `test:unit` ステージを追加し、テスト失敗でパイプラインが止まる状態にする。

**フェーズ1完了の目安：** Unit SILテストがCIで自動実行され、カバレッジレポートがMR画面で確認できる状態。

---

### フェーズ2（6〜12ヶ月）：プロセスの整備

**担当：** ソフトウェア担当者＋プロセス改善担当者（兼任可）  
**主な作業ツール：** GitLab（スクラムボード）、Robot Framework、Polyspace/PC-lint Plus  
**想定工数：** チーム全体として通常開発の20〜30%をプロセス整備に充てる

- [ ] スクラムを試験導入する（まず1チームから）  
  → スプリント期間は3〜4週間を推奨。GitLab Issuesのラベルで`sprint:N`を管理する。
- [ ] Safety Backlogを設ける  
  → GitLab Issuesのラベル`safety`を作成し、HARA由来のタスクを分離して追跡する。
- [ ] ATDDシナリオを主要機能（油圧制御・旋回制御）に対して整備する  
  → Robot Frameworkのテストスクリプトを10〜20ケース程度から始める。
- [ ] MISRA-Cチェックを静的解析ツールで自動化する  
  → CI YAMLの `lint:misra` ステージ（上記サンプル参照）を組み込む。

**フェーズ2完了の目安：** スクラムでスプリントが回り、MISRA-C違反ゼロがMRマージ条件として機能している状態。

---

### フェーズ3（12ヶ月以降）：機能安全証跡の自動化

**担当：** ソフトウェア担当者＋機能安全担当者  
**主な作業ツール：** Polarion ALM、CI証跡生成スクリプト、外部審査機関  
**想定工数：** 機能安全担当者1名 × 3〜6ヶ月（外部審査機関との調整を含む）

- [ ] トレーサビリティマトリクスの自動生成（要求 → コード → テスト）  
  → PolarionのAPIまたはCIスクリプトで、要求IDとテスト結果の対応表を自動生成する。
- [ ] ISO 25119証跡のCI出力への組み込み  
  → 上記YAMLサンプルの `report:evidence` ステージを参照。CIが通るたびに証跡PDFが自動生成される。
- [ ] 外部審査機関との連携  
  → TÜV SÜD / DNV などの機関に証跡パッケージを提出し、適合性評価を受ける段取りを整える。

**フェーズ3完了の目安：** CIが通るたびに機能安全証跡レポートが自動生成され、審査機関への提出資料として使える状態。

---

## 6. 外部委託先との開発プロセス連携

外部委託先がある場合、以下を契約・開発開始前に合意しておく。

| 項目 | 合意すべき内容 |
|---|---|
| ソースコード管理 | 自社GitLabリポジトリへのアクセス権と運用ルール |
| CIパイプライン | 委託先のコミットも同じCIを通すか、成果物納品時にCIで検証するか |
| テスト仕様書フォーマット | Robot Frameworkなど自社と統一するか、別途定義するか |
| 証跡の納品形式 | テストレポート・カバレッジレポートの形式と頻度 |
| レビュープロセス | PRレビューを自社担当者が行う手順 |
| Dockerイメージの共有 | 自社のSILビルドイメージを委託先も使うことで環境差異をなくす |
