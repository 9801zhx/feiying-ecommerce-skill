# 飞影电商作图

丢进一张商品图或一段参考视频，在对话里直接出电商主图、套图、详情页和短视频。

本仓库是给 Cursor、Claude、扣子等 Agent 用的 **Skill**：加载后 Agent 会按 [`SKILL.md`](./SKILL.md) 调用飞影 fyApi，创建任务并轮询结果，把生成图 / 视频地址交给你。不需要再装 MCP，也不用回到飞影 App 里点生成。

官网：[feiyingai.com](https://feiyingai.com/) · 个人中心拿 ApiKey：[feiyingai.com/#/center](https://feiyingai.com/#/center)

当前版本 **1.10.2** · License [MIT](./LICENSE)

---

## 能做什么

| 能力 | 你怎么说 | 结果 |
|------|----------|------|
| 爆款首图 | 「用这张商品图出 3 张 1:1 主图」 | 1～5 张电商主图 |
| 商品套图 | 「按智能套图出一套」 | 白底 / 场景 / 卖点等一组图 |
| 电商详情页 | 「出一套详情页，竖版」 | 1 / 6 / 7 / 8 / 9 / 10 张详情 |
| 短视频 | 「标准版 720P 出一条 6 秒」 | 图生视频；也可走专业版 |
| 全能作图 | 「换个场景 / 换装 / 跨境」 | 自由创作及相关子能力 |
| 内容复刻 | 「按这张参考图复刻商品」 | 固定 1 张 |
| 评论洞察 | 「分析这份评论表」 | 评论文件解读 |
| 批量印花 | 「这些印花铺到产品图上」 | 批量印花图 |
| 爆款视频复刻 | 「按这条爆款视频复刻」 | 参考视频 + 商品图出片 |

本地只有文件、没有公网链接时，Agent 会按 [`references/auth-and-upload.md`](./references/auth-and-upload.md) 上传后再生成。

---

## 使用前：拿到 ApiKey

1. 打开 [飞影官网](https://feiyingai.com/) 注册或登录（手机号 + 验证码）
2. 登录后会进入 [个人中心](https://feiyingai.com/#/center)
3. 复制 **ApiKey**
4. 回到 Agent 对话里发给它（不要写进仓库、不要发给别人）

积分不够时，打开充值页（把 `{your_api_key}` 换成自己的 ApiKey）：

`https://feiyingai.com/#/recharge?apiKey={your_api_key}`

不要让 Agent 代下单。

---

## 怎么加载这个 Skill

### SkillHub

SkillHub slug：`feiying-ai-api-docs`  
在 [SkillHub](https://skillhub.cn) 搜索「飞影电商作图」或该 slug，安装后即可在支持的 Agent 里使用。

### GitHub / 本地文件夹

把本仓库克隆或下载 zip，整份目录交给 Agent 当 Skill 用（入口是根目录的 `SKILL.md`）：

```bash
git clone https://github.com/9801zhx/feiying-ecommerce-skill.git
```

常见方式：

- **Cursor**：把仓库放进项目的 `.cursor/skills/`，或在对话里 @ 这个文件夹
- **Claude / Claude Code**：按客户端的 Skill / 项目知识添加本目录
- **扣子**：上传 Skill 包或填写本仓库；案例需用扣子里跑出来的公开任务链接

加载后**先发一句 ApiKey**，再提生成需求。

---

## 可以这样对 Agent 说

```
这是我的 ApiKey：xxxx

用这张商品图出 3 张 1:1 爆款首图：
https://example.com/product.jpg
```

```
按智能套图出一套，比例 3:4。
```

```
出电商详情页，imageCount 用 6，竖版。
```

```
标准版短视频，720P，6 秒。
```

没说张数 / 比例 / 清晰度 / 时长时，Agent 会先确认一轮，或按保守默认并告诉你用了什么（例如先出 1 张 1:1）。

---

## 仓库里有什么

```
├── SKILL.md                      # Agent 入口（必读）
├── manifest.json                 # 名称、版本、标签
├── README.md                     # 本说明
├── LICENSE
└── references/
    ├── tools.md                  # 各接口参数与上限
    ├── workflow.md               # 标准流程
    ├── auth-and-upload.md        # 鉴权与本地文件上传
    └── usage.md                  # Agent 内部用量计数（不必给用户看）
```

改接口或限制时，请改飞影 App 仓库里的 Skill 生成逻辑后重新打包；本仓库只放发布用的文档。

---

## 相关链接

- 飞影官网：https://feiyingai.com/
- 个人中心（复制 ApiKey）：https://feiyingai.com/#/center
- 充值：https://feiyingai.com/#/recharge
- SkillHub：https://skillhub.cn
- 本仓库：https://github.com/9801zhx/feiying-ecommerce-skill
