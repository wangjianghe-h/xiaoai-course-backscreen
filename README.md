# 超级小爱课程表背屏

作者：W.J.

在小米 17 Pro 系列背屏上显示课程提醒卡片。必须搭配 [课程表超级岛](https://github.com/Xposed-Modules-Repo/com.xiaoai.islandnotify)（或本仓库作者的背屏分支）与 [REAREye](https://github.com/killerprojecte/REAREye) 使用。

## 组件信息

- 组件名（business）：`xiaoai_course_schedule`（见 `widget_info.json`，不可改名）
- 类型：`notification`（由焦点通知驱动）
- 模板：`xiaoai_course_schedule.zip`（MAML v2，`frameRate=1` 低开销版）
- 当前版本：`v1.2.0`（2026-10-30）
- 最低 REAREye：251

## 版式（v1.2.0）

从上到下只有四行内容：

| 行 | 内容 | 样式 |
|---|---|---|
| 1 | 课程名 | 白色加粗 26 号；超过 5 字按 4 字一行折行（7 字 → `高等工程` / `热力学`），右上角保留超级小爱图标位 |
| 2 | 起止时间 | 青色 20 号 |
| 3 | 教室 | 薄荷绿 17 号；12 字以内单行，超长自动两行 |
| 4 | 倒计时 | 橙色 24 号，上方一条 1.5 单位细分割线 |

- 整块内容垂直居中，蓝色竖线随内容高度自适应（无倒计时时缩到教室行下方）
- 教室行与倒计时之间不预留空行，避免出现“教室只占一行、下面却空半行”的空洞
- 不再显示节次与教师（模块仍会上报这两个字段，模板不再渲染）

## 安装

1. 安装「课程表超级岛」模块（背屏支持版）并在 LSPosed 中启用，作用域包含 `com.miui.voiceassist`。
2. 安装 REAREye，开启：更多 → 通知与路由 → **允许焦点通知在背屏显示**。
3. REAREye → 配置 → 背屏中心 → 背屏个性化 → **组件管理** → `xiaoai_course_schedule` → 编辑 → 模板文件路径 → 选择文件 → 选 `xiaoai_course_schedule.zip` → 确定。
4. **必须保留一条通知路由**（删掉后背屏会收不到卡片）：
   REAREye → 通知与路由 → 通知路由 → 新增 →
   目标包名 `com.miui.voiceassist`、通知规则 `regex:.*`、组件 `xiaoai_course_schedule`。
   这条路由是 REAREye 对该包的**包级强制放行**来源，仅靠「允许焦点通知在背屏显示」的动态放行不够（实测会丢卡片）；`com.android.keyguard` 不需要路由。
5. 重启背屏服务让模板重新部署：`adb shell am force-stop com.xiaomi.subscreencenter`，然后核对 `re_xiaoai_course_schedule` 是否已生成。

## 与模块的字段契约

背屏智能助手只把 `miui.rear.param` 原样透传给 MAML 模板（`miui.focus.param` 会被过滤），因此模块把课程字段写进 `rear_param_v1`：`business` 必须等于组件 ID `xiaoai_course_schedule`，另有 `title` / `content` / `hint` / `status` / `section` / `teacher` / `countdown_label` / `timer_when`。

| 字段 | 含义 | 卡片位置 |
|---|---|---|
| `title` | 课程名（>5 字折两行，约 4 字/行） | 标题 |
| `content` | 起止时间 | 第二行（青） |
| `hint` | 教室（≤12 字单行，超出两行） | 第三行（薄荷绿） |
| `countdown_label` + `timer_when` | 倒计时前缀与目标时刻（epoch ms 字符串） | 底部橙色 |
| `section` / `teacher` | 节次 / 教师 | 保留字段，v1.2.0 起不渲染 |
| `status` | 状态（即将开始 / 上课中 / 已下课） | 保留字段 |

右上角为超级小爱图标（占原系统时钟位）。

## 模板实现要点（踩过的坑）

- **解析必须走 Trigger**：`<ExternalCommands><Trigger action="init">` + `notification_received` 调 `loadData`。
- **`num()` 只认变量**：`num(jsonGetString(...))` 恒为 0，要先赋字符串变量再 `num(@var)`。
- **熄屏倒计时**：根标签需 `useVariableUpdater="DateTime.Minute,DateTime.Second"`，`frameRate=1` 足够（比 60 帧省约 98% 重绘）。
- **行宽实测**：26 号字约 22.35 单位/字（标题一行为 4 字左右）；17 号教室行在加宽后可放约 10 字。折行必须按这个宽度预留行数，否则标题第二行会压到时间行。
- **教室行不要无条件预留两行**：只有 `len(@c_hint) > 12` 才按两行排，否则会出现视觉空洞。
- **布局避开摄像头**：背屏左侧约 296px 为遮挡区，内容从 `120*scale` 起排。
- **图标裁边**：按 alpha 包围盒裁切后再设定尺寸。

## 版本历史

- **v1.2.0**（2026-10-30）：重排版式——标题 4+3 折行、教室单行、移除节次/教师、细分割线 + 倒计时页脚、蓝色竖线自适应。
- **v1.1.0**：倒计时改为每秒刷新，图标移到右上角时钟位，蓝竖线加长。
- **v1.0.0**：首个版本。
