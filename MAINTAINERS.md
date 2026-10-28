# 维护说明（给未来的自己）

这份文档只回答三件事：**怎么改、怎么发、哪里不能碰。**
每次发版，在最后的「更新记录」加一行就行。

---

## 0. 现状

- 插件名：网易云音乐
- 版本：3.5.1
- 仓库：`luyi90720-sys/roche-music-player`（公开）
- 线上入口：`manifest.json` → `entry`（jsDelivr）
- 没有构建工具、没有 npm、没有 Vue/React，改完直接推。

---

## 1. 文件地图

| 文件 | 作用 |
|---|---|
| `manifest.json` | 插件身份证：版本、权限、入口 |
| `music-player.js` | 全部代码（界面 + 播放 + 登录 + 聊天注入） |
| `README.md` | 给用户看的安装和使用说明 |
| `MAINTAINERS.md` | 本文件，给维护者看 |

`music-player.js` 里几个关键区块：

| 区块 | 大概位置 | 说明 |
|---|---|---|
| 日志捕获 | 文件开头 | 只在 App 打开时接管 console |
| 全局状态 `STATE` | 开头 | 所有运行时状态 |
| 网易云 API / 登录 | 中段 | `neteaseApi`、`ncmApi`、扫码登录 |
| 灵动岛 | 中段 | 悬浮层、歌词、歌单弹窗 |
| 设置界面 | 中段 | 所有设置项和按钮 |
| 账号凭据存储 | 后段 | 插件独立 IndexedDB |
| `saveSettings` / `loadSettings` | 后段 | 设置与迁移 |
| `destroyAll` | 后段 | 彻底清理 |
| 插件注册 | 文件末尾 | `window.RochePlugin.register` |

---

## 2. 怎么发版

1. 改代码。
2. 改这三处版本号：
   - `music-player.js` 里的 `BUILD_TIME`（例如 `2026-10-28-v3.5.1`）
   - `manifest.json` 的 `version`
   - `manifest.json` 的 `description`
3. 推送到公开仓库：
   - `git add manifest.json music-player.js README.md MAINTAINERS.md`
   - `git commit -m "v3.5.2: 说明"`
   - `git push luyi main`
4. 等 1～2 分钟 CDN 缓存，再打开插件验证。
5. 如果线上还是旧代码：
   - 打开 `https://cdn.jsdelivr.net/gh/luyi90720-sys/roche-music-player@main/manifest.json` 看版本号；
   - 还是旧的就换一个固定 commit 链接，或等缓存过期。

> `entry` 现在用 `@main`，推上去自动跟随。
> 想固定版本，就把 `@main` 换成具体 commit。

---

## 3. 哪里不能碰

### 3.1 账号凭据不能再写回 `roche.storage`

- 账号 Cookie / 用户资料只能放在插件自己的 IndexedDB：`roche-plugin-music-player`。
- 原因：`roche.storage` 会进入 Roche 的导出/云同步/后端推送；后端没登记这些 key 就会推送失败。
- 不要为了省事把 `rmp_cookie`、`rmp_user_profile` 写回 `roche.storage`。

### 3.2 不要再永久替换 `window.console`

- 现在用 `installConsoleCapture()` / `restoreConsoleCapture()`。
- 只在 `mount` 时装，`cleanupApp` / `onUnload` 时恢复。
- 不要改回文件一加载就替换。

### 3.3 `mount` 和 `unmount` 要对称

- `mount` 里加的事件、定时器、样式，`cleanupApp` 里要清。
- `onUnload` 必须调用 `destroyAll()`。

### 3.4 不要删 `destroyAll()` 里的步骤

它按顺序清理：App 面板 → 扫码轮询 → 灵动岛 → 音频/blob → 残留弹层 → 媒体信息 → console → 状态复位。
少一步就会留下残留。

---

## 4. 数据字典（`roche.storage`）

这些是普通设置，可以跟随 Roche 数据管理：

| key | 内容 |
|---|---|
| `rmp_playlist` | 播放列表 |
| `rmp_volume` | 音量 |
| `rmp_quality` | 音质 |
| `rmp_play_mode` | 播放模式 |
| `rmp_island_top` | 灵动岛高度 |
| `rmp_island_visible` | 是否显示灵动岛 |
| `rmp_island_scroll_mode` | 歌名 / 歌词 |
| `rmp_lyrics_full_inject` | 完整歌词注入 |
| `rmp_theme_dark` | 夜间模式 |
| `rmp_netease_api_base` | 自部署 API 地址 |
| `rmp_agreed_disclaimer` | 免责声明标记 |

这些是旧版遗留，**只允许读和删，不允许再写**：

| key | 处理 |
|---|---|
| `rmp_cookie` | 迁移到独立库后删除 |
| `rmp_user_profile` | 迁移到独立库后删除 |
| `rmp_backend` | 直接删除 |

独立库 `roche-plugin-music-player` → `credentials`：

| key | 内容 |
|---|---|
| `cookie` | 网易云 Cookie |
| `profile` | 昵称/头像/VIP |

---

## 5. 常见问题定位

| 现象 | 先看哪里 |
|---|---|
| 提示「请先填写自部署 API 地址」 | 设计如此，不是 bug |
| 登录失败 / 二维码一直待确认 | `startNcmQrLogin`，检查实例和 `realIP` |
| 播放失败 | `getSongUrl`、`fetchBlobAndPlay`、Cookie 是否有效 |
| 角色不知道在听歌 | `contextProvider`、灵动岛是否被关闭 |
| 推送失败 | 是否还有旧 `rmp_cookie` / `rmp_user_profile` / `rmp_backend` |
| 灵动岛不显示 | `createIsland`、`rmp_island_visible` |
| 关掉 App 音乐还在响 | 正常设计，用「停止播放并清理插件」 |

---

## 6. 每次改完的测试清单

1. 安装 / 更新插件，能打开。
2. 设置里填 API 地址，保存。
3. 扫码或粘贴 Cookie，能登录。
4. 搜索一首歌，能播放。
5. 切歌、暂停、拖进度、调音量。
6. 点「关闭」关掉 App，音乐应继续。
7. 重新打开 App，状态正常。
8. 设置 →「停止播放并清理插件」，音乐停、灵动岛消失。
9. 再打开 App，能重新初始化。
10. 卸载/禁用插件，不应残留声音和悬浮层。

---

## 7. 已知风险 / 未完成

- 后端推送只在插件侧修好了；如果后端连 `plugin_storage` 整个都不收，还需要后端加放行。
- 没有在真实 Roche 客户端里跑过完整回归，只做了语法检查。
- Git 还有两个远端（`luyi` 公开、`yechen` 私有旧仓），且 remote URL 里带明文 token。
- 灵动岛仍是手写 `position: fixed`，没换成宿主 `minimize`。
- `rmp_extended_sources` 只读不写，是遗留，不影响运行。

---

## 8. 回滚

- 代码回滚：`git revert <commit>` 或切回上一个 tag。
- 线上回滚：把 `manifest.json` 的 `entry` 改回旧 commit，推上去。
- 数据回滚：账号数据在插件独立 IndexedDB，卸载插件不一定清；需要时用「清除已保存的网易云账号」。

---

## 9. 更新记录

| 版本 | 日期 | 改了什么 |
|---|---|---|
| 3.5.1 | 2026-10-28 | 账号凭据移出 `roche.storage`；console 可恢复；补 `destroyAll` 和卸载清理；权限去掉 `memory:read`；补 README / 维护说明 |
| 3.5.0 | 2026-08-08 | 网易云歌单与播放列表独立 + 歌单缓存 |
