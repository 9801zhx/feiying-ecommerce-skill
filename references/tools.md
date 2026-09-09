# 飞影 Tool 手册

**Base：** `https://safe2.zzbtool.com/fyApi`  
**Header：** `Token: {your_api_key}`（Header 名 Token，值 ApiKey）；POST 再加 `Content-Type: application/x-www-form-urlencoded`  
**无 ApiKey：** 官网 `https://feiyingai.com/` 登录 → 自动进 `https://feiyingai.com/#/center` 复制  
**积分不足：** `https://feiyingai.com/#/recharge?apiKey={your_api_key}`（query 带 `apiKey=`，勿自己下单）  
下面的路径都拼在 Base 后面。请求体示例可直接发送；`subType` 等标注「不要放进请求 body」的字段只用来选接口。

## 查询类 Tool

### 积分余额 (`get_user_balance`)

查询当前登录用户资料与积分（可用积分、赠送积分等）。

- **接口：** `GET /member/info`
- **说明：** 创建任务若返回积分不足：打开 `https://feiyingai.com/#/recharge?apiKey={ApiKey}`（勿调充值下单接口）。

**完整请求：** `GET https://safe2.zzbtool.com/fyApi/member/info`，Header 仅 `Token`（值为 ApiKey）。成功 `code=0`，积分看 `result.Credit` + `result.GiveCredit`。401 或无 ApiKey：引导 `https://feiyingai.com/` 登录，自动进个人中心 `https://feiyingai.com/#/center` 复制 ApiKey。

### 充值订单列表 (`list_recharge_orders`)

分页查询充值订单，支持按支付方式与时间筛选。

- **接口：** `GET /member/payOrderList`

**Query 示例**（拼到 URL 后直接 GET）

```
page=1
size=10
```

**参数**（`*` 必填）

- `page` number — 页码，默认 1
- `size` number — 每页条数，默认 20
- `type` string — 支付方式：1 支付宝 / 2 微信；仅 1/2
- `payTimeStart` string — 开始时间，如 2026-01-01 00:00:00
- `payTimeTimeEnd` string — 结束时间（字段名即为 payTimeTimeEnd），如 2026-01-31 23:59:59

### 获得与消耗 (`list_credit_records`)

分页查询积分流水（全部/消耗/获得）。

- **接口：** `GET /member/creditRecordList`

**Query 示例**（拼到 URL 后直接 GET）

```
page=1
searchType=1
```

**参数**（`*` 必填）

- `page` number — 页码，默认 1
- `pageSize` number — 每页条数，默认 20
- `searchType` string — 0 全部 / 1 消耗 / 2 获得；仅 0/1/2

## 辅助 Tool（上传 / AI帮写 / 查结果）

### 上传素材到 CDN (`upload_media`)

将本地图片/文件上传后得到公网 URL。已有公网 URL 可跳过。只调飞影 fyApi（Header: Token）拉凭证，再对 TOS PUT；不要对 static2.zzbtool.com 发上传请求。

- **接口：** `GET+PUT /util/bosSessionTokenV2（fyApi，Header: Token） + TOS PutObject`
- **说明：** 批量印花 zip 内图片用 UploadBase：folderName=batch-poe-img-gen，patternFolderPath 取对象存储目录 path（不是 CDN URL）。

**上传步骤**

- 已有公网 URL 则跳过上传，直接给生成接口
- 只调 fyApi：GET /util/bosSessionTokenV2?type=static_file，Header 只有 Token（值为 ApiKey；不要 kind、不要 zzb-time / zzb-sign）
- 禁止对 static2.zzbtool.com 做 POST/PUT 上传；用返回的 ak/sk/token 对 TOS 按下方规则 PUT
- 对象 key：`{dir}fy_upload/upload_img/{yyyy-MM-dd}/{随机前缀}_{fileName}`（kind=file 则目录改为 upload_file）
- 公网 URL = `https://{result.cdn}/` + 分段 encode 后的 key（cdn 来自 fyApi 凭证，不要写死域名）

**上传只走飞影 fyApi（Header: Token），不要对 `static2.zzbtool.com` 发任何上传请求。** 项目里对应接口是 `GET /fyApi/util/bosSessionTokenV2?type=static_file`。

**1. 用 ApiKey 拉临时凭证**（请求头仍为 Token）

```
GET https://safe2.zzbtool.com/fyApi/util/bosSessionTokenV2?type=static_file
Token: {your_api_key}
```

- Query **只有** `type=static_file`。不要传 `kind`、不要带 `zzb-time` / `zzb-sign`
- 成功 `code=0`，使用 `result`：`ak` `sk` `token` `endpoint` `region` `bucket` `dir` `cdn` `expires` `serverType`
- `serverType` 必须为 `2`；`expires` 约 300 秒，过期须重新 GET
- `result.token` 是 **STS**（给下一步 `x-tos-security-token`），不是 ApiKey
- `result.cdn` 是公网域名（不要写死 `static2.zzbtool.com`，以本次返回为准）

**2. 对象 key 与 PUT URL**（PUT 发往 TOS，不是 fyApi，也不是 static2）

- `kind=image` → `{dir}fy_upload/upload_img/{yyyy-MM-dd}/{随机前缀}_{fileName}`
- `kind=file` → `{dir}fy_upload/upload_file/{yyyy-MM-dd}/{随机前缀}_{fileName}`
- `dir` 以凭证为准（常见 `static_files/`，不要漏末尾 `/`）
- PUT 用 virtual-hosted（从 `result.endpoint` 取主机，把桶名接到前面）：

```
PUT https://{bucket}.{endpoint主机}/{key}
Host: {bucket}.{endpoint主机}
```

例：endpoint=`https://tos-cn-beijing.volces.com` → `https://{bucket}.tos-cn-beijing.volces.com/{key}`

Canonical URI 固定为 `/{key}`，**不要把路径里的 `/` encode 成 `%2F`**。

**3. TOS4 签名（必做）**

Authorization 算法名必须是 `TOS4-HMAC-SHA256`，service 必须是 `tos`。

必签 Header（SignedHeaders 如下）：

- `host`: `{bucket}.{endpoint主机}`
- `x-tos-content-sha256`: 文件字节的 SHA256，**小写 hex**
- `x-tos-date`: UTC `yyyyMMddTHHmmssZ`
- `x-tos-security-token`: 凭证 `result.token`（STS）
- `Content-Type` 可带（如 `image/jpeg`），**不要**放进 SignedHeaders

```
canonical_request =
PUT
/{key}
<空行>
host:{bucket}.{endpoint主机}
x-tos-content-sha256:{payloadHash}
x-tos-date:{amzDate}
x-tos-security-token:{stsToken}
<空行>
host;x-tos-content-sha256;x-tos-date;x-tos-security-token
{payloadHash}

string_to_sign =
TOS4-HMAC-SHA256
{amzDate}
{yyyyMMdd}/{region}/tos/request
{SHA256(canonical_request) 小写 hex}

SigningKey（第一跳用「原始 sk」当 HMAC key，不要加 TOS4 前缀）：
kDate    = HMAC-SHA256(key=UTF8(sk),        msg=yyyyMMdd)
kRegion  = HMAC-SHA256(key=kDate,           msg=region)
kService = HMAC-SHA256(key=kRegion,         msg=tos)
kSigning = HMAC-SHA256(key=kService,        msg=request)
Signature = hex(HMAC-SHA256(key=kSigning, msg=string_to_sign))

Authorization: TOS4-HMAC-SHA256 Credential={ak}/{yyyyMMdd}/{region}/tos/request, SignedHeaders=host;x-tos-content-sha256;x-tos-date;x-tos-security-token, Signature={Signature}
```

PUT body = 文件**原始字节**（不要再包成 multipart）。HTTP 200 即成功。

对话内可用 python/openssl **当场算 HMAC**；不要在用户项目里新建 `.py`/`.js`，也不要引入 TOS SDK。

**禁止**

- 对 `https://static2.zzbtool.com/` 或任意 CDN 域名做 POST/PUT 上传（那不是上传接口）
- 把 ApiKey 当 TOS PUT 的 `Authorization` / Header `Token`
- `AWS4-HMAC-SHA256` 或 `x-amz-*`（本桶返回 Unsupported Authorization Type）
- SigningKey 第一跳写成 `HMAC("TOS4"+sk, date)`（部分 TOS 公开文档如此，本桶 403 SignatureDoesNotMatch）
- Windows 下用 PowerShell 别名 `curl`（实为 Invoke-WebRequest）；请用 `curl.exe`

**4. 公网 URL（用 fyApi 返回的 cdn，不要写死 static2）**

`https://{result.cdn}/` + 按路径分段 encode 后的 key（斜杠保持为 `/`）。

例：`result.cdn=example.cdn.host` → `https://example.cdn.host/{dir}fy_upload/upload_img/2026-08-31/ab12_product.jpg`

把该 URL 填进后续生成接口的 `imgUrl` / `imgUrls`。

**参数**（`*` 必填）

- `file`* binary — 本地文件原始字节，作为 TOS PUT 的 body（不是 fyApi 表单字段）
- `kind` string — 只决定对象 key 目录：image=upload_img，file=upload_file；不要当成 bosSessionTokenV2 的 query；仅 image/file

### AI 帮写商品名与卖点 (`ai_help_write`)

根据商品图识别商品名称并生成卖点文案。商品套图、电商详情页在提交前应优先调用本接口，再把结果填入生成参数。

- **接口：** `POST /ai/generateProductSellingPoint`
- **说明：** 返回 result.ProductName、SellingPoints[]、TargetAudience[]、UsageScenarios[]。套图建议格式化为编号卖点文案写入 userInputRequirement；详情页写入 vocInfo，并可追加「生成要求： 图片内要有文案出现」。

**请求体示例**（`application/x-www-form-urlencoded`，对话内直接 POST）

```
imageUrls=https://{cdn}/.../product.jpg
productName=
```

**参数**（`*` 必填）

- `imageUrls`* string — 商品图 URL（多图取第一张）
- `productName` string — 已有商品名可传入；空则由 AI 识别

### 查询任务结果 (`get_task_result`)

按 taskNo 轮询异步生成任务状态与结果。大多数生成能力创建后都应轮询本接口。

- **接口：** `GET /task/result`
- **说明：** Status：0 未开始 / 1 生成中 / 3 完成 / 5 失败。建议每 10 秒轮询，直到 3 或 5。Result 可能是 JSON 字符串，先 parse。作图完成取 Result.ImageUrls[]；视频完成取 Result.VideoUrl；失败看 FailReason / msg。评论洞察报告正文也可再查 GET /commentAnalyze/result?taskNo=xxx。

**Query 示例**（拼到 URL 后直接 GET）

```
taskNo=T202601010001
needParams=true
```

**参数**（`*` 必填）

- `taskNo`* string — 创建任务返回的 TaskNo
- `needParams` boolean — 是否回填入参，建议 true

## 生成类 Tool

### 爆款首图 (`scene_image_gen`)

根据商品图生成电商爆款场景首图。

- **接口：** `POST /productAi/sceneImgGen`
- **硬限制：** 单次任务最多生成 5 张（num 只能是 "1"～"5"）；比例仅 1:1 / 3:4；`num`≤5；ratio∈1:1/3:4
- **上传：** 最多 3 个；≤10MB；jpeg、jpg、png、bmp、webp
- **说明：** 用户要 10 张：任务1 num="5" + 任务2 num="5"，禁止一次传 num="10"。未说明张数/比例/清晰度/时长时，先按 SKILL.md「生成规格」确认一轮；禁止静默使用 example 里的大 num/imageCount。

**请求体示例**（`application/x-www-form-urlencoded`，对话内直接 POST）

```
imgUrl=https://{cdn}/.../product.jpg
prompt=我是卖这个商品的电商商家，帮我生成一张爆款首图
ratio=1:1
num=1
```

**参数**（`*` 必填）

- `imgUrl`* string — 商品图 URL，最多 3 张逗号分隔
- `prompt` string — 提示词；不传可用默认：我是卖这个商品的电商商家，帮我生成一张爆款首图
- `ratio` string — 比例（仅 App 支持的两项）；仅 1:1/3:4
- `num` string — 单次生成张数，字符串，仅 "1"～"5"；禁止传 "10"；仅 1/2/3/4/5

### 商品套图 (`product_photo_set`)

批量生成多场景商品套图。推荐先 AI 帮写商品名与卖点，再提交生成。

- **接口：** `POST /productAi/productSuiteGenerate`
- **硬限制：** 单次任务张数由 suiteTypes 决定：智能匹配（suiteTypes=""）约 5 张；自定义最多勾选 9 种类型，每种 1 张；`suiteTypes`≤9；ratio∈1:1/3:4
- **上传：** 最多 3 个；≤10MB；jpeg、jpg、png、bmp、webp
- **建议先调 ai_help_write 回填：** `userInputRequirement`, `productName`
- **说明：** 张数由 suiteTypes 决定，没有 num 字段。suiteTypes 枚举：1 白底图 / 2 引流主图 / 3 细节展示 / 4 核心卖点 / 5 场景带入 / 6 角度展示 / 7 商品特效 / 8 竞品对比 / 9 结构特写。智能匹配传空串，约 5 张；自定义最多 9 种，每种 1 张。未说明张数/比例/清晰度/时长时，先按 SKILL.md「生成规格」确认一轮；禁止静默使用 example 里的大 num/imageCount。

**请求体示例**（`application/x-www-form-urlencoded`，对话内直接 POST）

```
imgUrl=https://{cdn}/.../product.jpg
productName=便携榨汁杯
ratio=1:1
mode=1
suiteTypes=
imageCount=1
userInputRequirement=1.核心卖点
- 一键榨汁
2.适用人群
- 上班族
3.期望场景
- 厨房/办公室
```

**参数**（`*` 必填）

- `imgUrl`* string — 商品图 URL，最多 3 张逗号分隔
- `productName`* string — 商品名称；可先调 ai_help_write 获取
- `ratio` string — 比例（仅 App 支持的两项）；仅 1:1/3:4
- `mode` number — 生图模式，当前固定传 1（标准生图）
- `suiteTypes` string — 自定义类型编号逗号串（1～9 最多勾选 9 种）；空字符串=智能匹配约 5 张；不要用 num 控张数
- `imageCount` number — 每种类型张数，App 固定传 1，勿改
- `userInputRequirement` string — 普通模式卖点&要求；与 commentAnalyzeTaskNo 互斥
- `commentAnalyzeTaskNo` string — 洞察模式：评论洞察任务号；传此字段时不要再传 userInputRequirement

### 电商详情页 (`detail_page_gen`)

根据商品图与卖点生成详情页长图。提交前应先 AI 帮写商品名与 vocInfo。

- **接口：** `POST /productAi/productDetailDescriptionGenerate`
- **硬限制：** 单次 imageCount 仅允许 1 / 6 / 7 / 8 / 9 / 10（默认 9），不是任意 2～5；`imageCount`∈1/6/7/8/9/10；ratio∈1:1/3:4/9:16
- **上传：** 最多 5 个；≤10MB；jpeg、jpg、png、bmp、webp
- **建议先调 ai_help_write 回填：** `vocInfo`, `productName`
- **说明：** imageCount 与 App 一致：1、6、7、8、9、10。不要假设任意整数都合法。未说明张数/比例/清晰度/时长时，先按 SKILL.md「生成规格」确认一轮；禁止静默使用 example 里的大 num/imageCount。

**请求体示例**（`application/x-www-form-urlencoded`，对话内直接 POST）

```
imgUrl=https://{cdn}/.../product.jpg
productName=便携榨汁杯
vocInfo=核心卖点：一键榨汁、易清洗
生成要求： 图片内要有文案出现
language=简体中文
imageCount=6
ratio=3:4
```

**参数**（`*` 必填）

- `imgUrl`* string — 商品图 URL，最多 5 张
- `productName`* string — 商品名称
- `vocInfo`* string — 商品说明/卖点；建议 AI 帮写后追加文案要求
- `language` string — 语言，默认简体中文
- `style` string — 配色/风格描述
- `imageCount` number — 详情页张数，仅允许下列枚举；未说明时先问用户，勿默认 9；仅 1/6/7/8/9/10
- `ratio` string — 比例；仅 1:1/3:4/9:16

### 短视频生成 (`short_video_gen`)

商品短视频。标准版 mode=i2v + model=happyhorse（首帧图生视频）；专业版 mode=r2v + model=seedance（参考图/视频）。两项必须一起传，缺 model 会被拒。

- **接口：** `POST /productVideoGenerate/createTask`
- **硬限制：** 单次 1 条视频；duration 4～15 秒；resolution 720P/1080P；标准版无 ratio；ratio∈16:9/9:16/1:1/4:3/3:4
- **上传：** 最多 5 个（视频≤3）；≤100MB；图片 jpeg/png/webp 等 ≤30MB；视频 mp4/mov ≤100MB（2～15s）；参考音频可选：wav/mp3，≤15MB，1～10s
- **说明：** 单次 1 条视频，多条拆多次。标准版：mode=i2v、model=happyhorse，勿传 ratio。专业版：mode=r2v、model=seedance，可传 ratio。不要传 extra。未说明张数/比例/清晰度/时长时，先按 SKILL.md「生成规格」确认一轮；禁止静默使用 example 里的大 num/imageCount。

**请求体示例**（`application/x-www-form-urlencoded`，对话内直接 POST）

```
imgUrls=https://{cdn}/.../first_frame.jpg
prompt=镜头缓缓推进，展示商品细节
mode=i2v
model=happyhorse
duration=4
resolution=720P
```

**专业版请求体示例**（`mode=r2v` 必须搭配 `model=seedance`）

```
imgUrls=https://{cdn}/.../ref.jpg
prompt=镜头缓缓推进，展示商品细节
mode=r2v
model=seedance
ratio=9:16
duration=4
resolution=720P
```

**参数**（`*` 必填）

- `imgUrls` string — 图片 URL，逗号分隔；标准版为首帧图
- `videoUrls` string — 参考视频 URL（专业版）
- `voiceUrls` string — 参考音频 URL（专业版可选）
- `prompt`* string — 视频描述，可含 [图1] 占位
- `mode`* string — i2v=标准版 / r2v=专业版；仅 i2v/r2v
- `model`* string — 与 mode 配套：i2v 必须 happyhorse，r2v 必须 seedance；不要省略；仅 happyhorse/seedance
- `ratio` string — 比例（仅专业版；标准版跟随首帧）；仅 16:9/9:16/1:1/4:3/3:4
- `duration` number — 时长（秒），4～15，默认 4
- `resolution` string — 清晰度；仅 720P/1080P
- `promptExtend` boolean — 智能改写（专业版）

### 全能作图 (`freedom_image_gen`)

通用 AI 作图。按子类型调用不同接口：base / model_dress / product_trans / jewelry。

- **接口：** `POST /productAi/commonImgGen（及子类型接口）`
- **硬限制：** 单次任务最多生成 5 张（num 为数字 1～5）；`num`≤5；ratio∈1:1/3:4/9:16
- **上传：** 最多 5 个；≤20MB；jpeg、jpg、png、bmp、webp；宽高 240～8000px，比例 1:8～8:1
- **说明：** 单次 num 最大 5。subType 只用于选接口，不要放进请求 body。未说明张数/比例/清晰度/时长时，先按 SKILL.md「生成规格」确认一轮；禁止静默使用 example 里的大 num/imageCount。 子类型接口映射（body 均为 imgUrl/prompt/ratio/num）：
- base → POST /productAi/commonImgGen（taskType 50，自由作图）
- model_dress → POST /productAi/modelDressUp（taskType 30，模特换装）
- product_trans → POST /productAi/productTrans（taskType 40，跨境翻译）
- jewelry → POST /productAi/jewelryTryOn（taskType 80，服饰穿戴/首饰上身）

**请求体示例**（`application/x-www-form-urlencoded`，对话内直接 POST）

```
imgUrl=https://{cdn}/.../ref.jpg
prompt=参考[图1]生成同款电商主图
ratio=1:1
num=1
```

**参数**（`*` 必填）

- `subType`* string — 子类型 → 接口映射见下方 notes；仅 base/model_dress/product_trans/jewelry；不要放进请求 body
- `imgUrl`* string — 参考图 URL，多图逗号分隔
- `prompt`* string — 提示词，可含 [图1]
- `ratio` string — 比例；仅 1:1/3:4/9:16
- `num` number — 单次生成张数，数字，仅 1～5；仅 1/2/3/4/5

### 评论洞察 (`comment_analyze`)

分析商品评论并生成洞察报告。创建任务只接收 fileUrl + title；拼多多链接需客户端先拉评论再上传。

- **接口：** `POST /commentAnalyze/createTask`
- **硬限制：** 单次任务生成 1 份评论洞察报告
- **上传：** 最多 1 个；xlsx、xls（评论 Excel）或先转成 md 再上传；拼多多链接不能直接传 createTask，需客户端拉评后上传 fileUrl
- **说明：** pddGoodsLink 不是 createTask 字段。洞察完成后，其 TaskNo 可给商品套图的 commentAnalyzeTaskNo 使用。

**请求体示例**（`application/x-www-form-urlencoded`，对话内直接 POST）

```
title=便携榨汁杯
fileUrl=https://{cdn}/.../comments.md
```

**参数**（`*` 必填）

- `title`* string — 商品名称
- `fileUrl`* string — 评论文件/Markdown 的 CDN URL，多文件逗号分隔
- `extra` string — 可选 JSON 字符串，如 {"ImgUrl":"缩略图","ItemId":"商品ID"}

### 批量印花图 (`batch_poe_image`)

解压 zip 印花图并上传到目录后，配合固定产品图批量生成。提交体为 application/x-www-form-urlencoded。

- **接口：** `POST /batchPoeImgGen/createTask`
- **编码：** `application/x-www-form-urlencoded`
- **硬限制：** 单次任务张数由 zip 内成功上传的印花图数量决定，无独立 num 字段
- **上传：** 最多 1 个；≤1024MB；zip 压缩包（内含 png、jpg、jpeg、webp）；固定产品图：1 张，≤10MB；patternFolderPath 为 TOS 目录 path
- **说明：** 先解压 zip，按 upload_media 同一套 TOS 签法把每张印花图 PUT 到 `{dir}fy_upload/batch-poe-img-gen/{unix秒}/文件名`；patternFolderPath 取该目录 path（含末尾 /），不是 CDN URL。产品图仍用公网 URL。taskName 与 TaskName 同值都传。

**请求体示例**（`application/x-www-form-urlencoded`，对话内直接 POST）

```
productImgUrl=https://{cdn}/.../product.jpg
patternFolderPath=xxx/fy_upload/batch-poe-img-gen/1710000000/
prompt=将印花自然贴合到产品表面
ratio=1:1
taskName=patterns.zip
TaskName=patterns.zip
```

**参数**（`*` 必填）

- `productImgUrl`* string — 固定产品图 CDN URL
- `patternFolderPath`* string — 印花图 TOS 目录 path（UploadBase 返回的 folderPath，不是 CDN URL）
- `prompt`* string — 统一生成说明
- `ratio` string — 比例；仅 1:1/4:3
- `taskName` string — 任务名，通常用 zip 文件名
- `TaskName` string — 与 taskName 同值再传一份（App 兼容字段）

### 内容复刻 (`content_recreate`)

参考图 + 商品原图复刻风格。提交体为 URLSearchParams；imgUrl 为「参考图,商品图」逗号拼接。

- **接口：** `POST /productAi/copyImgGen`
- **编码：** `application/x-www-form-urlencoded`
- **硬限制：** 单次任务固定生成 1 张（num 必须为 "1"）；`num`≤1；ratio∈1:1/4:3
- **上传：** 最多 1 个；≤10MB；jpeg、jpg、png、bmp、webp；参考图、商品原图各 1 张，提交时合并为 imgUrl
- **说明：** 单次固定 1 张。不要把 referenceImgUrl / productImgUrl 当作请求字段；它们只是前端表单字段。

**请求体示例**（`application/x-www-form-urlencoded`，对话内直接 POST）

```
imgUrl=https://{cdn}/.../ref.jpg,https://{cdn}/.../product.jpg
prompt=保持参考图构图，替换为我的商品
ratio=1:1
num=1
```

**参数**（`*` 必填）

- `imgUrl`* string — 顺序固定：参考图URL,商品原图URL
- `prompt`* string — 复刻说明
- `ratio` string — 比例；仅 1:1/4:3
- `num` string — 固定传 "1"，单次只能 1 张；仅 1

### 爆款视频复刻 (`viral_video_recreate`)

参考爆款视频 + 商品图，复刻运镜/节奏并替换主体。提交体为 application/x-www-form-urlencoded。

- **接口：** `POST /videoRecreate/createTask`
- **编码：** `application/x-www-form-urlencoded`
- **硬限制：** 单次 1 条复刻视频；resolution 仅 720P；originDuration=源视频秒数（3～60）
- **上传：** 最多 5 个（视频≤1）；≤100MB；视频 mp4/mov ≤100MB；图 jpeg/jpg/png/webp ≤20MB；视频 3～60s；图 1～5 张；originDuration 取真实视频秒数
- **说明：** 无 ratio/num/duration 字段。易错：videoUrl 单数不是 videoUrls；originDuration 必须 >0 且贴近真实时长；多条复刻拆多次 createTask。

**请求体示例**（`application/x-www-form-urlencoded`，对话内直接 POST）

```
videoUrl=https://{cdn}/.../viral.mp4
imgUrls=https://{cdn}/.../product1.jpg,https://{cdn}/.../product2.jpg
prompt=参考[视频1]的运镜与节奏，把主体换成[图1]商品
resolution=720P
originDuration=12
```

**参数**（`*` 必填）

- `videoUrl`* string — 参考爆款视频 CDN URL，仅 1 个（字段名单数 videoUrl）
- `imgUrls`* string — 复刻参考图 URL，逗号分隔，1～5 张
- `prompt`* string — 生成要求；可含 [视频1]/[图1]…
- `resolution`* string — 分辨率；App 当前仅 720P；仅 720P
- `originDuration`* number — 源视频时长（秒，整数 3～60）；须与真实视频时长一致，用于计费，不是输出时长
