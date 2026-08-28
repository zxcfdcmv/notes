# Croc 文件传输
> `termux` + `croc` = 跨设备快捷传输文件
> **注意**: **用户名要一致**
## PC端脚本(nixos)
```nix
(writeShellScriptBin "croc-send" ''
  export CROC_SECRET=${userSettings.username}

  # FILE=$(find ~/{Downloads,Documents,Pictures} -maxdepth 3 -type f 2>/dev/null | fzf --prompt="选择文件 > ")
  FILE=$(fd --type f --max-depth 3 . ~/Downloads ~/Documents ~/Pictures 2>/dev/null | fzf --prompt="选择文件 > ")

  if [ -n "$FILE" ]; then
      croc send "$FILE"
  fi
'')

(writeShellScriptBin "croc-recv" ''
  export CROC_SECRET=${userSettings.username}
  mkdir -p ~/Downloads/croc
  croc --yes --overwrite --out ~/Downloads/croc
'')
```

### 使用
命令行/窗口管理器/应用启动器中执行/按键，`croc-send`跳出`fzf`文件选择界面后选择文件即可传输

## android快捷安装脚本
> 需要安装`termux`/`termux-api`/`termux-widget`软件

> `~/bin/termux-file-editor`脚本可使用系统分享菜单直接分享文件

> `termux-widget`桌面小部件可显示`.shortcuts`目录下的脚本

> `.shortcuts`目录下的脚本前台执行，调用fzf选择文件

> `.shortcuts/tasks`目录下的脚本后台执行


```sh
pkg install fzf termux-api croc
mkdir -p ~/bin ~/.shortcuts/tasks

# 1. 创建分享钩子（挂到系统分享菜单）
cat > ~/bin/termux-file-editor << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
export CROC_SECRET=zxcfdcmv
termux-toast "正在发送: $(basename "$1")"
croc send "$1"
EOF
chmod +x ~/bin/termux-file-editor

# 2. 创建 Widget 快捷脚本（桌面小部件用）
cat > ~/.shortcuts/croc-send << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
export CROC_SECRET=zxcfdcmv

# 在常用目录里搜索文件，fzf 交互选择
FILE=$(find /storage/emulated/0/{Download,Documents,DCIM,Pictures} -maxdepth 3 -type f 2>/dev/null | fzf --prompt="选择文件 > ")

if [ -n "$FILE" ]; then
    termux-toast "正在发送: $(basename "$FILE")"
    croc send "$FILE"
else
    termux-toast "未选择文件"
fi
EOF
chmod +x ~/.shortcuts/croc-send

# 3. 创建接收脚本（Widget 用）
cat > ~/.shortcuts/tasks/croc-recv << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
export CROC_SECRET=zxcfdcmv
mkdir -p /storage/emulated/0/Download/croc
termux-toast "等待 croc 接收..."
croc --yes --overwrite --out /storage/emulated/0/Download/croc
EOF
chmod +x ~/.shortcuts/tasks/croc-recv
```

### 注意
注意修改其中的用户名
### 使用
在手机桌面添加桌面小部件termux:widget，小部件中会显示相关脚本，点击脚本后选择文件传输