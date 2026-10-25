# CRANE+ V2 模倣学習：Hello World を見せる 2 つのタスク

Gazebo シミュレータ上の CRANE+ V2 アームを、キーボードで遠隔操作（テレオペ）して集めたお手本から模倣学習（ACT）し、
学習したモデルだけで「隠された Hello World 看板を見せる」動作を再現しました。
隠し方の違う 2 つのタスクに取り組んでいます。

| | タスク | 動作 |
|---|---|---|
| 1 | 目隠しを倒す（`01_hello_world_cover/`） | 看板の前に立てた目隠しの板を、外側へ押し倒す |
| 2 | 襖を開ける（`02_hello_fusuma/`） | 看板の前の襖のつまみを横に押して、スライドさせて開ける |

## 共通の構成

- ロボット: CRANE+ V2（ROS 2 Humble / Ignition Gazebo）
- 学習手法: ACT（LeRobot）
- 入力: 関節角度のみ（カメラ画像は使用していません。ステージの配置を毎回固定しているため）
- 主な学習設定: training_steps 3000 / batch_size 8 / device cpu

### 使用した主なリポジトリ

- テレオペ: [matsuolab/physicalai_teleop](https://github.com/matsuolab/physicalai_teleop)
- 変換・学習・推論: [matsuolab/pai_training](https://github.com/matsuolab/pai_training)
- シミュレータ: [rt-net/crane_plus](https://github.com/rt-net/crane_plus)

## このリポジトリの内容

```
01_hello_world_cover/        タスク1：目隠しを倒す
  stage/
    hello_sign.sdf           Hello World 看板（固定）
    cover.sdf                目隠しの板
    spawn_stage.sh           看板と目隠しを出現させる
    reset_stage.sh           目隠しを元の位置に戻す
    textures/                看板の文字画像
  configs/                   pai_training の設定ファイル（タスク1用）

02_hello_fusuma/             タスク2：襖を開ける
  stage/
    hello_sign.sdf           Hello World 看板（固定）
    fusuma.sdf               襖（レールに沿って左右にだけ動く。上端につまみ付き）
    spawn_fusuma.sh          看板と襖を出現させる
    reset_fusuma.sh          襖を閉じた状態に戻す（消して出し直す）
    textures/                看板の文字画像・襖の柄
  configs/                   pai_training の設定ファイル（タスク2用）
```

ステージのファイルは、コンテナ内の `/workspace/stage/` に置いて使う前提です。

## 実行手順（共通）

### 1. シミュレータの起動とステージの配置

```bash
ros2 launch crane_plus_gazebo crane_plus_with_table.launch.py
/workspace/stage/spawn_stage.sh       # タスク1（別ターミナルで）
/workspace/stage/spawn_fusuma.sh      # タスク2（別ターミナルで）
```

### 2. テレオペでお手本を記録

```bash
ros2 run physicalai_teleop teleop_keyboard
ros2 bag record -a -o ~/dataset/rosbags/<フォルダ名>/ep0   # ep1, ep2, ... と増やす
```

1 回ごとに、テレオペの `X`（直立に戻す）→ リセット用スクリプト → 記録開始 → 操作 → 記録停止、を繰り返しました。
お手本のばらつきをなくすため、操作のキーと回数をあらかじめ決めて、毎回同じ手順で操作しています。

### 3. 変換・学習

`pai_training` の `configs/` を、各タスクの `configs/` に置き換えて実行します。

```bash
cd ~/pai_training
pixi run convert
WANDB_MODE=disabled pixi run train
```

### 4. 推論（再現）

テレオペを終了してから実行します。

```bash
pixi run infer --model-path /home/ubuntu/checkpoints/train/<モデルのフォルダ名>
```

## タスク1：目隠しを倒す

- 記録したエピソード数: 25 回
- 操作手順:  
    1.joint2 を選択 → S を 10 回  
    2.joint3 を選択 → S を 43 回  
    3.joint2 を選択 → W を 6 回  

## タスク2：襖を開ける

- 記録したエピソード数: 25 回
- 操作手順:  
    1.joint3 を選択 → S を 34 回  
    2.joint1 を選択 → W を 15 回  

## 注意

- 記録データ（rosbag）と学習済みモデルは、容量が大きいため含めていません。
- 襖の柄・看板の画像はオリジナルで作成したものです。
