# 鉴权与上传

用户已给公网 URL 时跳过本页，直接生成。只有本地文件才上传。

## ApiKey 说明（必读）

- 调 fyApi 时把用户的 **ApiKey** 放进请求头 **`Token`**
- **文档不写真实 ApiKey**，请用户在当前对话里发送；收到后再填请求头 `Token`
- **不要与下列 token 混淆：**
  - 上传凭证 `GET /util/bosSessionTokenV2` 返回的 `result.token` 是 **TOS 临时 STS**，不是 ApiKey
  - MCP 的 `set_fy_token` 只是在对话里写入 ApiKey；Skill **直接 HTTP** 时在 Header 写 `Token` 即可
- **用户还没有 ApiKey：**
  1. 打开飞影官网 `https://feiyingai.com/` 注册或登录（手机号 + 验证码）
  2. 登录成功后会**自动进入个人中心** `https://feiyingai.com/#/center`
  3. 在个人中心复制 **ApiKey**，回到对话发给你（勿泄露）
- **积分不足（创建任务失败或余额不够）：**
  - **不要**调用充值/下单接口
  - 新窗口打开 `https://feiyingai.com/#/recharge?apiKey={your_api_key}`（必须用**当前用户 ApiKey** 替换 `{your_api_key}`）
  - 用户付完后继续原任务

## 鉴权

- **Base URL：** `https://safe2.zzbtool.com/fyApi`
- **fyApi Header：** `Token: {your_api_key}`；POST 再加 `Content-Type: application/x-www-form-urlencoded`
- **禁止**调用 `/open/v1/*`（不存在，会 404）

先探活：

```
GET https://safe2.zzbtool.com/fyApi/member/info
Token: {your_api_key}
```

## 上传（强制）

- 只调 fyApi `GET /util/bosSessionTokenV2`（Header: Token，值为 ApiKey）拉凭证，再按 `tools.md` 的 `upload_media` 对 TOS 签名 PUT
- **上传凭证：** `https://safe2.zzbtool.com/fyApi/util/bosSessionTokenV2?type=static_file`，Header 仅 `Token`（无需 `zzb-time` / `zzb-sign`）。不要改走 CDN 域名
- **禁止**对 `static2.zzbtool.com` 发上传请求
- **TOS PUT：** 发往 `{bucket}.{endpoint主机}`，用凭证 `ak/sk/token` 做 `TOS4-HMAC-SHA256`
- **公网 URL：** `https://{result.cdn}/` + 分段 encode 的 key（`cdn` 以本次凭证为准，不要写死域名）
- 对话内当场算 HMAC，不要在用户项目里新建脚本；算不了就请用户发一张可打开的图链，继续生成

批量印花目录：`{dir}fy_upload/batch-poe-img-gen/{unix秒}/`，`patternFolderPath` 用这个目录 path（含末尾 `/`），不是 CDN URL。

## 积分不足

不要自己创建充值订单。新窗口打开 `https://feiyingai.com/#/recharge?apiKey={your_api_key}`（用当前用户 ApiKey 替换 `{your_api_key}`，query 参数名是 `apiKey`）。用户付完后页面会尝试自动关闭，再继续任务。没有 ApiKey 时，请用户打开 `https://feiyingai.com/` 登录；登录后会自动进入 `https://feiyingai.com/#/center`，在个人中心复制 ApiKey。
