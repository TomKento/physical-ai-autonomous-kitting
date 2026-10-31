# AI-driven Autonomous Kitting System

**QR → SOP → Qwen → Task Validator → AGV模擬搬送 → Franka Pandaのキッティング**

日本語で記述された製造作業指示をQwenが構造化し、独立したPython Validatorで実行可能性を検査してから、PyBullet上のFranka Pandaが指定区画に部品を配置するシミュレーションです。**同じロボット制御プログラムでProduct A/Bの配置先を切り替える**ことを検証しました。

## 本実証の位置づけとLLMの適用限界

本作品の研究目的は、**自然言語の定型SOPをLLMで構造化し、独立したValidatorと既存のロボット制御を組み合わせた構成が成立するか**を確認することです。実験で確認した柔軟性は、同じ制御ロジックでProduct A/Bの**配置先を切り替えたこと**に限られます。工程数、同一部品の数量、未知の部品や自由記述への対応は実証していません。

**LLMの必要性・従来方式に対する優位性は未実証です。** 今回の定型SOPは、実際に`verify_against_sop()` が正規表現で部品色と配置先を抽出できるため、登録済みレシピやルールベース方式でも同様の処理が可能です。したがって、LLMによって従来方式では不可能だった品種切替を実現したとは主張しません。

今後は、定型／非定型の作業指示、文脈依存、曖昧な指示、欠品などの条件下で、**ルールベース・Qwen・国産LLM（Sarashina）**を精度、誤実行、確認工数、遅延、総コストの観点から比較し、LLMの採用が合理的になる境界条件を検証します。VLMによる現場認識・再計画への拡張は、その先の研究課題です。

## 何が実際に動くのか

```text
Work-order ID（QR画像）
    ↓ OpenCVによるデコード
SOP-001 / SOP-002（ローカル定義）
    ↓ Qwen2.5-0.5B-Instruct（Colab CPUで推論）
構造化計画 {object_color, slot} × 3工程
    ↓ JSON整形・日本語正規化・SOP原文照合・Preflight
AGV模擬搬送・定位置ドッキング
    ↓ 部品座標を既知の把持位置に合わせる
Franka Panda（PyBullet／IK＋従来型制御）
    ↓ Pick → Lift → Transport → Place → Vertical Retreat
結果検証（配置・区画占有・既配置部品の変位）
```

Notebookの処理順では、**AGVドッキング後にQR読取・Qwen推論**を行い、その後ロボットを動かします。上図は機能の構成を示します。

> **Demo disclosure**：本プロジェクトのAGVは、車輪の物理制御やSLAMを持たない運動学的な模擬搬送です。QRはローカルPNGをOpenCVで読み取り、PyBullet内の仮想カメラから読み取っていません。AIは自然言語からタスクを抽出し、ロボットの関節角を直接生成しません。実演映像はシミュレーションで記録した結果を編集したものです。

## 実測結果（2026-10-29記録、各1試行）

記録ID：`run_20261029T124105Z`。**成功率の一般化や統計的信頼性を示すものではありません。**

| 指標 | Product A | Product B |
| --- | --- | --- |
| Work-order ID | `WO-2026-A` | `WO-2026-B` |
| 指示された配置 | red→A1, blue→B1, green→A2 | red→A1, blue→A2, green→B1 |
| 3工程Pick & Place | **3/3成功** | **3/3成功** |
| 配置誤差 red (mm) | 14.24 | 14.24 |
| 配置誤差 blue (mm) | 8.68 | 12.95 |
| 配置誤差 green (mm) | 4.58 | 1.80 |
| 配置誤差 平均 (mm) | 9.17 | 9.66 |
| 既配置部品の最大変位 (mm) | 1.20 | 1.20 |
| Qwen推論時間・3工程合計 (秒) | 16.65 | 19.22 |
| 異常指示の事前検証 | 欠品と占有を拒否 | 欠品と占有を拒否 |

配置誤差は配置先中心からのXY距離、状態変位は先行工程で配置した部品の基準位置から後続工程終了時までの距離です。小数値は表示用に四捨五入しており、正確な値は各manifestに記録しています。

### Product A / Product B の違い

試験したのは同じ3色の部品1個ずつの「配置先変更」です。赤2個・青1個／赤1個・緑2個という部品点数変更は実装・検証していません。品種切替への拡張可能性を示す限定的なPoCです。

## 起動方法（Google Colab）

1. [`notebooks/PAI2026_Autonomous_Kitting_Public.ipynb`](notebooks/PAI2026_Autonomous_Kitting_Public.ipynb) をGoogle Colabで開く。
2. Colab Pythonランタイムで上から **「すべてのセルを実行」**。インストール・Qwenモデルの取得が発生します。
3. `SAVE_TO_DRIVE = True`（既定）の場合はDriveマウントを許可。保存したくない場合はFalseへ変更。
4. 最終的な `ALL PRODUCT CAPTURES SUCCESS: True` を確認。新しい日時フォルダが作られ、A/B各々のQR画像、AGV MP4、ロボット MP4、実行JSONと `dashboard_index.json` が保存されます。
5. Qwen初回ダウンロードやColab実行環境により所要時間と結果が異なります。再実行で同じ誤差を保証するものではありません。

記録時の環境：Google Colabクラウド／**CPU**、Python **3.13.18**、PyBullet **3.2.7**、NumPy **2.1.3**、PyTorch **2.11.0+cpu**、Transformers **4.57.6**。モデルは `Qwen/Qwen2.5-0.5B-Instruct`。**外部推論APIは使用しませんが、モデル取得にネットワーク接続を使用します。** Notebookは依存ライブラリのバージョン範囲を指定しており、完全な再現性を保証するロックファイルではありません。

## リポジトリ構成

```text
notebooks/PAI2026_Autonomous_Kitting_Public.ipynb  # 実行コード
assets/dashboard_index.json                         # 再生用の索引
assets/PRODUCT_A/manifest_PRODUCT_A.json            # AI/Validator/実測ログ
assets/PRODUCT_A/qr_PRODUCT_A.png                   # OpenCV読取画像
assets/PRODUCT_A/agv_docking_PRODUCT_A.mp4          # AGV模擬搬送（録画）
assets/PRODUCT_A/robot_sorting_PRODUCT_A.mp4        # Pandaの実行映像（録画）
assets/PRODUCT_B/...                                # 同様に製品B
reports/PAI2026_Final_Report_with_LLM_Appendix_v2.pdf  # 5ページの実証レポート
```

`assets/dashboard_index.json` のパスは `assets/` 起点の相対パスです。各manifestの `robot_execution.timeline_events[].video_time_s` は**ロボットMP4内の時刻**を指し、AGV・Qwen・QRと共通の単一時系列を計測したものではありません。

## 検証方法と実験の限界

- **Qwen**：SOPの各工程から色・配置先を出力。JSONコードフェンスを除去し、許容値と原文の定型表現を照合します。
- **Validator**：欠品・配置先占有でロボットを動かさず停止するスモークテスト（各製品1回）を確認。実機の機能安全や未知指示一般への対応は未評価です。
- **Panda**：IK・関節位置制御を使用。既配置部品への干渉による大きな変位を、配置後の垂直退避で改善しました。
- **環境**：部品の色・出発座標、区画位置をシミュレータ内で既知としています。カメラ認識・VLM・VLA学習は実装していません。
- **AGV**：車輪力学・積載荷重・SLAM・障害物回避・本物のQR搭載カメラは未実装です。部品位置は台車と同期してプログラムで移動し、低床荷台は視覚用モデルです。
- **結果**：Product AとBの成功ログはそれぞれ1試行です。推論時間はColab CPUでの3工程合計で、モデルロード時間は含めていません。

### 参考評価（主系統の動作確認とは別実験）

日本語指示12問でQwenとSarashina2.2-0.5B-Instructを比較した探索的評価では、厳密な3項目一致でQwen **6/12**、Sarashina **8/12**。明確な指示ではQwen 6/6、Sarashina 5/6、曖昧な指示ではQwen 0/6、Sarashina 3/6でした。これは小規模・プロンプト調整を経た評価であり、統計的なモデル優劣の証明ではありません。ロボットに接続した主モデルはQwenです。 詳細は最終レポート5ページ目のAppendixに掲載しています。なお `reports/PAI2026_Final_Report_SOURCE.html` は当初の4ページ本文のソース、`reports/PAI2026_Appendix_Model_Comparison_SOURCE.html` はAppendixのソースです。配布PDFは両者を結合した5ページ版です。

## 実演動画・ライセンス

- **実演動画**：[YouTube — AI-driven Autonomous Kitting System](https://youtu.be/iKiY20Zqvz4)（2分25秒の録画・編集デモ）。
- 本リポジトリにはライセンスを明示していません。再利用する場合は、ソフトウェアおよび素材の権利条件をご確認ください。

**Provenance**：Colab上で生成された実行結果を無加工で `assets/` に同梱しています。制御コードの主要部分はColabで動作確認済みのNotebook v2から移植し、公開版では出力・実行履歴を消去しただけです。作業過程では生成AIの開発支援を利用しています。
