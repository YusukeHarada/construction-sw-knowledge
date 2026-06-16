# 車載ソフトウェア開発ツールランドスケープ

- **作成日：** 2026年6月17日
- **対象読者：** 開発エンジニア・開発プロセス担当者
- **資料種別：** 知識整理メモ
- **ステータス：** Draft

---

## はじめに

本資料は、自動車・建設機械向け組み込みソフトウェア開発で使われる主要ツールの全体像を整理したものである。ツール名・ベンダー名・国内販売代理店・特徴を記載し、ツール選定時の参考情報として活用する。

> 記載価格・バージョン・ベンダー情報は参考値であり、採用前に必ずベンダーへ最新情報を確認すること（「要確定」と付記した項目は特に注意）。

---

## 1. ALM（Application Lifecycle Management）/ 要求管理

要求・設計・テストの一貫管理と変更履歴の追跡に使う。ASPICE・ISO 26262ではトレーサビリティの証跡が必須であり、ALMツールの選定は開発プロセス全体に影響する。

### 主要ツール

| ツール名 | ベンダー | 国内販売 | 特徴 |
|---|---|---|---|
| **CodeBeamer ALM** | PTC（旧 Intland Software） | PTC Japan | 現在の車載業界デファクト。ASPICE・ISO 26262対応テンプレート内蔵。Jira連携あり |
| **IBM DOORS / DOORS Next** | IBM | 日本IBM | かつてのデファクト標準。DXL言語でカスタマイズ可能。レガシー案件・大手OEMでまだ多い |
| **Polarion ALM** | Siemens | シーメンス | CodeBeamerと競合する有力製品。SVN/Gitとの統合が強み |
| **Jira + Confluence** | Atlassian | アトラシアン（代理店多数） | アジャイル開発での採用増加。機能安全対応プラグイン（Xray等）で補完可能だが証跡管理に限界 |
| **PTC Integrity** | PTC | PTC Japan | 旧 MKS Integrity。現在はCodeBeamerに統合・移行が推奨されており、新規採用は少ない |

### 選定の考え方

- **ASPICE・ISO 26262対応が最優先：** 要求→設計→テストのトレーサビリティを自動生成できるか確認する
- **委託先との共有：** クラウド型（SaaS）ならライセンス管理が容易。委託先にゲストアクセスを付与できるかも確認
- **移行コスト：** DOORS資産をDOORS Nextへ、またはCodeBeamerへ移行するケースが増えているが、移行工数は大きい（要確定）
- **建設機械メーカーの現実解：** CodeBeamer（クラウド）またはPolarion。Jiraは社内タスク管理との兼用ならコスト効率が良いが、機能安全証跡には追加設計が必要

---

## 2. 静的解析・コーディング規約チェック

MISRA-C準拠の確認、未定義動作の検出、セキュリティ脆弱性の静的発見に使う。

### 主要ツール

| ツール名 | ベンダー | 国内販売 | 特徴 |
|---|---|---|---|
| **Helix QAC** | Perforce | 東陽テクニカ | 自動車業界での採用実績多数。MISRA-C/C++対応。AUTOSAR Coding Guidelines対応（要確定） |
| **PC-lint Plus** | Gimpel Software | 直販 | MISRA-C対応の老舗。軽量でCI組み込みが容易。GUIは貧弱 |
| **Klocwork** | Perforce | 東陽テクニカ | 大規模コードベース向け。CI統合容易。セキュリティ脆弱性検出も得意 |
| **Polyspace Bug Finder / Code Prover** | MathWorks | MathWorks Japan | MISRA-C・ISO 26262対応。Simulinkとの統合が強み。形式検証（Code Prover）で実行時エラーを網羅的に検証 |
| **LDRA TBvision** | LDRA | 直販・代理店（要確定） | MISRA-C準拠チェック＋コードカバレッジ計測の統合ツール |
| **SonarQube + 車載プラグイン** | Sonarsource | 代理店多数 | OSSベースでCI統合しやすい。車載向けルールセットはプラグイン追加が必要 |

### 選定の考え方

- **MISRA-C 2012対応と証跡出力：** ISOスタンダードに準拠した違反レポートをそのまま審査証跡に使えるか確認
- **CI/CDパイプライン統合：** コマンドライン実行・JUnit形式レポート出力があるかどうかがポイント
- **コスト：** SonarQubeは初期コストが低いが、車載MISRA対応の精度はHelix QAC・Polyspaceに劣る

---

## 3. 単体テスト・コードカバレッジ

ISO 26262ではASIL Bから構造カバレッジ（MC/DC）の証跡が要求される。ツールによって計測可能なカバレッジ種別が異なる。

### 主要ツール

| ツール名 | ベンダー | 国内販売 | 特徴 |
|---|---|---|---|
| **VectorCAST** | Vector | ベクター・ジャパン | 業界シェアが高い。MC/DC計測対応。スタブ自動生成。ASIL D対応の証跡生成 |
| **C/C++test** | Parasoft | テクマトリックス（国内販売） | MISRA-C検査＋単体テスト自動生成を統合。コードカバレッジ・静的解析の一体管理が可能 |
| **LDRA TBrun** | LDRA | 直販・代理店（要確定） | カバレッジ計測と単体テスト実行を統合。TBvisionと組み合わせて使うことが多い |
| **WinAMS** | アドヴァンスソフト | アドヴァンスソフト（直販） | 国内自動車業界で採用事例多数。組み込みC向け。ルネサスマイコン環境との親和性が高い |
| **Polyspace Test** | MathWorks | MathWorks Japan | MathWorksエコシステム内で静的解析・単体テスト・カバレッジを完結できる |
| **Google Test / CppUTest** | OSS | — | 無償で使えるが、機能安全証跡の生成には追加の仕組みが必要。ASIL対応ツールとして認定されていない |

### カバレッジ種別の目安（参考）

| カバレッジ種別 | ISO 26262での要求ASIL |
|---|---|
| Statement Coverage（ステートメント網羅） | ASIL A相当 |
| Branch Coverage（分岐網羅） | ASIL B・C相当 |
| MC/DC（修正条件判定網羅） | ASIL C・D相当 |

> 適用ASILはシステム設計による。上記は参考値であり、実プロジェクトでは安全計画書に基づいて決定すること。

---

## 4. 統合テスト・HIL（Hardware-in-the-Loop）

実ECUまたはモデルをリアルタイム環境に接続し、通信・タイミング・フェールセーフを検証する。

### 主要ツール

| ツール名 | ベンダー | 国内販売 | 特徴 |
|---|---|---|---|
| **CANoe / CANalyzer** | Vector | ベクター・ジャパン | 業界標準。CAN/LIN/FlexRay/Ethernet/SOME/IPに対応。AUTOSAR環境のシミュレーション・テスト自動化が可能 |
| **dSPACE SCALEXIO / DS1202** | dSPACE | dSPACE Japan | HILシミュレーターの最有力。AUTOSAR対応が強い。建設機械・農機での採用実績もあり（要確定） |
| **ETAS LABCAR** | ETAS（ボッシュグループ） | ETAS Japan | AUTOSAR BSWとの統合が得意。ボッシュ系Tier1での採用が多い |
| **NI VeriStand / NI TestStand** | National Instruments（NI） | 日本ナショナルインスツルメンツ | テスト自動化フレームワーク。LabVIEWとの統合。汎用性が高くコスト感は中程度（要確定） |
| **Speedgoat** | Speedgoat | 直販・代理店（要確定） | Simulinkとの親和性が高いリアルタイムターゲット。Simulink Real-Time環境でそのまま使える |

### 選定の考え方

- **CANoe はデファクト：** 通信プロトコルのシミュレーション・テストにはまず CANoe を検討する
- **HIL環境のコスト：** dSPACE SCALEXIO は高コスト（参考値：数百万〜数千万円規模、要確定）。規模・ASIL要求に見合う投資かを判断する
- **建設機械での活用：** J1939やCANオープンのバスシミュレーションに CANoe が有効。HILは油圧・電気系の模擬環境構築が必要になる

---

## 5. バージョン管理・CI/CD

ソースコード・ArXMLモデル・テスト証跡の構成管理と、ビルド・テストの自動化に使う。

### 主要ツール

| ツール名 | ベンダー | 特徴 |
|---|---|---|
| **Git + GitLab** | GitLab Inc. | 現在の主流。GitLab CIでパイプライン構築。オンプレ版（GitLab EE）もあり機密情報管理に対応 |
| **Git + GitHub Actions** | GitHub（Microsoft） | クラウドCI/CD。オープンソースプロジェクトとの相性が良い。機密コードはオンプレRunnerで対応 |
| **Perforce Helix Core** | Perforce | 大容量バイナリ（Simulinkモデル・ArXMLファイル等）の管理に強い。大手OEM・Tier1での採用が多い |
| **Jenkins** | OSS | オンプレCI。レガシー案件でまだ多い。柔軟性は高いがメンテナンスコストがかかる |
| **Artifactory** | JFrog | アーティファクト管理。SBOM管理との統合が進む。車載向けセキュリティスキャンとの連携も可能 |

### 選定の考え方

- **GitはもはやデファクトだがSimulinkモデルに注意：** バイナリファイルはGit LFSまたはPerforce Helix Coreで管理する
- **オンプレ vs クラウド：** 機密性の高い制御ソフトはオンプレGitLabが現実解。GitHubはself-hosted Runnerで対応可能
- **CI設計のポイント：** ビルド→静的解析→単体テスト→カバレッジレポートをパイプライン化し、証跡を自動生成する

---

## 6. AUTOSARコンフィギュレーター・コード生成

Classic AUTOSARのBSW（基礎ソフトウェア）を設定し、ArXMLからCコードを生成するツール。BSWベンダーと密接に連携するため、BSW選定とセットで検討する。

### 主要ツール

| ツール名 | ベンダー | 国内販売 | 特徴 |
|---|---|---|---|
| **DaVinci Configurator Pro** | Vector | ベクター・ジャパン | 国内外でシェアが高い。Vector BSW（MICROSAR）とセット運用が基本 |
| **EB tresos Studio** | Elektrobit（Tier1） | 代理店（要確定） | 欧州Tier1での採用が多い。EB BSWとのセット提供 |
| **ISOLAR-AB / ISOLAR-EVE** | ETAS（ボッシュグループ） | ETAS Japan | ボッシュ系。AUTOSARフルスタック（RTA-BSW）を提供。ISOLAR-EVEで仮想ECU評価が可能 |
| **AUTOSAR Builder** | Siemens EDA（旧 Mentor Graphics） | シーメンス | Siemens EDAエコシステムとの統合。要確定 |

### 選定の考え方

- **BSWとコンフィギュレーターはセット選定が現実的：** ツール単体では動かない。BSWベンダーのバンドルを基本とする
- **国内採用実績：** Vector DaVinci Configurator Pro + MICROSAR の組み合わせが国内案件で最も実績が多い（参考情報）
- **ArXMLの互換性：** ツール間でArXMLの解釈に差異があるため、委託先が使うツールと合わせることが重要

---

## 7. モデリング・アーキテクチャ設計

システムアーキテクチャ・ソフトウェア設計をモデルで表現し、設計書・コードとのトレーサビリティを確保する。

### 主要ツール

| ツール名 | ベンダー | 国内販売 | 特徴 |
|---|---|---|---|
| **Enterprise Architect（EA）** | Sparx Systems | チェンジビジョン（代理店） | UML/SysMLモデリング。コストが比較的低め。AUTOSAR向け拡張プロファイルあり |
| **IBM Rhapsody** | IBM | 日本IBM | 組み込み・リアルタイム向けUML/SysML。コード生成機能あり。AUTOSAR統合拡張あり（要確定） |
| **Capella** | Eclipse Foundation（元 Thales） | OSS（無償） | MBSE（モデルベースシステムズエンジニアリング）ツール。ARCADIA手法と組み合わせる。国内採用は増加中 |
| **MagicDraw / Cameo** | No Magic（Dassault Systèmes） | 代理店（要確定） | SysML・UPDM対応。Dassault 3DEXPERIENCEとの統合が進む |

### ArXMLエディタについて

AUTOSARのシステム記述はArXML（XML形式）で管理される。コンフィギュレーターのGUIで編集するのが基本だが、差分管理・レビューにはテキストエディタ＋スキーマ検証ツールの組み合わせも有効。

---

## 8. 変更管理・トレーサビリティ

設計変更の影響範囲分析と、要求↔設計↔コード↔テストの一貫したトレーサビリティ確保に使う。

### 主要ツール

| ツール名 | ベンダー | 特徴 |
|---|---|---|
| **Jira** | Atlassian | 変更チケット管理。CodeBeamer・Polarionと連携可能 |
| **CodeBeamer ALM** | PTC | 要求・変更・テストのトレーサビリティを一元管理。前述「1. ALM」を参照 |
| **Polarion ALM** | Siemens | 同上 |
| **Confluence** | Atlassian | ドキュメント管理。Jira連携。設計書の版管理に使われることが多い |
| **ServiceNow** | ServiceNow | 大企業での変更管理・インシデント管理。IT系の変更管理が主目的 |

### トレーサビリティ自動化のポイント

- **要求↔テストケースの自動紐付け：** CodeBeamerやPolarionはテスト実行結果を要求IDに自動連携できる
- **コード↔テストの連携：** VectorCASTやC/C++testはJUnit形式でレポートを出力し、CI/CDパイプライン経由でALMに連携できる
- **変更影響分析：** 要求が変更された際に影響するテストケース・コードを自動抽出できるか、ALMツール選定時に確認する

---

## 9. セキュリティ・脆弱性管理

車載サイバーセキュリティ（ISO/SAE 21434・UN-R155対応）ではSBOM管理とOSS脆弱性の継続的監視が必要。建設機械でも将来的に要件化される可能性がある。

### 主要ツール

| ツール名 | ベンダー | 国内販売 | 特徴 |
|---|---|---|---|
| **Black Duck** | Synopsys | シノプシス・ジャパン | SBOM生成・OSSライセンス管理・脆弱性スキャンの統合ツール。車載業界での採用実績が多い |
| **Snyk** | Snyk | 代理店あり | OSS依存関係の脆弱性スキャン。CI統合が容易。スピード重視の開発チームに向く |
| **Veracode** | Veracode | 代理店（要確定） | SAST（静的アプリセキュリティテスト）。クラウド型で導入が容易 |
| **Checkmarx** | Checkmarx | 代理店（要確定） | SAST・OSS解析・IaCスキャンの統合プラットフォーム |
| **FOSSA** | FOSSA | 直販 | OSSライセンス管理・SBOM生成に特化。CI/CD連携が容易 |

### 建設機械での優先度

現時点では車載（乗用車）ほど規制が厳しくないが、ConnectedなOTAアップデートを導入する場合は早期にSBOM管理体制を整えることを推奨する。

---

## 10. ツール選定の現実解：予算規模別の組み合わせ例

### 小規模（チーム5名以下・初期投資を抑えたい）

| カテゴリ | 採用ツール |
|---|---|
| ALM | Jira + Confluence（機能安全対応はJiraプラグインで補完） |
| 静的解析 | PC-lint Plus または SonarQube |
| 単体テスト | Google Test / CppUTest（証跡は別途整備） |
| バージョン管理/CI | Git + GitLab（無償枠またはセルフホスト） |
| 通信テスト | CANalyzer（CANoeより低コスト） |

> 注意：機能安全（ISO 26262）要件がある場合、OSSツールだけでは証跡の認定が困難になる。ASILレベルに応じて認定済みツールへの投資を検討する。

### 中規模（チーム10〜30名・ASPICE Level 2〜3を目指す）

| カテゴリ | 採用ツール |
|---|---|
| ALM | CodeBeamer ALM（クラウド版） |
| 静的解析 | Helix QAC または Polyspace Bug Finder |
| 単体テスト | VectorCAST または C/C++test |
| バージョン管理/CI | Git + GitLab（オンプレ）+ Jenkins または GitLab CI |
| 通信・統合テスト | CANoe |
| AUTOSARコンフィギュレーター | DaVinci Configurator Pro + MICROSAR |

### 大規模（OEM・大手Tier1・フルASPICE対応）

| カテゴリ | 採用ツール |
|---|---|
| ALM | IBM DOORS Next または Polarion ALM |
| 静的解析 | Polyspace Code Prover（形式検証） |
| 単体テスト | VectorCAST + LDRA TBrun |
| バージョン管理 | Perforce Helix Core（バイナリ管理）+ Git（ソースコード） |
| CI/CD | Jenkins + Artifactory |
| HIL | dSPACE SCALEXIO + CANoe |
| AUTOSARコンフィギュレーター | DaVinci Configurator Pro または EB tresos |
| セキュリティ | Black Duck + Veracode |

---

## 11. OSSで代替できるもの・できないもの

| カテゴリ | OSSで代替できる | OSSでは困難（理由） |
|---|---|---|
| バージョン管理 | Git（デファクト） | — |
| CI/CDパイプライン | GitLab CI / Jenkins | — |
| 単体テストフレームワーク | Google Test / CppUTest | ASIL認定ツールの証跡が得られない |
| 静的解析 | SonarQube（基本機能） | MISRA-C準拠の精度・証跡品質でHelix QAC等に劣る |
| ALM | Redmine / GitLab Issues | トレーサビリティ・機能安全対応テンプレートが不足 |
| HIL環境 | — | dSPACE等の専用ハードウェアの代替は現実的に困難 |
| AUTOSARコンフィギュレーター | — | BSW込みのツールチェーンはベンダー製品が必須 |
| SBOM管理 | FOSSA（一部無償） | 大規模・継続監視には有償ツールが現実的 |

---

## 12. 建設機械メーカーとして優先度の高いカテゴリ

現状の課題（仕様・設計・テスト不在）を踏まえた優先順位の考え方。

1. **ALM（要求管理）：** 仕様が文書化されていない現状の改善が最優先。まずCodeBeamerまたはJiraで要求を整理する
2. **静的解析：** 既存コードの品質把握にHelix QACまたはPC-lint Plusを導入し、MISRA-C違反の現状を把握する
3. **バージョン管理・CI：** Git + GitLabを基盤として、ビルドの再現性を確保する
4. **単体テスト：** 制御系コア（安全クリティカル）から優先的にVectorCASTまたはWinAMSで単体テストを整備する
5. **通信テスト：** J1939/CAN通信の検証にCANoe / CANalyzerを導入する
6. **HIL：** 導入コストが大きいため、委託先が保有するHIL環境を活用するか、段階的導入を検討する

---

## 参考：主要ベンダー連絡先（国内）

| ベンダー | 主な製品 | 国内窓口 |
|---|---|---|
| ベクター・ジャパン | CANoe, CANalyzer, DaVinci, MICROSAR | https://www.vector.com/jp/ |
| dSPACE Japan | SCALEXIO, TargetLink | https://www.dspace.com/ja/ |
| MathWorks Japan | MATLAB/Simulink, Polyspace | https://jp.mathworks.com/ |
| テクマトリックス | C/C++test（Parasoft国内販売） | https://www.techmatrix.co.jp/ |
| 東陽テクニカ | Helix QAC, Klocwork（Perforce国内販売） | https://www.toyo.co.jp/ |
| MathWorks Japan | Polyspace, Simulink Test | https://jp.mathworks.com/ |
| ETAS Japan | ISOLAR, LABCAR, RTA-BSW | https://www.etas.com/ja/ |
| アドヴァンスソフト | WinAMS | https://www.advancesoft.jp/ |
| PTC Japan | CodeBeamer ALM | https://www.ptc.com/ja/ |

> 上記URLは参考情報。アクセス前に最新情報を確認すること。

---

*本資料は知識整理メモであり、特定製品の推奨ではない。ツール導入にあたっては各ベンダーへの問い合わせおよび社内評価を必ず実施すること。*
