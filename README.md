# Roche 网易云音乐插件（roche-music-player）

在 Roche 里播放网易云音乐：搜索、歌单、每日推荐、听歌记录、灵动岛歌词，以及角色点歌和“当前在听”注入。

当前版本：**v3.5.1**

---

## 1. 安装

在 Roche 的插件安装页填这一个链接：

```txt
https://cdn.jsdelivr.net/gh/luyi90720-sys/roche-music-player@main/manifest.json
```

不要填 GitHub 网页链接（`github.com/.../blob/...`）。

> manifest 里的 `entry` 使用 `@main`，推送后自动跟随最新代码。
> 如果你更喜欢固定版本，可以把 `@main` 换成具体 commit，例如 `@85840c3`。

---

## 2. 第一次使用

1. 打开「网易云音乐」App。
2. 进入「设置」。
3. **填写自部署的网易云 API 地址**（必填）。
   - 这是 NeteaseCloudMusicApi 兼容实例，例如 `https://你的实例.vercel.app`。
   - 没填地址时，插件会拒绝所有网易云请求，不会走任何公共后端。
4. 登录，二选一：
   - 点「登录」扫码；
   - 或在设置里粘贴 Cookie：`MUSIC_U=值; __csrf=值`。
5. 点「保存设置」。

---

## 3. 权限

只申请两个：

| 权限 | 用途 |
|---|---|
| `storage` | 保存播放列表、音量、灵动岛位置等设置 |
| `ui` | 打开/关闭插件页面、Toast、确认弹窗 |

不再申请 `memory:read`（旧版本申请了但从未使用）。

---

## 4. 数据存在哪（重要）

### 普通设置 → `roche.storage`

- `rmp_playlist`：播放列表
- `rmp_volume`、`rmp_quality`、`rmp_play_mode`
- `rmp_island_*`：灵动岛位置/显示/滚动模式
- `rmp_theme_dark`、`rmp_lyrics_full_inject`
- `rmp_netease_api_base`：你填的 API 地址

这些是插件自己的设置，体积小，可以跟随 Roche 的数据管理。

### 账号凭据 → 插件独立 IndexedDB

- 数据库名：`roche-plugin-music-player`
- 内容：网易云 Cookie、用户昵称/头像/VIP 信息

**为什么要单独放：**

- 旧版本把 Cookie 存在 `roche.storage`（也就是 Roche 主存储）里。
- Roche 的数据导出、云同步或后端推送会把它一起带走。
- 如果后端没有登记这些 key，就会直接报错，推送失败。
- 现在账号凭据放在插件自己的 IndexedDB，**不进入 Roche 的同步/推送数据**，冲突从源头消失。

### 旧数据会自动迁移

首次加载 v3.5.1 时：

1. 读取旧版 `roche.storage` 里的 `rmp_cookie` / `rmp_user_profile`；
2. 写入插件独立 IndexedDB；
3. 写入成功后删除旧 key。

如果迁移失败，旧 key 会保留，不会丢账号。

### 怎么彻底删除账号数据

设置 →「清除已保存的网易云账号」。

注意：插件自己的 IndexedDB 在卸载插件时**不一定会被自动清除**，所以提供了这个按钮。

---

## 5. 关闭页面 ≠ 停止播放

- 关闭 App 页面：音乐继续播，灵动岛继续显示。
- 想真正停止并释放资源：设置 →「停止播放并清理插件」。
- 卸载/禁用插件：会触发完整清理（停止播放、销毁灵动岛、移除监听和定时器）。

---

## 6. 调试日志

设置页底部有「调试日志」。

- 只在 App 打开期间接管 `console`；
- 关闭 App 或卸载插件时恢复宿主原始 `console`；
- 不再永久污染 Roche 和其它插件。

---

## 7. 常见问题

**Q：提示「请先在设置中填写自部署的网易云 API 地址」**
A：这是设计如此。插件不内置公共后端，必须填你自己的 NeteaseCloudMusicApi 实例。

**Q：扫码后一直显示待确认**
A：检查实例是否能正常访问网易云；插件已加 `timestamp` 和 `realIP` 参数绕过 CDN 缓存与部分风控。

**Q：播放失败 / 只有部分歌能播**
A：多为版权或 VIP 限制。请确认 Cookie 有效，且账号有对应权限。

**Q：Roche 数据推送失败**
A：v3.5.1 已把账号 Cookie / 用户资料移出 `roche.storage`。如果仍失败，请检查是否还有旧版本残留（打开一次插件即可触发迁移），或在设置里点「清除已保存的网易云账号」。

**Q：换设备后要重新登录吗**
A：要。账号凭据只存在当前设备的插件 IndexedDB 里，不会跟着 Roche 云同步走。

---

## 8. 文件结构

```txt
roche-music-player/
  manifest.json     # 插件清单
  music-player.js   # 插件本体
  README.md         # 本说明
```

---

## 9. 隐私说明

- Cookie 只发送给你自己填写的网易云 API 实例，以及该实例返回的网易云官方音频地址。
- 插件作者不收集、不上传任何账号数据。
- 播放链接来自网易云官方接口，仅供个人学习与技术研究，请勿用于商业用途。
