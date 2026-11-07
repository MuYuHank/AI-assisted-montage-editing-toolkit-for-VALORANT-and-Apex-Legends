[README.md](https://github.com/user-attachments/files/34510785/README.md)
# montage

把**游戏实录 + 击杀时间点**剪成一支快节奏击杀 montage。纯本地、ffmpeg 驱动、无云端、无账号、无 GPU。

## 它凭什么像样

不是凭感觉。参考片（RAIDEN 的 *Espresso* montage）被下载下来逐帧量过：

| 量到的东西 | 数值 | 被用在哪 |
|---|---|---|
| 切点数 | 88.6 秒里 101 个 | 全局密度 |
| 中位镜头长度 | **0.60 秒** | `tier_base_s` 的量级 |
| 镜头长度分布 | 57/91 的间隔落在 0.35–0.7 秒 | 快切区间 |
| 节奏走向 | 前 10 秒平均 1.00 秒/镜 → 70–80 秒降到 0.62 秒 | `pace_curve` 收紧跟 |
| 切点对齐音频重音 | 80 ms 内命中 **65%**（随机时间只有 36%） | `beat_snap` 存在的原因 |
| 中位对齐误差 | 62 ms（随机 111 ms） | `beat_snap_s ≈ 0.075` |
| 音色 | 暖橙、高对比、明显辉光 | `espresso` look（**可换**） |
| 转场 | 横向甩镜拖影，不是叠化 | `whips` 生成方式 |
| 片尾 | 8 秒纯黑 + 文字 | 默认关闭，`outro_s` 可开 |
| 击杀检测精度 | 合成素材实测 89% 召回 / 73% 精度 | 仅是候选，必须人工复核 |

关键结论：参考片的"卡点感"是真实存在的，约 2 倍于随机的命中率——所以本工具把切点吸附到节拍网格，但**只允许移动 ≤75 ms**，且边界整体移动（不会产生空隙或重叠），宁可不对齐也不扭曲剪辑。

## 安装

```bash
# 需要 ffmpeg/ffprobe（libx264 + aac）。仓库已内置静态 arm64 版本在 tools/bin/。
python3 -m montage.cli doctor      # 自检：编码器、滤镜、磁盘
```

依赖：Python 3.9+、numpy、scipy。无其他依赖。

## 用法

```bash
export PYTHONPATH=/path/to/montage

# 1. 分析素材（逐帧信号，一次即可，结果可缓存）
python3 -m montage.cli probe --input in/game.mp4 --out work/sig.json

# 2. 让它自己找击杀点（基于击杀提示条弹出 + 画面动静），结果供人工复核
python3 -m montage.cli kills --signals work/sig.json --out work/kills.auto.json

# 3. 产出剪辑方案（确定性：同输入必同输出）
python3 -m montage.cli plan --signals work/sig.json --markers in/kills.json \
        --input in/game.mp4 --look espresso --target 30 --out work/plan.json

# 4. 渲染（含音频合成）
python3 -m montage.cli render --plan work/plan.json --input in/game.mp4 \
        --out out/montage.mp4

# 5. 验收：时长/帧率/尺寸/完整解码/锚点位置
python3 -m montage.cli check --plan work/plan.json --video out/montage.mp4

# 6. 缩略图联系人表，肉眼过一遍
python3 -m montage.cli thumb --video out/montage.mp4 --out out/sheet.png
```

`in/kills.json`：

```json
{"source_id": "s1",
 "markers": [
   {"id": "k01", "time_s": 42.35},
   {"id": "k02", "time_s": 57.10}
 ]}
```

时间用**源片时间**（秒）。`confidence` 可选，缺省按 1.0 算。

## 风格档位（节奏）

`--style` 切换剪辑节奏。档位是从实测数据来的，不是拍脑袋：

| 档位 | 中位镜头 | 最长镜头 | 卡点 | 依据 |
|---|---|---|---|---|
| `valorant` | 短、快切 | 4.0 s | 开 | 实测参考片（0.60s 中位 / 81 切每分钟） |
| `apex` | 长 | 7.5 s | 关 | 实测四条 Apex montage |
| `apex_montage_fast` | 中 | 4.5 s | 开 | 非实测，想要的更快档 |

```bash
python3 -m montage.cli styles          # 列出
python3 -m montage.cli plan ... --style apex
```

**反直觉的实测结论：Apex 剪辑比 VALORANT 剪辑慢得多。** Apex 中位镜头 1.3–3.4 秒、每分钟 14–31 切；VALORANT 参考片是 0.60 秒、81 切。Apex 靠镜头内的位移和交战带节奏，不靠切点。完整数据见 `docs/风格对比.md`。

风格和色调是**两条独立的轴**：换档位动切点，换色调不动切点。

## 色调是可换的

调色完全独立于剪辑决策，换色调不会动一个切点（有测试守着这条）。

```bash
python3 -m montage.cli looks            # 列出全部
python3 -m montage.cli plan ... --look cold
```

内置：`espresso`（参考片气质）、`neutral`、`cold`、`punch`、`film`、`clean`、`none`。

**用自己的 LUT**：把 `.cube` 放到 `looks/<name>.cube`，然后 `--look <name>`。
**永久改默认**：改 `config/default.json` 的 `"look"` 字段。
**改强度**：`style.py` 里每个 preset 的 `halation / vignette / grain / grade` 都是普通数据，直接调。

## 它不会做的事

- 不下载参考片、不复用其音乐、不模仿其成片。参考只取节奏与气质。
- 不声称检测准确率。`kills` 命令的输出是**候选**，必须人工复核。
- 素材不够就报短，**不循环、不重复、不补假镜头**。
- 音乐没标 `rights_confirmed: true` 就不混进去。
- 不自动上传、不发布。

## 结构

```
montage/
  ffmpeg.py     安全 argv 封装、probe、doctor
  analysis.py   逐帧信号：运动、帧差、亮度、击杀提示条、切点
  beats.py      节拍检测（谱通量 → 自相关 → 相位锁定网格）
  selection.py  窗口合并、节奏分配、节拍吸附、转场/推近决策
  timeline.py   源时间 ⇄ 成片时间映射
  render.py     逐段渲染 + 甩镜生成 + 拼接
  audio.py      游戏音、音乐、闪避、限幅
  style.py      可换色调 preset / 外部 LUT
  cli.py        命令行
```
