---
tags:
    - 日常
    - 效率
    - 工具
---

> `termux` + `cowyo` = 跨设备剪切板同步
> **注意**: **用户名要一致**
# PC端脚本(nixos)
```nix
(writeShellScriptBin "cowyo-up" ''
  TEXT=$(copyq clipboard)
  [ -z "$TEXT" ] && exit 1

  echo -n "$TEXT" | curl -s --data-binary @- https://cowyo.com/${userSettings.username}
'')

(writeShellScriptBin "cowyo-down" ''
  TEXT=$(curl -s https://cowyo.com/${userSettings.username})
  [ -z "$TEXT" ] && exit 1
  copyq copy "$TEXT"
  copyq add "$TEXT"
'')
```

## 使用
在窗口管理器中绑定键位例如(`ctrl+alt+c`/`ctrl+alt+v`)来上传拉取

# android快捷安装脚本
> 需要安装`termux`/`termux-api`/`termux-widget`软件

> `~/.shortcuts/tasks`脚本放该目录下, 后台快速执行
```sh
mkdir -p ~/.shortcuts/tasks

# 上传
cat > ~/.shortcuts/tasks/cowyo-push << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
TEXT=$(termux-clipboard-get)
[ -z "$TEXT" ] && { termux-toast "剪贴板为空"; exit 1; }
echo -n "$TEXT" | curl -s --data-binary @- https://cowyo.com/zxcfdcmv
termux-toast "已上传"
EOF
chmod +x ~/.shortcuts/tasks/cowyo-push

# 下载
cat > ~/.shortcuts/tasks/cowyo-pull << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
TEXT=$(curl -s https://cowyo.com/zxcfdcmv)
[ -z "$TEXT" ] && { termux-toast "cowyo 为空"; exit 1; }
termux-clipboard-set "$TEXT"
termux-toast "已写入剪贴板"
EOF
chmod +x ~/.shortcuts/tasks/cowyo-pull
```

## 使用
在手机桌面添加桌面小部件termux:widget，小部件中会显示相关脚本