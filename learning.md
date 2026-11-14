# 万象画卷 OmniDraw 个人魔改说明（zzai 视频接入 + 1440p + /画 /视频 人设锁定）

> 适用于SnowLuma + Astrbot配置的onebot插件
>
> - 视频：zzai（api.seedance.nz）端点修复 + `enhance_prompt` 默认开启 + **1440p 默认**
> - 绘图/视频：`/画` 和 `/视频` 都锁定为「只出当前人设角色」
> - 推送：视频发送失败时自动退化为链接，不再误报失败
> - 超时：提交超时 30s → 180s，带图失败自动退回纯文本重试
> - 发送：视频单独成段（QQ 要求），文字另发；发不出去才退回链接

---

## 个人配置过程中发现的问题如下：

## 一、填写请求地址时lnvalid URL问题

### 1. 主因：插件把视频接口写死成 `videos`（复数），zzai 是 `video`（单数）

万象画卷所有视频协议（`async_task` / `openai_sync` / `openai_chat`）都会在“接口地址”后面
自动拼接 `/videos/generations`，代码在 `providers/base.py` 的
`build_video_generations_endpoint()`：

```python
return _append_url_path(base_url, "videos/generations")   # 复数
```

而 zzai 的文档与真实接口是 **`/v1/video/generations`（单数）**。

对 `api.seedance.nz` 的真实请求结果（不带 key 也能看出路径是否存在）：

| 请求 | 结果 |
| --- | --- |
| `POST https://api.seedance.nz/v1/videos/generations` | **404** `{"error":{"message":"Invalid URL (POST /v1/videos/generations)"}}` |
| `POST https://api.seedance.nz/v1/video/generations` | **401** `Invalid token`（路径正确，只差令牌） |

所以接口地址无论填 `https://api.seedance.nz/v1` 还是填完整的
`https://api.seedance.nz/v1/video/generations`，插件实际请求的都是错误地址。

### 2. 次因：任务状态读不到

zzai 的查询结果长这样（状态在 `data` 里，视频地址还嵌了一层）：

```json
{
  "code": "success",
  "data": {
    "status": "SUCCESS",
    "progress": "100%",
    "result_url": "https://…/v1/videos/<task_id>/content",
    "data": { "status": "completed", "url": "https://…/v1/media/xxx.mp4" }
  }
}
```

旧代码只读**顶层** `status`，于是每 10 秒轮询都是“状态:（空）”，一直空转到超时报
“视频生成轮询超时”。旧代码取视频地址时也不认 `result_url` 和 `data.data.url`。

---

## 二、安装 加载出错

文件：`astrbot_plugin_omnidraw_3.5.1_video-local.zip`

AstrBot 面板 → 插件管理 → 安装插件 → **上传 zip** → 重载插件。

> 若提示插件已存在：先卸载旧版再上传（`data/config/astrbot_plugin_omnidraw_config.json`
> 里的配置不会丢），上传后重新启用并检查配置值。

改动清单：

| 文件 | 改动 |
| --- | --- |
| `providers/base.py` | 视频端点改为 `/video/generations`，填了完整路径就原样使用，同时兼容 `/videos/generations` |
| `core/video_manager.py` | 轮询状态读 `data.data.status` → `data.status` → 顶层；视频地址认 `result_url` / `data.data.url`；提交时带 `enhance_prompt: true` 和 `metadata.resolution`（默认 1440p）；带参考图被拒时自动退回纯文本重试；视频发不出去时改发链接 |
| `main.py` | `/画` 和 `/视频` 都锁定人设角色（新增 3 个方法 + 两处命令分支） |

---

## 三、`/画` 和 `/视频` 胡乱生成角色

### 修改行为

- `/画 在咖啡店看书` → 自动拼成
  `人设基础描述 + 在咖啡店看书 + 「画面中的唯一角色是「小真」，五官、发型、体型和服装必须与参考图一致，不要出现其他人物」`，
  并把当前人设的参考图一起送进图像接口。
- `/视频 在咖啡店看书 --seconds 5` → 同样拼上人设描述和约束，并把当前人设的参考图作为
  参考图/首帧一起提交。
- **人设没配**（既没写描述也没传参考图）→ 直接回复提示，让你先去配置，不出图/不出片、不扣次数。
- **临时恢复普通生成**：`/画 一只猫 --persona false`、`/视频 一只猫 --persona false`
- 不影响 `/自拍`、`/gif`、LLM 工具（`generate_image` / `generate_video` 仍按原逻辑）。
- 视频带参考图如果被中转站拒绝（zzai 只认 URL、不认 base64 的情况），插件会自动去掉参考图
  用纯文本提示词重试一次，不会整条失败。

### 效果上的取舍

`/画`、`/视频` 现在固定是“人设 + 你描述的场景/动作”。所以 `/画 一只猫` 会画成
“人设抱着一只猫”，而不是单独一只猫；想要纯物体或纯风景，加 `--persona false`。

### 人设怎么配

插件配置 → 人设（persona）：

| 字段 | 建议 |
| --- | --- |
| 人设基础描述 | 外貌、年龄、发型发色、瞳色、体型、常穿服装，越具体越稳 |
| 人设参考图 | 1–3 张正面、光线均匀、无遮挡；多张比单张更容易锁住脸 |
| 当前启用人设 | 选你要用的那个 |

> 只写描述、不传参考图也能用，但长相一致性会差一些；参考图 + 描述一起用效果最好。

---

## 四、提示词优化（`enhance_prompt`）

zzai 文档：

> `"enhance_prompt": true` 会在调用模型前，用与 Playground 相同的优化器改写提示词。默认关闭。
> 优化与模型分开计费，按 `deepseek/deepseek-v4.1-flash` 的实际 Token 用量收费；
> 请求带参考图片或视频时，另加 `qwen/qwen3.8-omni-flash` 的用量。
> 生成失败时优化费不退；优化失败时不会调用模型，也不收模型费用。

现在的行为：

- **默认开启**：每次 `/视频` 都会带 `"enhance_prompt": true`。
- **单次关闭**：`/视频 一只猫 --enhance_prompt false`
- **永久关闭**：把 `core/video_manager.py` 里
  `payload["enhance_prompt"] = _coerce_bool(payload.get("enhance_prompt", True))`
  的 `True` 改成 `False`。

费用提醒：优化是**额外按 token 计费**的（视频本身另算），带参考图时会多一层
`qwen3.8-omni-flash` 的用量。

> 插件自带的「副脑提示词优化」（`optimizer_config.enable_optimizer`）是另一套东西，用你自己的
> 文本模型先改写。已经开了 zzai 的 `enhance_prompt`，副脑保持关闭即可，避免重复花钱。

---

## 五、视频节点



| 字段 | 填写值 |
| --- | --- |
| 节点 ID | `video_node_1` |
| 调用协议 | `async_task`（异步任务） |
| 接口地址 | `https://web/v1` |
| API Keys | 你的 API令牌（`sk-` 开头） |
| 当前模型 | `module` |
| 视频模型池 | `module` |
| 请求超时（秒） | `900` |
| 默认尺寸 | 留空（代码里默认 1440p；想改就填 `1440p`、`1080p` 这类） |

链路：视频链路 = `video_node_1`。

图像节点（同一个 zzai）：接口模式 `custom_endpoint`，接口地址
`https://web/v1/images/generations`，模型填你买的图像模型。

---

## 六、视频分辨率（默认 1440p）

zzai 的分辨率放在请求体的 `metadata.resolution` 里，插件原来没有这个字段，现在默认
**1440p / 16:9**：

- 默认请求：`metadata: {"resolution": "1440p", "ratio": "16:9"}`
- 单次改：`/视频 … --resolution 1080p --ratio 9:16`
- 也可以在视频节点的「默认尺寸」填 `1440p`（只认 `数字+p` 这种写法；填 `1280x720` 不生效）

可选值：`480p / 608p / 768p / 896p / 1088p / 1280p / 1440p`。分辨率越高越慢、越贵。

---

## 七、关于“未知错误”/“TimeoutError”

### TimeoutError

最可能是**提交阶段超时**：人设参考图会转成 base64 塞进请求体，1 张手机照片就 2–4 MB，
三张就是十几 MB，旧版提交只给 **30 秒**，上传慢一点就必然 `TimeoutError`。

处理：

- 提交超时从 30 秒提到 **180 秒**（`provider timeout` 更大时按更大的来，上限 300 秒）
- 提交时日志会打印**请求体大小和参考图张数**，一眼看出是不是图太大
- 带参考图提交超时/失败 → 自动去掉参考图，用纯文本提示词**再试一次**
- 视频参考图最多带 **2 张人设图**、总数 **4 张**，控制上传体积
- 轮询单次请求超时 15 秒 → **30 秒**

查看日志这一行就知道是不是这个原因：

```
🎬 [Async Task 模式] 提交视频任务至: https://api.seedance.nz/v1/video/generations（请求体约 8600 KB，参考图 3 张）
```

### `retcode=1400` / `message element "video" must be the only segment`

这是 QQ（OneBot）协议限制：**一条消息里 video 段必须是唯一段**，不能和文字、图片、@ 拼在一起。

旧代码把「说明文字 + 视频」塞进同一条 chain，所以视频生成成功、发送时被 1400 拒绝，然后触发了
“改发链接”的兜底——你在群里看到的那个链接就是这么来的，不是生成失败。

## 调整视频生成

1. 先单独发一条**只有视频**的消息（代码里 chain 只有一个 `Video` 段，不带任何文字）；
2. 视频发成功后再单独发一条说明文字；
3. 只有视频这条也发不出去时，才退回链接；
4. 日志会打印 `🔗 视频地址: …`，即使发送失败也能直接拿到片子。

## 追加的排查/兜底：

- 发送前先探测直链：`🔍 视频直链探测: HTTP 200, video/mp4`
- 如果框架发送仍然报 1400，会**直接调用 OneBot 接口**发单段视频（绕开框架可能附加的引用/空格段）
- 日志里能明确看到是「生成失败」「直链不可访问」还是「平台拒收」

## 追加的第三条路与诊断回传：

- 前两条都失败时，把 mp4 **下载到本地**，再用 `file://` 本地路径通过 OneBot 发送
  （绕开“协议端拉不到远程 URL”的情况，NapCat 与本机同机时最有效）
- 仍然失败时，群里那条链接消息会带一行**诊断**：
  `（诊断：直链 HTTP 200, video/mp4；发送报错 …；直发：…；本地文件：…）`
  —— 把这一行原样发我就能定位，不用翻日志

### 视频到底能不能用参考图（人设图的真实状态）

能读、能转、能发，但 zzai 的**视频**接口对 base64 参考图不友好：

- 图像接口（`/v1/images/generations`）：base64 参考图正常，`/画`、`/自拍` 没问题；
- 视频接口（`/v1/video/generations`）：把几 MB 的 base64 塞进 `images`，服务端一直不返回，
  客户端只能超时；去掉 `images` 后纯文本提交秒过。


### 生成成功但没收到视频

1. **视频生成成功、但聊天平台发不出去**（QQ 适配器不支持视频、CDN 链接被墙、链接要鉴权）。
   旧版直接抛“视频结果发送失败”，中间的异常被吞掉时就显示成“未知错误”。
   现在：发送失败会自动改发一条**带下载链接的文本**，不再说“失败”。
2. **提交/轮询阶段异常，异常信息为空**。旧版统一写成“未知错误”。
   现在：错误信息会带异常类型，超时会带**最后状态和进度**，日志里也带完整 traceback。


---

## 八、测试

```
/视频 在窗边转身看向镜头 --seconds 5
/视频 一只橘猫打哈欠 --persona false --seconds 5
/视频 … --resolution 1080p --ratio 9:16
/画 在咖啡店看书
/画 一只猫 --persona false
```

视频日志应出现：

```
🎬 [Async Task 模式] 提交视频任务至: https://api.seedance.nz/v1/video/generations
⏳ [视频轮询] Task ID: xxx, 状态: IN_PROGRESS (尝试 1)
⏳ [视频轮询] Task ID: xxx, 状态: SUCCESS (尝试 N)
```

如果日志里出现「带参考图的视频请求失败…改为纯文本提示词重试」，说明api不接受 base64
参考图，插件已经自动降级出片（只是人设只能靠文字描述约束，锁脸会弱一些）。

---

## 九、不想装 zip？手动替换文件

覆盖到 `AstrBot/data/plugins/astrbot_plugin_omnidraw/` 对应位置，然后重载插件：

- `providers/base.py`（视频端点）
- `core/video_manager.py`（状态/地址/提示词优化/参考图降级重试）
- `main.py`（`/画`、`/视频` 人设锁定）

---

## 十、给插件作者反馈

- Issue #72：`async_task` 轮询不读 `data` 对象内的状态
- Issue #73：视频三种协议都强行追加固定路径





