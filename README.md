# 四级备考教练 · GitHub Pages 静态站点

本目录就是**站点根目录**，上传到 GitHub 仓库根目录后即可通过 Pages 生成静态链接。
前端与后端分离：页面托管在 GitHub Pages，接口走你自己的后端（经 Cloudflare Tunnel 暴露为受信任 HTTPS）。

---

## 一、文件说明

| 文件 | 作用 | 能否删除 |
|---|---|---|
| `index.html` | 主应用（诊断 / 计划 / 练习 / 工具 / 设置） | 否 |
| `login.html` | 登录页 | 否 |
| `register.html` | 邀请码注册页 | 否 |
| `config.js` | **唯一需要你修改的文件**：后端 API 地址 | 否 |
| `.nojekyll` | 关闭 Jekyll，按原样发布 | 否 |
| `404.html` | 未知路径自动回首页 | 可 |
| `.github/workflows/deploy.yml` | 可选的 Actions 自动发布 | 可（用分支发布时可删） |
| `README.md` | 本说明 | 可 |

---

## 二、发布到 GitHub Pages

### 方式 A：分支发布（最简单，推荐）

1. 在 GitHub 新建仓库，可见性选 **Public**（免费账号的 Pages 只支持公开仓库）。
2. 把**本目录内的所有文件**（包括 `.nojekyll` 和 `.github` 隐藏项）上传到仓库**根目录**。
   - 网页上传：Add file → Upload files → 把文件拖进去 → Commit。
   - `.nojekyll` 是隐藏文件，网页上传若看不到它：Add file → Create new file → 文件名填 `.nojekyll` → 内容留空 → Commit。
   - 命令行：
     ```bash
     git init
     git add .
     git commit -m "deploy: static site"
     git branch -M main
     git remote add origin https://github.com/<用户名>/<仓库名>.git
     git push -u origin main
     ```
3. 仓库 **Settings → Pages → Build and deployment**
   - Source 选 **Deploy from a branch**
   - Branch 选 **main**，目录选 **/(root)** → Save
4. 等待 1–2 分钟，访问：
   ```
   https://<用户名>.github.io/<仓库名>/
   ```

### 方式 B：GitHub Actions 自动发布

1. 同样先把文件推到仓库根目录。
2. **Settings → Pages → Source** 选 **GitHub Actions**。
3. 之后每次 push 到 `main`，`.github/workflows/deploy.yml` 会自动发布，无需手动操作。

---

## 三、配置后端地址（必须）

打开 `config.js`，把示例地址换成你的隧道地址：

```js
window.CET4_CONFIG = {
  API_BASE: "https://random-words-1234.trycloudflare.com"
};
```

要求：

- 必须是 **https**（Pages 是 https，调用 http 会被浏览器按混合内容拦截）
- **末尾不要加 `/`**
- 改完保存并提交，等待 Pages 重新发布

> 未配置或仍是示例地址时，首页会显示“还差一步：配置后端地址”的提示页，不会白屏。

---

## 四、后端需要配合的改动

后端不能原样使用，需要：

1. 允许你的 Pages 域名跨域（CORS 白名单）
2. 允许 `Authorization: Bearer` 登录态（跨站 Cookie 在 Safari 下会被拦截）
3. 登录响应返回 token

对应的文件、命令与验证方法见工作区中的 `后端适配补丁/README.md`（该目录不随站点上传）。

其中包含 Cloudflare Tunnel 的安装与常驻配置。**顺序建议：先改后端 → 起隧道 → 再填 config.js → 最后发布页面。**

---

## 五、上线后验证清单

- [ ] 打开站点自动跳到 `login.html?next=...`
- [ ] 输入账号密码能登录，并回到首页
- [ ] 首页左下角出现绿色“已同步”，修改学习记录后短暂显示“同步中…”
- [ ] 刷新页面仍保持登录（token 存在 localStorage）
- [ ] 练习页题库正常加载（首次需联网，之后有本地缓存）
- [ ] 退出登录后回到登录页，直接访问首页会再次跳转登录
- [ ] 手机浏览器（Safari / Chrome）同样能登录

---

## 六、安全与限制

- 前端把 JWT 存在 `localStorage`，有效期 1 小时；退出登录后服务端 `token_version + 1`，旧 token 立即失效。
- 本站没有第三方脚本，XSS 面较小；但仍建议不要在公共电脑上保持登录。
- 隧道地址是公网入口，请依赖后端的登录限流与账号锁定，不要在页面里放任何密钥。
- 快速隧道每次重启会换地址，需要同步更新 `config.js`；长期使用请配置命名隧道 + 自有域名。
- 仓库是公开的，任何人都能看到前端代码；数据库密码、JWT 密钥等**只保留在服务器**，不要提交。

---

## 七、常见问题

| 现象 | 排查方向 |
|---|---|
| 一直停留在登录页 | `config.js` 地址是否正确；后端 `CORS_ORIGINS` 是否包含 `https://<用户名>.github.io`（不带仓库名、不带斜杠） |
| 控制台报 CORS 错误 | 后端补丁未生效，或 `CORS_ORIGINS` 值多了路径/斜杠；用 `curl -i -X OPTIONS` 检查预检响应 |
| 登录返回 200 但没登录 | 后端登录响应里没有 `token` 字段，说明 `authController.js` 补丁未部署或未重启服务 |
| 502 / 隧道错误页 | `cloudflared` 或后端进程未运行；`journalctl -u cloudflared-cet4`、`journalctl -u cet4-backend` 查看日志 |
| 页面提示“题库加载失败” | 登录态失效或 `/api/bank` 被 CORS 拦截；先确认 `/api/verify` 是否 200 |
| 修改后页面没变化 | Pages 有缓存，等 1–2 分钟或强制刷新（Ctrl+F5）；确认提交到了正确分支 |
