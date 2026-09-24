---
tags:
    - 工具
---
---
> [!info] 基于 Cloudflare Pages + CopyQ + HTTP Shortcuts 的全平台静默云剪切板
> 本方案实现了**电脑（Linux/NixOS）**与**手机（Android）**之间的双向、无缝、静默剪切板同步。
> - ⚡ **核心特性**：无第三方闭源服务、全轻量化处理、静默运行无弹窗阻碍、支持双向选区同步。

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


# PC端
# 安装端

https://imgbed-5fd.pages.dev/file/files/1790238348714_http_shortcuts.zip