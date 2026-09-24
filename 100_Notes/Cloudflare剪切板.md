---
tags:
    - 工具
---
---
> [!info] 基于 Cloudflare Pages + CopyQ + HTTP Shortcuts 的全平台静默云剪切板
> 本方案实现了**电脑（Linux/NixOS）**与**手机（Android）**之间的双向、无缝、静默剪切板同步。
> - ⚡ **核心特性**：无第三方闭源服务、全轻量化处理、静默运行无弹窗阻碍、支持双向选区同步。

---
# 服务端
> 作为数据中转站，利用 Cloudflare KV 存储文本。未找到文本时返回 `404` 状态码，保持数据纯净。

在一个目录下新建文件`_worker.js`：
```js
export default {
  async fetch(request, env) {
    const room = new URL(request.url).pathname.split('/').filter(Boolean)[0];
    if (!room) return new Response(null, { status: 400 });

    if (request.method === "GET") {
      const text = await env.CLIPBOARD_KV.get(room);
      return new Response(text, { status: text ? 200 : 404 });
    }

    if (request.method === "POST") {
      await env.CLIPBOARD_KV.put(room, await request.text(), { expirationTtl: 86400 });
      return new Response(null, { status: 200 });
    }

    return new Response(null, { status: 405 });
  }
};
```

---
# PC端
> [!note]
> - **my-copy**：抓取本地剪切板并静默推送到云端。
> - **my-paste**：从云端拉取文本

NixOS配置脚本：
```sh
    (writeShellScriptBin "my-copy" ''
      TEXT=$(copyq clipboard)
      [ -n "$TEXT" ] && echo -n "$TEXT" | curl -s --data-binary @- https://paste-6bw.pages.dev/${userSettings.username}
    '')

    (writeShellScriptBin "my-paste" ''
      TEXT=$(curl -s -f https://paste-6bw.pages.dev/${userSettings.username})
      [ -z "$TEXT" ] && exit 0

      copyq copy "$TEXT"
      copyq copy text/plain "$TEXT" && copyq select 0
      copyq add "$TEXT"
    '')
```
- 注意更换链接
- 配合窗口管理器绑定按键

---
# 安卓端
> [!note] 
> 通过安卓端 `HTTP Shortcuts` 软件实现一键同步，并将触发器放入**下拉通知栏快捷开关（Quick Settings Tiles）**中。(当然也能通过`termux`脚本)

## 动作一：手机同步到云端 (copy)
- **Method**: `POST`
- **URL**: `https://pages.dev`
- **Request Body Type**: `Text / Custom Expression`
- **Body Content**: 点击右侧变量图标选择 **`{clipboard}`**

## 动作二：云端同步到手机 (paste)

- **Method**: `GET`
- **URL**: `https://pages.dev`
- **Response Display**: 选择 `No display (run in background)` (后台静默)
- **Scripting -> Run on Success (成功后运行)**:
    
    ```js
    copyToClipboard(response.body);
    showToast("已同步到手机剪切板");
    ```

## 可直接导入配置
https://imgbed-5fd.pages.dev/file/files/1790238348714_http_shortcuts.zip