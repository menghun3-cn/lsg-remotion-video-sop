# 工程

每个片子一个目录：`videos/<slug>/`。不要把两片塞进同一个 Remotion 根。

```
videos/<slug>/
  package.json
  tsconfig.json
  remotion.config.ts
  src/index.ts          # registerRoot
  src/Root.tsx          # 一个 Composition
  src/theme.ts          # FPS、DURATION、每一镜的帧界、颜色
  src/cues.ts           # {start, end, text}，单位帧
  src/<Main>.tsx
  src/ui.tsx            # 字幕、底纹；宣传片再放复刻界面
  public/               # 截图、svg
  audio/narrate.py      # 阶段 4 才写、才跑
  out/<slug>.mp4        # 无声
  out/<slug>-voiced.mp4 # 有声
```

`package.json` 对齐范本：`remotion`、`@remotion/cli`、`react`、`react-dom` 四个依赖，脚本 `dev` / `render` / `typecheck`。render 指向 `src/index.ts` 和 Composition id。

`remotion.config.ts` 至少 `Config.setOverwriteOutput(true)`。Remotion 自带的 headless shell 能渲染就不要改浏览器。找不到浏览器时，再 `Config.setBrowserExecutable` 指向本机 Chrome，写法见 `videos/zhu-ye-promo/remotion.config.ts`。

## 时间轴

- `FPS = 30`，`DURATION` 是最后一镜的结束帧。
- 架构片的镜界命名沿用 hotpath：`INTRO_END`、`OVERVIEW_END`、`WALK_END`、各张卡的起点、`CLOSE_AT`。节点起点用 `OVERVIEW_END + index * 帧长`。
- 宣传片的镜界命名沿用 promo：钩子、主视觉、网格、扇形四段、下载、实机各段、结论卡、收束。
- 组件用 `useCurrentFrame()`。禁止 `Math.random()`、禁止 CSS transition / animation，否则渲染帧不稳定。

## 字幕跟随

两个范本都是底中深色胶囊、白字、只在当前句的区间内挂载。新片统一用帧：

```ts
export const CUES: {start: number; end: number; text: string}[] = [
  {start: 30, end: 240, text: "与这一镜旁白相同的一句"},
];
```

`Subtitle` 查找 `frame >= start && frame < end`。没有命中就返回 null，不要留着上一句。

字幕文案规则，两条片子都遵守：

- 一句只说这一镜画面上能指认的事实。
- 画面上没有的数字、模块、系统范围，不写进字幕。
- 句与句之间留空档（范本大约 0.3–0.5 秒），空档时字幕消失。

## 无声渲染

阶段 2 的 Composition 不 import 音频。渲染命令：

```powershell
npx remotion render src/index.ts <CompositionId> out/<slug>.mp4
```

渲染前 `npx tsc --noEmit`。
