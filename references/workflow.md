# 标准流程

所有生成类共用：

```
确认规格(未说明则问或保守默认) → 上传素材(可选) → AI帮写(套图/详情页推荐) → 按硬限制创建任务(超量则拆多次) → 轮询 /task/result → 把 URL 交给用户
```

**规格确认：** 见 `SKILL.md`「生成规格」— 张数/比例/清晰度/时长未说明时，先问或保守默认并告知；禁止静默用大示例值。

1. 创建接口返回 `result.TaskNo`（或 `taskNo` / `TaskNoStr`）
2. `GET /task/result?taskNo={TaskNo}&needParams=true`，约每 **10 秒**一次
3. Status：`0` 未开始 / `1` 生成中 / `3` 完成 / `5` 失败
4. `Result` 可能是 JSON 字符串，先 parse。作图取 `ImageUrls[]`，视频取 `VideoUrl`，失败取 `FailReason`
5. 到完成或失败为止；**全部任务结束后把结果 URL 交给用户**，不要让用户回 App 查

**超量拆分（强制）：** 需求数大于单次上限时，按上限拆成多次 createTask（如要 10 张爆款首图 → 两次 `num="5"`），**禁止**一次传超限值。每次 create 成功都按 `references/usage.md` 各打一次点。

## 单次上限总表

| Tool | 单次上限 | 说明 |
|------|----------|------|
| `scene_image_gen` | num≤5 | 单次任务最多生成 5 张（num 只能是 "1"～"5"）；比例仅 1:1 / 3:4 |
| `product_photo_set` | suiteTypes≤9 | 单次任务张数由 suiteTypes 决定：智能匹配（suiteTypes=""）约 5 张；自定义最多勾选 9 种类型，每种 1 张 |
| `detail_page_gen` | imageCount=1/6/7/8/9/10 | 单次 imageCount 仅允许 1 / 6 / 7 / 8 / 9 / 10（默认 9），不是任意 2～5 |
| `short_video_gen` | 单次 1 份/条 | 单次 1 条视频；duration 4～15 秒；resolution 720P/1080P；标准版无 ratio |
| `freedom_image_gen` | num≤5 | 单次任务最多生成 5 张（num 为数字 1～5） |
| `comment_analyze` | 单次 1 份/条 | 单次任务生成 1 份评论洞察报告 |
| `batch_poe_image` | 见说明 | 单次任务张数由 zip 内成功上传的印花图数量决定，无独立 num 字段 |
| `content_recreate` | num≤1 | 单次任务固定生成 1 张（num 必须为 "1"） |
| `viral_video_recreate` | 单次 1 份/条 | 单次 1 条复刻视频；resolution 仅 720P；originDuration=源视频秒数（3～60） |

## 套图 / 详情页

提交前应先 `POST /ai/generateProductSellingPoint`（`imageUrls` + 可选 `productName`），用返回的商品名与卖点填表，再调生成接口。不要空卖点硬提。

## 禁止调用

- **账户写操作**：登录/登出、改昵称、改密码、发验证码
- **充值写操作**：创建充值订单、支付、查单支付状态
- **邀请推广**：邀请奖励、推广信息
