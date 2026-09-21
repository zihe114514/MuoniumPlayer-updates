# MuoniumPlayer 清单（更新 + 公告二合一）

这个仓库只有一个作用：给 MuoniumPlayer 客户端 mod 提供一份**公开可读的清单**。
代码仓库是私有的，而客户端必须匿名读到它，所以清单单独放在这里。

一份 `update.json` 同时管两件事：

| 段落 | 干什么 | 弹几次 |
| --- | --- | --- |
| `channels` / `fallback` | 版本更新提示 | 有新版每次启动都弹，「先不更新」每个版本 3 次 |
| `announcements` | 公告中心 | 有未读时启动弹一次，关掉即整批记为已读 |

## 客户端行为

- 启动时在守护线程上把清单源**全部同时拉一遍**（每个源一条线程，连接 / 读取超时各 5 秒，
  整体等待预算 12 秒，失败静默），拿到多份就取**版本号最高的那一份**。
- 三个源：
  1. `https://ghfast.top/https://raw.githubusercontent.com/zihe114514/MuoniumPlayer-updates/main/update.json`
  2. `https://raw.githubusercontent.com/zihe114514/MuoniumPlayer-updates/main/update.json`
  3. `https://gcore.jsdelivr.net/gh/zihe114514/MuoniumPlayer-updates@main/update.json`

  **取最高而不是「谁先答用谁」**：各源缓存策略不同，同一时刻可能一个给新清单、一个给几小时前的
  旧清单（jsDelivr 对 `@main` 这种 branch 引用是 12 小时）；而每次谁快谁慢都可能变，所以「先答」
  不代表「更新」。全部试一遍取最高，才不会因为恰好先问到缓存源而漏掉新版本。

  公告与更新用**同一条取最高规则**（拿胜出那一份清单里的 `announcements`），所以两份内容永远来自
  同一个源，不会出现「公告是新的、版本号是旧的」这种拼接结果。平局时按源序号取靠前的，而源 1 的
  缓存只有 5 分钟 —— 也就是说平局恰好总是选到最新鲜的那一份。

  > 2026-09-16：去掉了 `cdn.jsdmirror.com` 与 `cdn.jsdelivr.net` 两个源
  > （前者是 jsDelivr 系镜像，后者对 `@main` 是 12 小时缓存），剩下上面三个。
- 本地缓存 30 分钟，期间不再联网（更新与公告各自记账，互不牵连）。
- 只发匿名 GET，不上传任何用户数据；客户端也**永远不会**自动下载或替换 jar。

## 清单格式

```jsonc
{
  "schema": 2,
  "channels": {
    // 键是 "<Minecraft 版本>-<加载器>"，加载器取 fabric / forge / neoforge
    "1.21.11-neoforge": {
      "latest": "1.6.0",              // 必填。比本地版本新才弹更新提示
      "critical": false,              // true 时不给「先不更新」，直接要求更新
      "page": "https://melodifymc.cc.cd/download", // 下载页。只允许 https 且域名在白名单里
      "notes": "这一版改了什么"        // 可空
    }
  },
  "fallback": { "latest": "1.6.0" },  // 找不到对应 channel 时用这个
  "announcements": [                  // 可空数组；没有这个字段就是「暂时没有公告」
    {
      "id": "2026-09-21-160-room-automix",  // 必填，见下面的「id 规则」
      "date": "2026-09-21",                 // 选填，卡片右上角展示
      "kind": "feature",                    // 选填：feature / fix / event / warning，其它值按 info 配色
      "title": "一句话标题",                 // 必填，空的整条会被客户端丢掉
      "body": "正文，\\n 分段",              // 必填
      "link": "https://...",                // 选填，走同一份域名白名单
      "linkText": "查看说明"                 // 选填，默认「打开链接」
    }
  ]
}
```

`channels` 的查表顺序：`<mc>-<loader>` → `<mc>` → `fallback`。

`page` / `link` 的域名白名单（客户端硬编码，不在名单里就退回客户端内置的官网下载页
`https://melodifymc.cc.cd/download`，链接则整条不显示）：
`melodifymc.cc.cd` / `github.com` / `githubusercontent.com` / `jsdelivr.net` / `modrinth.com` / `gitee.com`。
这是为了防止清单被篡改后把用户导去任意站点。

> 官网域名是后加进白名单的，所以**老版本客户端**读到指向官网的 `page` 会判定不合规、退到它们
> 各自内置的 GitHub 仓库页 —— 能打开，只是不在官网。等用户升级上来即生效。

## 公告的写法

- **数组里靠前的先显示**，最新的写在最前面。客户端不做时间排序，只按这里的顺序摆。
- **`id` 规则**：客户端按 `id` 记「读过没有」，所以
  - 必须唯一；
  - **改了内容就要换 `id`**（比如末尾加 `-v2`）—— 不换的话老用户已经把它记成读过，
    永远看不到你改的新内容；
  - 已读记录里多余的 `id` 不影响任何东西，只是留在本地（客户端只保留最近 100 条）。
- **撤回一条公告 = 从数组里删掉它**，不用改 `id` 也不用做别的。
- `body` 里的 `\n` 是**分段**，段内由客户端按窗口宽度自己折行；整页可以滚动，所以正文没有长度上限，
  但建议一条控制在三四段以内 —— 公告是给人扫一眼的，不是给人读文档的。
- `kind` 只影响卡片上那枚徽标的颜色和图标，不认识的值一律按默认的 info 处理，不会画不出来。
- 公告**与渠道无关**：所有版本、所有加载器看到同一份。要只对某个渠道说话，写进那条 channel 的
  `notes` 里（那里是「有更新」提示的正文）。

## 发一个新版本要做什么

1. 把新 jar 发到官网下载页 / Releases（或任何白名单域名上的页面）。
2. 改这里对应 channel 的 `latest` / `page` / `notes`。
3. 顺手往 `announcements` 里加一条这一版的公告（可选，但发布说明没几个人会去仓库翻）。
4. 完事。客户端最多 30 分钟后就会开始提示。

> 「取最高」也意味着：**发版时不要手滑把 `latest` 写成一个还没发布的更高版本号** —— 各源内容一样，
> 但客户端一旦读到就会提示用户更新，而用户下载到的还是旧包。真要撤回只能再推一次。

## 改完怎么确认生效

`update.json` 必须是**合法 JSON**（客户端解析失败当作「什么都没有」，不会报错给你看）。
推之前本地跑一下：

```bash
python -c "import json,sys; d=json.load(open('update.json',encoding='utf-8')); \
print('channels', len(d['channels']), 'announcements', len(d.get('announcements', [])))"
```

生效时间：源 1 / 源 2 的缓存只有 5 分钟，加上客户端自己的 30 分钟缓存，正常情况下**十几分钟到
半小时内**有人拿到；只有当这两个都不通、只剩 jsDelivr 时，那 12 小时缓存才会把生效时间拖后。
公告窗底部有一个「刷新」按钮，点它会强制立刻重拉一次 —— 自己测试时用这个最快。
