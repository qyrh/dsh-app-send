# dsh-app-send

给 DSH 一个工具：**让 agent 给微信 / QQ / 企业微信里的某个联系人发一条文字，不用打字。**

```
app_send({ app: "QQ",   to: "我的电脑", text: "..." })
app_send({ app: "企业微信", to: "智能助理", text: "..." })
app_send({ app: "微信", to: "宋紫秦", text: "..." })
```

共用的核心不是「一套坐标」，而是一套**先探测、再选通路、最后固化**的流程。

---

## 两种通路（这是关键）

| | 通路 A：无障碍元素 | 通路 B：截图 + OCR |
|---|---|---|
| 适用 | App 暴露无障碍树 | App 屏蔽无障碍树 |
| 实测 | **QQ、企业微信** | **微信** |
| 做法 | 取元素 id → tap / type | 截图 → tesseract → 按坐标点 |
| 速度 | 6～14 秒 | 约 10 秒 |
| 精度 | 高（元素精确） | 中（依赖 OCR 认字） |

探测一个 App 属于哪种，一条命令：

```
app_probe({ app: "钉钉" })
→ { strategy: "a11y" | "ocr", elements: 32, screenshot: "ok", sample: [...] }
```

结果会写进 `~/.dsh/app-send/profiles/<app>.json`，之后 `app_send` 直接按记录的策略跑。

## 研究一个新 App 的 5 步法

1. **探测**：`app_probe({ app })` —— 有无障碍元素就选 A，没有就选 B。
2. **演练**：`app_send({ app, to, text, dry: true })` —— 只输入不发送，看步骤对不对。
3. **看现场**：失败时把报错里列出的元素/OCR 文本贴出来，就能定位点错在哪一步。
4. **调参**：改 `lib/index.js` 顶部的 `CONFIG`（见下表）。
5. **固化**：跑通后把结论写进 profile（`app_probe` 会自动写），下次直接命中。

> 演练模式（`dry`）是安全网：它会把字真的打进输入框，但不点「发送」。
> 先在「我的电脑」这类自己给自己的会话上试，再发给真人。

## 换一台手机怎么调

| 现象 | 改什么 |
|---|---|
| 元素点不到 / 报「已过期」 | 不用改，代码每次都会重取屏幕再点；还不行就把 `tapWaitMs` 调到 1000 |
| 找不到联系人的会话行 | 名字在行首匹配，带 emoji 或特殊符号时，用 `to` 传更短的前缀 |
| 搜索页没有输入框 | 该 App 走列表点击；把 `CONFIG` 里搜索逻辑跳过即可 |
| OCR 全是乱码 | `ocrLang` 换语言包；`tesseract --list-langs` 确认 `chi_sim` |
| 点「发送」没反应 | `confirmSendRatio`（通路 B 的按钮相对位置）±0.03 |
| 分享面板慢 | `shareWaitMs` 加到 1800 |

## 现状

- 微信：通路 B（分享 → 选人 → 发送）。名字匹配用「列表里只有唯一一行」做安全闸，名字对不齐就中止，不猜。
- QQ：通路 A。`App → 会话行 → 输入框 → 发送`，6 秒。
- 企业微信：通路 A。会先走搜索再进会话，14 秒（可优化）。
- 通路 B 只能发文字；通路 A 理论上还能发图（tap 相册），未实现。

## 依赖

```bash
apt-get install -y tesseract-ocr tesseract-ocr-chi-sim python3-pil   # 只有通路 B 需要
```

## License

MIT
