CRANE+ V2 お辞儀動作の模倣学習（ACT）

Physical AI講座 最終課題

Gazebo上のCRANE+ V2をキーボードでテレオペしてデモンストレーションを収集し、ACT（Action Chunking with Transformers）で「お辞儀」動作を学習、閉ループ推論でロボットを動かしました。

- デモ動画: https://www.youtube.com/watch?v=9dT5cSdn6Ew
- ベース: [matsuolab/pai_training](https://github.com/matsuolab/pai_training)（このリポジトリはそのフォークです）

## 概要

| 項目 | 内容 |
|---|---|
| ロボット | CRANE+ V2（Gazebo シミュレーション） |
| タスク | 関節3（ひじ）を曲げて戻す「お辞儀」動作 |
| データ | キーボードテレオペで記録した rosbag 各10エピソード（`pekori`） |
| 観測・行動 | 関節角5次元（joint1〜4 + gripper）、20 Hz |
| ポリシー | ACT（LeRobot 実装） |
| 計算環境 | GPUなし、Docker コンテナ内でCPU学習・推論 |
| 実験管理 | Weights & Biases（プロジェクト: `pai_crane_gazebo`） |

## 処理の流れ

```
Gazebo (crane_plus_gazebo)
  │  キーボードテレオペ (physicalai_teleop)
  ▼
rosbag2 記録 (/joint_states, arm/gripper の joint_trajectory)
  │  pixi run convert
  ▼
LeRobotDataset v2.1
  │  pixi run train (ACT)
  ▼
学習済みポリシー
  │  pixi run infer（/joint_states を購読し、joint_trajectory を 20 Hz で発行）
  ▼
Gazebo 上の CRANE+ V2 が閉ループでお辞儀
```

## 元リポジトリからの主な変更点

| ファイル | 変更内容 |
|---|---|
| `scripts/infer.py` | LOOKAHEAD=2 として、予測アクション列の先読み位置を調整し、閉ループ動作を安定化
| `scripts/train.py` | 学習ステップ数200を2000に変更
| `configs/train_config.yaml` | 学習ステップ数200を2000に変更
| `pyproject.toml`, `pixi.lock` | wandb ログイン

各変更の差分はコミット履歴から確認できます。

## 動作環境

- Docker イメージ: 書籍『ROS 2とPythonで作って学ぶAIロボット入門 改訂第2版』の公式イメージ
  [`airobotbook/ros2-desktop-ai-robot-book-humble`](https://github.com/AI-Robot-Book-Humble/docker-ros2-desktop-ai-robot-book-humble)（ROS 2 Humble）
- CRANE+ V2 モデル / Gazebo: [AI-Robot-Book-Humble/crane_plus](https://github.com/AI-Robot-Book-Humble/crane_plus)（`airobot` ブランチ `19c1d60`、変更なし。上記イメージに同梱）
- テレオペ: [matsuolab/physicalai_teleop](https://github.com/matsuolab/physicalai_teleop)（`0ef2a00`、変更なし）
- 学習・推論: 本リポジトリ（pixi 環境、lerobot サブモジュール `768e366`）

## 再現手順

### 1. コンテナ起動

```bash
docker pull airobotbook/ros2-desktop-ai-robot-book-humble
docker run -it --name pai-ros2 \
  -p 6080:80 \
  -p 9090:9090 \
  -p 9876:9876 \
  airobotbook/ros2-desktop-ai-robot-book-humble /bin/bash
```

- 6080: ブラウザから VNC でデスクトップを操作（`http://127.0.0.1:6080`、パスワード `ubuntu`）
- 9090 / 9876: データセット可視化（Rerun）用

※ 作者は上記イメージに以下の手順を加えたものを `docker commit` して使用しました。

### 2. 本リポジトリとテレオペパッケージの準備（コンテナ内）

```bash
# 学習・推論環境
cd ~
git clone --recursive https://github.com/ONODERA446/pai_training.git
cd pai_training
curl -fsSL https://pixi.sh/install.sh | bash
source ~/.bashrc
pixi install

# テレオペ
mkdir -p ~/pai_ws/src && cd ~/pai_ws/src
git clone https://github.com/matsuolab/physicalai_teleop.git
cd ~/pai_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install --packages-select physicalai_teleop
source install/setup.bash
```

### 3. Gazebo 起動

```bash
ros2 launch crane_plus_gazebo crane_plus_with_table.launch.py
```

### 4. テレオペしながら rosbag を記録（1エピソードごと）

ターミナル1（記録）:

```bash
ros2 bag record -a -o ~/dataset/rosbags/pekori/ep1    # ep1, ep2, ... とエピソードごとに変える
```

ターミナル2（操作）:

```bash
cd ~/pai_ws && source install/setup.bash
ros2 run physicalai_teleop teleop_keyboard
```

主なキー: `Q/E` 関節選択、`W/S` 選択した関節の角度を増減、`O/C` グリッパー開閉、`X` 初期姿勢、`ESC` 終了。
動作が終わったら、ターミナル1を `Ctrl+C` で止めます。これを10エピソード分繰り返します。

### 5. LeRobot 形式へ変換

`configs/conversion_config.yaml` の `bag_dir` / `output_dir` と、`configs/crane_features.py` の `repo_id` / `root` / `task_name` を対象データセットに合わせてから実行します。

```bash
cd ~/pai_training
pixi run convert
```

### 6. 学習

設定は `configs/train_config.yaml` です。

```bash
pixi run wandb login
pixi run train
```

### 7. 推論（Gazebo を起動した状態で）

```bash
pixi run infer --model-path /home/ubuntu/checkpoints/train/crane_plus_pekori_act
```

## 結果

お辞儀ができた。しかし、お辞儀の後に開かせたかったクリッパーは期待通りではなかった。
（理由：学習データ作成時のクリッパ開きコマンドの入力が推論チャンク時間の0.8秒以上かかってしまった為と考えられる）。



## データセット・学習済みモデル

容量の都合で、rosbag・データセット・チェックポイントはリポジトリに含めていません。

## Credits / License

- [matsuolab/pai_training](https://github.com/matsuolab/pai_training)（Tatsuya Kamijo）: rosbag 変換、学習・推論スクリプトのベース。各スクリプトは Apache License 2.0 のヘッダに従います。変更したファイルには変更した旨を記載しています。
- [huggingface/lerobot](https://github.com/huggingface/lerobot): ACT 実装（Apache License 2.0）
- [matsuolab/physicalai_teleop](https://github.com/matsuolab/physicalai_teleop): キーボードテレオペ（MIT）
- [AI-Robot-Book-Humble](https://github.com/AI-Robot-Book-Humble): 書籍『ROS 2とPythonで作って学ぶAIロボット入門 改訂第2版』（出村・萩原・升谷・タン著、講談社）の Docker 環境と crane_plus（元は [rt-net/crane_plus](https://github.com/rt-net/crane_plus)、Apache License 2.0）

