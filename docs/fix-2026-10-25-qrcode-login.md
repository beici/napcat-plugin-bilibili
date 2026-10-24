# 修复记录：扫码登录成功后提示「解析登录凭据失败」（2026-10-25）

## 现象

在插件 WebUI 打开扫码登录，手机 B 站扫码并在手机上点击「确认登录」后，B 站侧显示登录成功，但插件持续提示**「解析登录凭据失败」**，WebUI 仍显示未登录，凭据文件中不会写入 `credential` 字段。

## 根因

插件原逻辑只支持一种凭据获取方式：从扫码轮询接口
`https://passport.bilibili.com/x/passport-login/web/qrcode/poll?qrcode_key=...`
返回的 `data.url` **查询参数**中提取 `SESSDATA` / `bili_jct` / `DedeUserID`：

```
data.url = "https://passport.biligame.com/crossDomain?DedeUserID=xxx&SESSDATA=xxx&bili_jct=xxx&..."
```

B 站近期调整了扫码登录流程：**`data.url` 中不再携带上述 Cookie 参数**，登录态改由轮询接口的
**HTTP `Set-Cookie` 响应头**直接下发。旧解析逻辑因此拿不到三个必要字段，返回
`解析凭据失败: 缺少必要的 Cookie 字段`，登录流程在最后一步断开。

## 修复内容

修改文件：`src/services/bilibili-login-service.ts`

1. **新增 `parseCredentialFromSetCookie(headers)`**：解析轮询接口响应的 `Set-Cookie` 头，
   逐条取 `;` 前的 `key=value`，当 `SESSDATA` / `bili_jct` / `DedeUserID` 三者齐备时构造凭据对象
   （`sessdata` 做 `decodeURIComponent`，`refresh_token` 置空，记录 `login_time`）。
   对 `Headers.getSetCookie()` 做了防御性访问，兼容未暴露该方法的运行时。
2. **SUCCESS 分支改为双通道解析**：

   ```ts
   const credential =
       parseCredentialFromUrl(pollData.url, pollData.refresh_token) ||
       parseCredentialFromSetCookie(response.headers);
   ```

   优先沿用旧的 URL 参数解析（保持对旧流程的兼容），失败时回退到 Set-Cookie 解析。
3. **新增调试日志**：扫码成功时输出 `urlLen` / `urlHead`（前 70 字符）/ `setCookieCount`，
   便于后续 B 站再次调整响应格式时快速定位。

## 兼容性影响

- 扫码登录：修复。
- 账号登录后，视频解析请求会携带登录 Cookie，同时解决了 2026-10-21 起
  `api.bilibili.com` 对「浏览器 UA + 无 Cookie」请求返回 **HTTP 412** 的反爬拦截
  （未登录状态下视频链接解析会提示「无法获取视频信息，跳过」）。
- 对旧版 B 站返回格式（URL 参数携带 Cookie）保持完全兼容。

## 验证

在 NapCat Shell 方式部署的线上环境实测：

1. 应用补丁并重启 NapCat；
2. WebUI 扫码 → 手机确认 → 日志出现 `扫码登录成功！`、`B 站登录凭据已保存`，
   `config.json` 写入 `credential` 字段；
3. `vite build` 构建通过。

## 顺带说明：构建产物与热更新

NapCat Shell 方式部署时插件代码位于
` NapCat安装目录/QQ/resources/app/app_launcher/napcat/plugins/<插件名>/`，
直接替换其中的构建产物 `index.mjs` 并重启 NapCat 即可生效；源码仓库修改后执行
`pnpm install && pnpm build`，将 `dist/` 内容同步到上述目录。
