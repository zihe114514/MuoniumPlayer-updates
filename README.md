# MuoniumPlayer 更新清单

这个仓库只有一个作用：给 MuoniumPlayer 客户端 mod 提供**版本清单**。
代码仓库是私有的，而客户端必须能匿名读到清单，所以清单单独放在这个公开仓库里。

## 客户端行为

- 启动时在守护线程上拉一次 `update.json`（连接 / 读取超时各 5 秒，失败静默）。
- 两个源，按顺序尝试：
  1. `https://raw.githubusercontent.com/zihe114514/MuoniumPlayer-updates/main/update.json`
  2. `https://cdn.jsdelivr.net/gh/zihe114514/MuoniumPlayer-updates@main/update.json`
- 本地缓存 6 小时，期间不再联网。
- 清单里的 `latest` 高于本地版本时，在**主菜单**弹一次提示；用户可以「之后再说」、「跳过此版本」或「不再检查更新」。
- 只发一个匿名 GET，不上传任何用户数据；`config/muonium/update_notice.json` 里把 `enabled` 改成 `false` 就彻底不联网。

## 清单格式

```jsonc
{
  "schema": 1,
  "channels": {
    // 键是 "<Minecraft 版本>-<加载器>"，加载器取 fabric / forge / neoforge
    "1.21.11-neoforge": {
      "latest": "1.5.0",              // 必填。比本地版本新才弹提示
      "critical": false,              // true 时不给「跳过此版本」
      "page": "https://github.com/...", // 下载页。只允许 https 且域名在白名单里
      "notes": "这一版改了什么"        // 可空
    }
  },
  "fallback": { "latest": "1.5.0" }   // 找不到对应 channel 时用这个
}
```

查表顺序：`<mc>-<loader>` → `<mc>` → `fallback`。

`page` 的域名白名单（客户端硬编码，不在名单里就退回本仓库首页）：
`github.com` / `githubusercontent.com` / `cdn.jsdelivr.net` / `modrinth.com` / `gitee.com`。
这是为了防止清单被篡改后把用户导去任意站点。客户端**永远不会**自动下载或替换 jar。

## 发一个新版本要做什么

1. 把新 jar 发到 Releases（或任何白名单域名上的页面）。
2. 改这里对应 channel 的 `latest` / `page` / `notes`。
3. 完事。老客户端最多 6 小时后就会开始提示。

