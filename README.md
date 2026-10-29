# 超级小爱课程表背屏

在小米 17 Pro 系列背屏上显示课程提醒卡片。必须搭配 [课程表超级岛](https://github.com/Xposed-Modules-Repo/com.xiaoai.islandnotify)（或本仓库作者的背屏分支）与 [REAREye](https://github.com/killerprojecte/REAREye) 使用。

## 组件信息

- 组件名（business）：`xiaoai_course_schedule`（见 `widget_info.json`，不可改名）
- 类型：`notification`（由焦点通知驱动）
- 模板：`xiaoai_course_schedule.zip`（MAML v2，`frameRate=1` 低开销版）
- 最低 REAREye：251

## 安装

1. 安装「课程表超级岛」模块（背屏支持版）并在 LSPosed 中启用，作用域包含 `com.miui.voiceassist`。
2. 安装 REAREye，开启：更多 → 通知与路由 → **允许焦点通知在背屏显示**。
3. REAREye → 配置 → 背屏中心 → 背屏个性化 → **组件管理** → `xiaoai_course_schedule` → 编辑 → 模板文件路径 → 选择文件 → 选 `xiaoai_course_schedule.zip` → 确定。
4. 重启背屏服务让模板重新部署：`adb shell am force-stop com.xiaomi.subscreencenter`，然后核对 `re_xiaoai_course_schedule` 是否已生成。

## 与模块的字段契约

背屏智能助手只把 `miui.rear.param` 原样透传给 MAML 模板（`miui.focus.param` 会被过滤），因此模块把课程字段写进 `rear_param_v1`：`business` 必须等于组件 ID `xiaoai_course_schedule`，另有 `title` / `content` / `hint` / `status` / `section` / `teacher` / `countdown_label` / `timer_when`。

| 字段 | 含义 | 卡片位置 |
|---|---|---|
| `title` | 课程名（约 9 字折两行） | 标题 |
| `content` | 起止时间 | 第二行（青） |
| `hint` | 教室 | 第三行（薄荷绿） |
| `section` / `teacher` | 节次 / 教师 | 第四行（淡紫） |
| `countdown_label` + `timer_when` | 倒计时前缀与目标时刻（epoch ms 字符串） | 底部橙色 |

右上角为超级小爱图标（占原系统时钟位）。

## 模板实现要点（踩过的坑）

- **解析必须走 Trigger**：`<ExternalCommands><Trigger action="init">` + `notification_received` 调 `loadData`。
- **`num()` 只认变量**：`num(jsonGetString(...))` 恒为 0，要先赋字符串变量再 `num(@var)`。
- **熄屏倒计时**：根标签需 `useVariableUpdater="DateTime.Minute,DateTime.Second"`，`frameRate=1` 足够（比 60 帧省约 98% 重绘）。
- **多行标题锚点不同**：用字数判断（`len(@c_title) > 9`）+ 固定行高，不要依赖 `.text_height`。
- **布局避开摄像头**：背屏左侧约 296px 为遮挡区，内容从 `120*scale` 起排。
- **图标裁边**：按 alpha 包围盒裁切后再设定尺寸。
