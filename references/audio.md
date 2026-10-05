# 成片后的语音和音效

前置：`out/<slug>.mp4` 已通过阶段 3。此前不调用 ElevenLabs，不跑 Edge TTS，不写 mp3。

## 顺序

1. 从 `cues.ts` 生成旁白清单。每句：`id`、`start = startFrame / FPS`、`max_dur = (endFrame - startFrame) / FPS - 0.12`、`text` 与字幕相同。0.12 秒是句尾空隙，避免叠到下一句。
2. 语音：先 ElevenLabs，失败再 Edge TTS。同一条片子只用一个引擎，不要一句一个引擎。
3. 音效：确认表写了「要」才生成。只走 ElevenLabs `sound-effects`。失败就整片不要音效，交付里说明。
4. 用 ffprobe 量每句时长。超过 `max_dur` 就停，改短文案或加长该镜。加长该镜要改 `theme.ts` 和 `cues.ts`，重渲无声 mp4，再从步骤 1 重来。宣传片范本允许最多约 1.18 倍 `atempo` 压进窗口；架构片范本超了就直接失败。新片默认按架构片：超了就改，不偷偷加速，除非用户同意。
5. ffmpeg 混流。视频流 copy，不重编码画面。

## ElevenLabs

先读技能再调用，不要凭记忆写参数。

- 语音：`text-to-speech`。中文用 `eleven_multilingual_v2`（或该技能里当前推荐的多语言模型）。气质按确认表：讲解沉稳或宣传轻快。
- 音效：`sound-effects`。一条 0.4–1.2 秒，描述对准那一镜的动作（卡片落下、令牌掠过、粒子散开）。`prompt_influence` 偏高（约 0.7）。起点用该动作的帧 / FPS。

下面任一情况视为不支持，语音改走 Edge TTS，音效放弃：

- 没有 `ELEVENLABS_API_KEY`，或 key 无效
- 额度、网络、接口报错
- 当前模型或音色不能稳定出中文

不要为了配音去走一遍完整的 key 开通流程把成片卡住。用户明确要求必须用 ElevenLabs 时，才转 `setup-api-key`。

## Edge TTS 降级

依赖：`pip install edge-tts`。声音按确认表，不要临时换人：

| 气质 | voice | rate |
| --- | --- | --- |
| 讲解沉稳（架构默认） | zh-CN-YunyangNeural | -4% |
| 宣传轻快（宣传默认） | zh-CN-XiaoxiaoNeural | -5% |

写法对齐 `videos/zhu-ye-hotpath/audio/narrate.py` 与 `videos/zhu-ye-promo/audio/narrate.py`：`edge_tts.Communicate(text, voice, rate=rate)`，一句一个 mp3，放在 `audio/clips/`。

## 混流

与两个 `narrate.py` 相同：

- 输入 0：无声 mp4
- 输入 1：`anullsrc` 立体声 48000，时长等于成片
- 输入 2 起：旁白，音效接在旁白后面
- 每条：`aresample=48000`，`adelay` 为起点毫秒，左右声道同一个值
- `amix`，`duration=first`，`dropout_transition=0`，`normalize=0`
- `-map 0:v -map [aout] -c:v copy -c:a aac -ar 48000 -b:a 160k`
- 输出 `out/<slug>-voiced.mp4`
- 把每句 `raw / window / OK|OVER` 写到 `audio/clips/report.txt`

有 OVER 就不要把 voiced 文件交给用户。

音效和旁白一起进 amix。音效不占字幕窗口，但不要盖住旁白的词头：点状音效放在该动作的第一帧，音量保持生成结果，不另做闪避，除非听感明显盖词。
