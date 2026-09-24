# 训练样本格式

GVHMR + Holosoma 物体交互重定向的输出对齐到本格式后，才能作为 TWIST 风格跟踪的参考。一条样本是一段连续技能片段，不是单帧。

字段使用 SI 单位：米、秒、弧度。四元数顺序 `xyzw`。坐标系为世界系，重力沿 `-z`。球相关部署观测在训练时再变到基座系，文件里保留世界系以便和重定向结果对照。

## 文件

一个片段一个 `.npz`。数组第一维都是时间 `T`。`npz` 键名如下。

### 元数据

| 键 | 形状 / 类型 | 含义 |
| --- | --- | --- |
| `fps` | 标量 float | 采样率，目标 50 |
| `skill` | 字符串 | `pickup`、`carry`、`dribble_in_place`、`dribble_walk`、`pass`、`shoot`、`catch` 之一 |
| `source` | 字符串 | `video` 或 `mocap` |
| `source_id` | 字符串 | 原始视频或动捕片段标识 |
| `robot` | 字符串 | 固定为 `casbot02` |
| `hand_pose` | 字符串 | v1 固定为 `open_cup` |
| `label_params` | 字符串（JSON） | 打接触标签时用的阈值，见下文 |

### 机器人

与 Holosoma 转到 MuJoCo 顺序后的跟踪参考一致：根平移、根四元数、关节角。

| 键 | 形状 | 含义 |
| --- | --- | --- |
| `root_pos` | `(T, 3)` | 根位置，世界系 |
| `root_rot` | `(T, 4)` | 根四元数，`xyzw`，世界系 |
| `dof_pos` | `(T, N)` | 关节角。`N` 只含进入策略的本体关节，不含锁死的手指 |
| `dof_names` | `(N,)` 字符串 | 与 `dof_pos` 列对齐的关节名 |

手指不记录为目标轨迹。`hand_pose=open_cup` 表示整段使用同一张开手型。若重定向器仍输出了手指角，丢弃，不写入 `dof_pos`。

### 球

球是参考的一部分。没有球轨迹的片段不能用于接触技能。

| 键 | 形状 | 含义 |
| --- | --- | --- |
| `ball_pos` | `(T, 3)` | 球心位置，世界系 |
| `ball_vel` | `(T, 3)` | 球心线速度，世界系 |
| `ball_radius` | 标量 float | 半径，标准篮球约 `0.123` |

角速度不作为 v1 必需字段。有动捕或仿真真值时可加 `ball_ang_vel`，形状 `(T, 3)`，缺省则省略该键。

### 接触相位

| 键 | 形状 | 含义 |
| --- | --- | --- |
| `contact_phase` | `(T,)` 整数 | `0=free`，`1=impact`，`2=hold`，`3=flight` |
| `contact_hand` | `(T,)` 整数 | 该帧主要近手侧：`0=无`，`1=左`，`2=右`，`3=双手` |

相位含义与 [architecture.md](architecture.md) 的数据层一致：

- `impact`：球近手，且竖直速度突变
- `hold`：球随手运动
- `flight`：离手后符合重力抛物线
- `free`：其余，含静止在地上

`label_params` 至少包含三个数，单位与上表一致：

```json
{"near_hand_m": 0.15, "dvz_threshold": 1.5, "flight_residual_m": 0.05}
```

- `near_hand_m`：球心到掌心参考点小于该距离，才可能标成 `impact` 或 `hold`
- `dvz_threshold`：相邻帧竖直速度变化超过该值，且近手，标成 `impact`
- `flight_residual_m`：用重力外推的位置与实测位置之差小于该值，标成 `flight`

视频检测缺帧时，该帧 `contact_phase` 仍要填；用前后帧插值球位置，并在 `ball_visible` 里标 0。不要把缺帧直接标成 `free`。

| 键 | 形状 | 含义 |
| --- | --- | --- |
| `ball_visible` | `(T,)` 整数 | `1` 该帧球位置来自检测或真值，`0` 为插值 |

仿真真值片段全部为 1。

## 与技能的对应

| `skill` | 阶段 | 片段里应出现的相位 |
| --- | --- | --- |
| `pickup` | L1 | 以 `free` 开始（球在地上），以 `hold` 结束 |
| `carry` | L2 | 全程 `hold` |
| `dribble_in_place` | L3 | `impact` 与 `flight` 交替 |
| `dribble_walk` | L3 | 同上，且根位置有位移 |
| `pass` | L4 | `hold` 然后 `flight` |
| `shoot` | L5 | `hold` 然后 `flight` |
| `catch` | L6 | `flight` 然后 `impact`，再进入 `hold` |

## 不写入文件的量

接触力、接触点、法向、冲量只在仿真训练时由物理引擎读取，不进 `.npz`。实机没有这些传感器，写进数据会造成学生策略依赖特权信息。

图像、检测框、EKF 协方差也不进参考片段。它们属于部署时的估计器，训练时用噪声和延迟作用在球状态上即可。
