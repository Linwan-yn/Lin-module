# Lin-Shizuku 模块开发指南

基于 `Joyose掉帧自动优化_v1.0` 实测结构整理。Lin 模块与 Magisk / KernelSU 模块在**设计哲学**上不同：它不刷系统分区、不依赖内核，而是通过 **Shizuku 提供的 root 权限** 执行脚本，用 **WebUI** 与用户交互。适合做「运行时进程监控 / 系统调优 / 服务管理」类模块。

---

## 一、Lin 模块 vs Magisk / KernelSU

| 维度 | Magisk / KernelSU | Lin-Shizuku |
|---|---|---|
| 运行环境 | 内核级 / 系统分区 | Shizuku（root 或 ADB 权限） |
| 安装位置 | `/data/adb/modules/<id>/` | `/data/local/tmp/Lin-Shizuku/module/<id>/` |
| 系统修改 | systemless overlayfs / magic mount | **不支持**，只做运行时操作 |
| 启停机制 | `post-fs-data.sh` / `service.sh` 自动执行 | 模块页开关 → `run.sh` 的 `#on` / `#off` 块 |
| 开机自启 | 有 | **无**（40 轮 APK 只自启 Shizuku） |
| 安装钩子 | `customize.sh` 自动执行 | 无（安装 = 解压 + `module.json` 声明） |
| 卸载钩子 | `uninstall.sh` 自动执行 | 无（`uninstall.sh` 是手动清理入口） |
| WebUI | 需 KernelSU 的 `kernelsu` npm 包 | 经 Lin 桥 `ksu.exec(cmd, opts, cb)` |
| 元数据 | `module.prop` | `module.prop` + `module.json` 双份 |

**结论**：Lin 模块本质是「Shizuku 权限下运行的脚本 + WebUI 控制台」，能力边界 = shell 能做什么。

---

## 二、目录结构

```

/data/local/tmp/Lin-Shizuku/module/<module_id>/
├── module.prop              # 模块元信息（必需）
├── module.json              # WebUI 声明（必需，含 webui 入口）
├── run.sh                   # 模块页开关入口（#on / #off 块）
├── service.sh               # 可选：开机钩子（保留）
├── stop.sh                  # 可选：手动停止入口
├── uninstall.sh             # 可选：手动清理入口
├── README.md                # 可选：说明文档
│
├── config/                  # 配置目录（推荐）
│   ├── module.conf          # 运行时扁平键值（脚本读）
│   └── settings.json        # WebUI 权威配置（JSON）
│
├── scripts/                 # 脚本目录（推荐分层）
│   ├── core/                # start / stop / restart
│   ├── config/              # read / write
│   ├── utils/               # logger / status
│   └── monitor.sh           # 主逻辑
│
├── webroot/                 # WebUI 根目录（必需，若提供 WebUI）
│   └── index.html           # 入口
│
└── .runtime/                # 运行时文件（脚本自建，不打包）
├── .pid                 # 进程 PID
├── .state               # 状态键值
├── .monitor.log         # 日志
└── .cooldown            # 冷却时间戳

```

**硬性约定**：
- `module.prop` 和 `module.json` 都必须存在
- WebUI 必须在 `webroot/index.html`
- 所有脚本路径**必须硬编码**为绝对路径（Lin 桥执行时不继承 `$PWD`）

---

## 三、module.prop

```

id=joyse_jank_opt
name=Joyose掉帧自动优化
version=v1.0
versionCode=1
author=Lin
description=锁帧/大幅掉帧强停一次Joyose，冷却1分钟防负优化；可自定义游戏监控列表，WebUI控制台

```

规则：
- `id` 必须匹配 `^[a-zA-Z][a-zA-Z0-9._-]+$`（不能以数字、短横线开头）
- `versionCode` 必须是整数
- 换行必须用 `LF`，不要 `CRLF`
- 不要有 BOM

---

## 四、module.json

```json
{
  "id": "joyse_jank_opt",
  "name": "Joyose掉帧自动优化",
  "version": "v1.0",
  "versionCode": 1,
  "author": "Lin",
  "description": "...",
  "webui": "webroot/index.html"
}
```

字段：

· webui：WebUI 入口相对路径，Lin 管理器根据它打开 WebView
· 其余字段与 module.prop 冗余，兼容不同版本的管理器

---

五、run.sh：模块页开关入口

这是 Lin 模块最核心的约定。模块页开关切换时，Lin 管理器会从 run.sh 中抽取 #on / #off 块执行。

```sh
#!/system/bin/sh
# 模块页开关：开 → 执行 #on 块；关 → 执行 #off 块

# 手动执行入口（终端 sh run.sh 默认启动）
MODDIR="/data/local/tmp/Lin-Shizuku/module/joyse_jank_opt"
sh "$MODDIR/scripts/core/start.sh"
exit 0

#on
MODDIR="/data/local/tmp/Lin-Shizuku/module/joyse_jank_opt"
sh "$MODDIR/scripts/core/start.sh"

#off
MODDIR="/data/local/tmp/Lin-Shizuku/module/joyse_jank_opt"
sh "$MODDIR/scripts/core/stop.sh"
```

关键点：

1. #on / #off 块会被单独抽取执行，块内必须自带 MODDIR，不要依赖块外变量
2. 块内不要写 exit 0（会提前结束抽取的脚本）
3. 手动入口放在最前面 + exit 0，避免手动执行时进入 #on 块
4. 开关是唯一启停来源，WebUI 不提供启停按钮

---

六、WebUI 与 Lin 桥

桥注入

Lin 管理器把 JS 桥注入到 WebView 的 window.ksu：

```js
ksu.exec(cmd, opts, callbackName)
```

实际签名（与 KernelSU 官方 kernelsu npm 包不同）：

· cmd：shell 命令字符串
· opts："{}" 或选项字符串（Lin 版本传空对象）
· callbackName：回调函数名，Lin 通过 window[callbackName](errno, stdout, stderr) 回调

通用封装（推荐直接抄）

```js
var MODDIR = "/data/local/tmp/Lin-Shizuku/module/joyse_jank_opt";

function sh(cmd, cb) {
  var n = "cb" + Date.now().toString(36) + Math.floor(Math.random() * 1e6).toString(36);
  var done = false;
  window[n] = function(errno, out, err) {
    if (done) return;
    done = true;
    clearTimeout(tmr);
    delete window[n];
    if (cb) cb(out, err);
  };
  var tmr = setTimeout(function() {
    if (done) return;
    done = true;
    delete window[n];
    if (cb) cb("", "«命令超时»");
  }, 20000);
  try { ksu.exec(cmd, "{}", n); } catch (e) {
    if (!done) { done = true; clearTimeout(tmr); delete window[n]; if (cb) cb("", String(e)); }
  }
}
```

要点：

· 必须加超时兜底（20s），否则桥异常时回调永不返回
· 必须防重复回调（done 标志），桥可能多次触发
· 回调后 delete window[n] 防止内存泄漏

与脚本交互的三种模式

A. 读配置（base64）

```js
sh("sh " + MODDIR + "/scripts/config/read.sh all_b64", function(out) {
  var state = JSON.parse(base64Decode(out.trim()));
});
```

B. 写配置（base64 传 JSON，防注入）

```js
var payload = base64Encode(JSON.stringify(state));
sh("sh " + MODDIR + "/scripts/config/write.sh set_b64 '" + payload + "'", function(out) {
  if ((out || "").indexOf("ERR") === 0) { toast(out); return; }
  toast("已保存");
});
```

C. 读状态（JSON）

```js
sh("sh " + MODDIR + "/scripts/utils/status.sh", function(out) {
  var s = JSON.parse(out);  // {"running":true,"pid":"1234",...}
});
```

状态字段约定

status.sh 输出标准 JSON，建议字段：

```json
{
  "running": true,
  "pid": "1234",
  "lastEvent": "锁帧(com.xxx fps 55/120) → 强停 com.xiaomi.joyose",
  "eventTime": "2026-10-01 12:34:56",
  "logSize": 12345,
  "cooling": true,
  "cooldownRemain": 45
}
```

WebUI 每 5 秒轮询一次 status.sh + tail -n 30 .monitor.log。

---

七、脚本分层架构（推荐）

```
scripts/
├── core/               # 生命周期管理
│   ├── start.sh        # 幂等启动 + 心跳检测
│   ├── stop.sh         # 精确匹配 PID 停止
│   └── restart.sh      # stop + start
│
├── config/             # 配置读写
│   ├── read.sh         # get / all / all_b64
│   └── write.sh        # set_b64 → settings.json + module.conf
│
├── utils/              # 通用工具
│   ├── logger.sh       # log() / tail_log()
│   └── status.sh       # 输出 JSON 状态
│
└── monitor.sh          # 主业务循环（由 core/start.sh 拉起）
```

关键脚本模板

logger.sh

```sh
LOG_FILE="$MODDIR/.monitor.log"

log() {
    [ -z "$1" ] && return 0
    ts=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$ts] $1" >> "$LOG_FILE" 2>/dev/null
    # 日志轮转：超 400 行截 200 行
    if [ -f "$LOG_FILE" ]; then
        n=$(wc -l < "$LOG_FILE" 2>/dev/null)
        if [ "$n" -gt 400 ]; then
            tail -n 200 "$LOG_FILE" > "$LOG_FILE.tmp" 2>/dev/null \
                && mv "$LOG_FILE.tmp" "$LOG_FILE" 2>/dev/null
        fi
    fi
}
```

start.sh（幂等 + 心跳）

```sh
MODDIR="/data/local/tmp/Lin-Shizuku/module/<module_id>"
PID_FILE="$MODDIR/.pid"

read_pid() { cat "$1" 2>/dev/null | tr -d '[:space:]'; }

# 幂等：已有存活实例直接返回
if [ -f "$PID_FILE" ]; then
    old_pid=$(read_pid "$PID_FILE")
    if [ -n "$old_pid" ] && kill -0 "$old_pid" 2>/dev/null; then
        mtime=0
        command -v stat >/dev/null 2>&1 && mtime=$(stat -c %Y "$PID_FILE" 2>/dev/null || echo 0)
        if [ "$mtime" -le 0 ] 2>/dev/null; then
            echo "already-running"; exit 0
        fi
        now=$(date +%s 2>/dev/null || echo 0)
        if [ $((now - mtime)) -lt 120 ] 2>/dev/null; then
            echo "already-running"; exit 0
        fi
    fi
fi

# 冷启动清理
rm -f "$MODDIR/.cooldown" 2>/dev/null

# setsid 脱离进程组 + nohup 防挂断
nohup setsid sh "$MODDIR/scripts/monitor.sh" >/dev/null 2>&1 < /dev/null &

# 轮询等待 PID 写入（最多 5 秒）
i=0
while [ "$i" -lt 10 ]; do
    if [ -f "$PID_FILE" ]; then
        pid_new=$(read_pid "$PID_FILE")
        if [ -n "$pid_new" ] && kill -0 "$pid_new" 2>/dev/null; then
            echo "started $pid_new"; exit 0
        fi
    fi
    sleep 0.5; i=$((i + 1))
done
echo "start-failed"; exit 1
```

stop.sh（精确匹配防误杀）

```sh
MODDIR="/data/local/tmp/Lin-Shizuku/module/<module_id>"
PID_FILE="$MODDIR/.pid"

# 精确匹配 monitor 主进程（cmdline 以 /scripts/monitor.sh 结尾）
# 避免误杀执行本脚本的 sh -c 桥接进程
kill_monitors() {
    if command -v pgrep >/dev/null 2>&1; then
        for p in $(pgrep -f "/scripts/monitor\.sh$" 2>/dev/null); do
            kill "$p" 2>/dev/null
        done
    fi
}

[ -f "$PID_FILE" ] && kill "$(cat "$PID_FILE" 2>/dev/null | tr -d '[:space:]')" 2>/dev/null
kill_monitors
rm -f "$PID_FILE"
echo "stopped"
```

monitor.sh 主循环骨架

```sh
#!/system/bin/sh
MODDIR="/data/local/tmp/Lin-Shizuku/module/<module_id>"
PID_FILE="$MODDIR/.pid"
STATE_FILE="$MODDIR/.state"
COOLDOWN_FILE="$MODDIR/.cooldown"

. "$MODDIR/scripts/utils/logger.sh"

readconf() { grep "^$1=" "$MODDIR/config/module.conf" 2>/dev/null | cut -d= -f2-; }

setstate() {
    key=$1; val=$2
    touch "$STATE_FILE"
    if grep -q "^$key=" "$STATE_FILE" 2>/dev/null; then
        sed "s|^$key=.*|$key=$val|" "$STATE_FILE" > "$STATE_FILE.tmp" \
            && mv "$STATE_FILE.tmp" "$STATE_FILE"
    else
        echo "$key=$val" >> "$STATE_FILE"
    fi
}

main() {
    log "监控启动 (pid=$$)"
    setstate running 1

    while :; do
        touch "$PID_FILE"   # 心跳

        # enabled 检查
        enabled=$(readconf enabled)
        [ "$enabled" = "1" ] || {
            setstate running 0
            rm -f "$PID_FILE"
            exit 0
        }

        # 冷却检查
        if [ -f "$COOLDOWN_FILE" ]; then
            until_ts=$(cat "$COOLDOWN_FILE" | tr -d '[:space:]')
            now=$(date +%s)
            if [ "$until_ts" -gt "$now" ] 2>/dev/null; then
                setstate cooldown "$((until_ts - now))"
                sleep 1
                continue
            fi
            rm -f "$COOLDOWN_FILE"
            log "冷却结束"
        fi

        # === 业务逻辑 ===
        # ...

        sleep 1
    done
}

# 幂等防重
if [ -f "$PID_FILE" ]; then
    old_pid=$(cat "$PID_FILE" 2>/dev/null | tr -d '[:space:]')
    if [ -n "$old_pid" ] && kill -0 "$old_pid" 2>/dev/null; then
        mtime=$(stat -c %Y "$PID_FILE" 2>/dev/null || echo 0)
        if [ "$mtime" -le 0 ] 2>/dev/null \
           || [ $(( $(date +%s) - mtime )) -lt 120 ] 2>/dev/null; then
            exit 0
        fi
    fi
fi

# 原子写 PID
echo $$ > "$PID_FILE.tmp" && mv -f "$PID_FILE.tmp" "$PID_FILE"
trap 'rm -f "$PID_FILE"; exit 0' TERM INT
main
```

---

八、配置管理

双份配置设计

文件 格式 读者 说明
settings.json JSON WebUI 权威源，含结构化数组（如监控列表）
module.conf 扁平键值 monitor.sh 由 write.sh 从 JSON 生成，脚本读取快

write.sh 核心流程

```
base64 解码 JSON
    ↓
校验结构（含 watch 字段）
    ↓
原子写 settings.json（临时文件 + mv）
    ↓
用 grep -oE 严格提取字段 → 生成 module.conf
    ↓
若 monitor 正在运行 → restart.sh（不擅自拉起）
```

JSON 提取函数模板

```sh
json_num() {
    grep -oE "\"$1\"[[:space:]]*:[[:space:]]*[0-9]+" "$SETTINGS_FILE" \
        | head -1 | grep -oE '[0-9]+$'
}
json_str() {
    grep -oE "\"$1\"[[:space:]]*:[[:space:]]*\"[^\"]*\"" "$SETTINGS_FILE" \
        | head -1 | sed 's/^[^:]*:[[:space:]]*"//; s/"$//'
}
json_bool() {
    grep -qE "\"$1\"[[:space:]]*:[[:space:]]*true" "$SETTINGS_FILE" \
        && echo 1 || echo 0
}
```

避免 grep -o '"key":"value"' 这种脆弱写法，要兼容 "key": value、"key" : "value" 等空格变体。

---

九、安全与稳健性清单

风险 防护
命令注入 WebUI → 脚本用 base64 传 JSON；脚本内对包名做 case "$pkg" in *[!A-Za-z0-9._-]*) 校验
路径穿越 不接收路径参数；包名严格正则
重复启动 start.sh 用 kill -0 + 心跳新鲜度（120s）双重判定
PID 漂移 pgrep -f "/scripts/monitor\.sh$" 精确尾匹配，排除 sh -c 桥进程
误杀进程 am force-stop 为主，pgrep + /proc/$p/cmdline 精确匹配为辅
配置半写 临时文件 + mv 原子替换
日志膨胀 超 400 行截 200 行
base64 兼容 base64 -w0 失败降级 base64 \| tr -d '\n'
桥超时 sh() 封装加 20s 超时 + done 防重
WebUI 字段缺失 默认 state + 逐字段 != null ? : default 兜底
子 shell break 失效 IFS='\|' + for pkg in $watch，不用 \| while
环境差异 脚本用 #!/system/bin/sh，不依赖 bash 特性

---

十、开发流程建议

1. 先写 module.prop + module.json，确定 id、name、webui 入口
2. 搭骨架：run.sh 的 #on / #off 块先做「echo 测试」
3. 写核心业务脚本（如 monitor.sh），独立终端调试通过后再接 WebUI
4. 写 config/read.sh / write.sh，用 echo '...' | base64 -w0 手动测试
5. 写 utils/status.sh，输出 JSON 后用 python -m json.tool 校验
6. 最后写 WebUI，先做状态显示，再做配置编辑
7. 实测清单：
   · 模块页开关开 → start.sh 返回 started PID
   · 模块页开关关 → stop.sh 返回 stopped
   · WebUI 保存配置 → write.sh 返回 OK，module.conf 更新
   · WebUI 刷新状态 → status.sh 返回合法 JSON
   · 重复点开关 → PID 不变（幂等）
   · 卸载 → uninstall.sh 清理运行时文件

---

十一、调试技巧

```bash
# 语法自检
sh -n scripts/monitor.sh

# 手动跑核心脚本
sh scripts/core/start.sh
sh scripts/utils/status.sh
sh scripts/config/read.sh all_b64 | base64 -d

# 手动测 write.sh
echo '{"watch":[{"name":"X","pkg":"com.a.b"}],"kill_pkg":"com.xiaomi.joyose","threshold":20,"count":5,"interval":3,"cooldown":90,"drop_pct":35,"force_stop":true,"enabled":true}' | base64 -w0
# 把输出传给 write.sh

# 看日志
tail -f .monitor.log

# 看进程
pgrep -af "monitor.sh"

# 看桥报错（WebUI 侧）
# 在 sh() 回调里 console.log(out, err)，用 Chrome 远程调试
```

---

十二、WebUI 兼容性：Lin-Shizuku WebView 的已知限制与绕过

Lin-Shizuku 用系统 WebView 加载 webroot/index.html，比标准浏览器少了一些特性，也用不了一些"高级"CSS。以下都是实测踩过的坑，直接给绕过方案。

12.1 position: fixed + flex + vh 的组合不可靠

现象：把选择器做成"底部弹出覆盖层"时

```css
.overlay {
  position: fixed; inset: 0;
  display: flex; align-items: flex-end;
}
.sheet {
  max-height: 88vh;
  overflow-y: auto;
}
```

打开后只显示 sheet 顶部一小条，内容区全部塌陷到屏幕底部外面（内容其实存在，但高度算成 0）。

原因：Lin-Shizuku WebView 对 position:fixed 子项 + flex 的高度计算有 bug，max-height:88vh 在 flex 上下文中不生效。

绕过方案：不用覆盖层，改成卡片内联展开。

```css
.picker-inline {
  display: none;
  margin-top: 12px;
  padding: 12px;
  background: rgba(0,0,0,.25);
  border: 1px solid rgba(255,255,255,.12);
  border-radius: 16px;
}
.picker-inline.open { display: block; }

/* 滚动区用固定像素高度，不用 vh */
.app-list {
  height: 240px;              /* 不用 max-height:50vh */
  overflow-y: auto;
  -webkit-overflow-scrolling: touch;
}
```

触发：

```js
function togglePicker() {
  var el = document.getElementById("pickerInline");
  el.classList.toggle("open");
  if (el.classList.contains("open")) {
    setTimeout(function() {
      try { el.scrollIntoView({ behavior: "smooth", block: "start" }); } catch (e) {}
    }, 100);
  }
}
```

结论：

· 优先用文档流，不用覆盖层
· 必须滚动时用固定 px 高度，不用 vh / max-height:xxvh
· 需要"置顶/置底"效果时，用 scrollIntoView 把内联区块滚到可视区

12.2 vh 单位不可靠

100vh / 88vh / 50vh 在不同 WebView 版本上表现不一致（有的按 URL 栏可见高度算，有的按整个屏幕算）。一律改用 px 或 auto。

错误示范：

```css
.log { max-height: 30vh; }        /* 高度飘忽不定 */
.sheet { max-height: 88vh; }      /* 常塌陷为 0 */
```

正确：

```css
.log { max-height: 220px; }
.app-list { height: 240px; }
```

12.3 覆盖层内的滚动失效

position:fixed 元素里嵌 overflow-y:auto 子项，在 Lin WebView 上滚动条可能不出现，或手指滑动被外层吞掉。

绕过：同上，改用内联展开。

如果非要做覆盖层，给 .overlay 加：

```css
.overlay {
  position: fixed; inset: 0;
  z-index: 50;
  display: none;
  overflow-y: auto;            /* 让整个 overlay 可滚 */
  -webkit-overflow-scrolling: touch;
}
.overlay.open { display: block; }
.sheet {
  margin: 40px auto 0;         /* 顶部留白 */
  width: 92%;
  max-width: 520px;
  background: #151d33;
  border-radius: 20px;
  padding: 16px;
}
```

即：不要在固定高度容器里再套滚动容器，直接让整个 overlay 滚。

12.4 pm list packages 在桥里可能失败

现象：调用

```js
sh("pm list packages -3 2>&1 | sed 's/package://' | sort | head -500", cb);
```

返回空字符串或错误。

原因：

1. 桥的执行环境不继承父 shell 的 PATH，pm 可能找不到
2. 管道 |、重定向 2>&1 在部分 WebView 的 exec 实现里不完整支持
3. head / sed / sort 在 Lin 桥环境里不一定存在

绕过方案（三层保险）：

A. 命令简化，只保留最基本形式：

```js
sh("pm list packages -3", function(out, err) { ... });
```

用 JS 侧做解析，不用管道：

```js
txt.split("\n").forEach(function(line) {
  var p = line.replace(/^\s*package:/, "").trim();
  if (p && /^[A-Za-z0-9._-]+$/.test(p)) allApps.push({ pkg: p });
});
```

B. 失败可见，把 stderr 和桥错误展示给用户：

```js
if (!txt) {
  list.innerHTML = '<div class="empty">未返回应用列表' +
    (err ? '（' + esc(err) + '）' : '') +
    '<br>请使用「手动输入」或「常用游戏」添加</div>';
  return;
}
```

C. 兜底入口：永远提供

· 常用游戏快捷标签：预置 10–20 款主流游戏包名，点一下添加，不依赖 pm
· 手动输入包名：任何情况下都能加包

```js
var QUICK_GAMES = [
  { name: "王者荣耀", pkg: "com.tencent.tmgp.sgame" },
  { name: "和平精英", pkg: "com.tencent.tmgp.pubgmhd" },
  { name: "原神", pkg: "com.miHoYo.Yuanshen" },
  { name: "崩坏：星穹铁道", pkg: "com.miHoYo.hkrpg" },
  // ...
];
```

12.5 dumpsys package packages 输出过大

用 dumpsys package packages 提取应用名 label，会返回几百 KB 到几 MB，桥的 stdout 通道可能被截断或超时。

绕过：

```js
// 不用 head -c 300000（字节截断会切坏行）
// 用 head -n 8000（行截断，更安全）
sh('dumpsys package packages | grep -E "Package \\[|label=" | head -n 8000', function(out) {
  if (!out) return;  // 失败静默，UI 退回包名显示
  // ...
});
```

优先显示包名，label 作为异步增强。label 拿不到不影响功能。

12.6 浏览器 API 缺失

WebView 里不一定有：

· localStorage（部分受限）
· Intl（本地化）
· fetch / XMLHttpRequest（无网络时无用）

绕过：

· 持久化状态全部放脚本侧（settings.json），不用 localStorage
· 时间格式化手写 fmtDuration()，不用 Intl
· 所有数据交换通过 ksu.exec，不走网络

12.7 中文文件名 / 路径

WebUI 里 MODDIR 要硬编码：

```js
var MODDIR = "/data/local/tmp/Lin-Shizuku/module/joyse_jank_opt";
```

注意：

· 不要用 location.pathname 推导（WebView 里可能是 file:///android_asset/...）
· 不要用 document.currentScript.src（同上）
· 路径中如含中文，在 sh("...") 命令里可能被 shell 解析出问题，模块路径推荐全英文

12.8 桥回调不触发的排查

现象：sh() 调用后 cb 一直不执行，UI 卡在"加载中…"。

排查顺序：

1. 检查桥是否存在：console.log(typeof ksu, typeof ksu.exec)
2. 检查回调函数名：window[n] 是否真的被赋上
3. 看超时兜底是否工作：20s 后应该调 cb("", "«命令超时»")
4. 看桥的 error 参数：err 里可能有线索

推荐的调试封装：

```js
function sh(cmd, cb) {
  console.log("[sh] exec:", cmd);
  var n = "cb" + Date.now().toString(36) + Math.floor(Math.random() * 1e6).toString(36);
  var done = false;
  window[n] = function(errno, out, err) {
    console.log("[sh] cb:", n, "errno=", errno, "out=", out, "err=", err);
    if (done) return;
    done = true; clearTimeout(tmr); delete window[n];
    if (cb) cb(out, err);
  };
  var tmr = setTimeout(function() {
    console.log("[sh] timeout:", cmd);
    if (done) return;
    done = true; delete window[n];
    if (cb) cb("", "«命令超时»");
  }, 20000);
  try { ksu.exec(cmd, "{}", n); } catch (e) {
    console.log("[sh] throw:", e);
    if (!done) { done = true; clearTimeout(tmr); delete window[n]; if (cb) cb("", String(e)); }
  }
}
```

然后用 Chrome 远程调试（chrome://inspect）看 console 输出。

12.9 UI 兼容性检查清单

每次改 WebUI 后自测：

☐ 页面滚动流畅，无内容被遮挡
☐ 输入框聚焦时键盘不遮挡重要按钮
☐ 长列表滚动到底部能看到最后一项
☐ 弹层关闭后背景恢复正常（不用覆盖层的话忽略）
☐ 横竖屏切换（若允许）布局不错乱
☐ 深色模式（HyperOS 强制深色）下文字对比度足够
☐ 按钮最小点击区域 ≥ 40px（手指友好）
☐ 数字输入框弹出数字键盘（inputmode="numeric"）
☐ 刷新页面后状态能正确重建（读脚本侧配置）

12.10 推荐 UI 结构（实测稳定）

```
卡片（.card，文档流）
├── 状态卡（只读，5s 轮询）
├── 折线图卡（canvas，固定 120px 高）
├── 历史卡（只读，15s 轮询）
├── 监控列表卡
│   ├── 列表（.item）
│   └── ＋ 按钮 → 内联展开选择器（.picker-inline）
│       ├── 常用快捷标签
│       ├── 手动输入
│       ├── 搜索 + 应用列表（固定 240px 高，内部滚动）
│       └── 收起 / 添加按钮
├── 参数卡（表单）
├── 控制卡（按钮）
└── 日志卡（固定 220px 高，内部滚动）
```

核心原则：

1. 一切走文档流，不用覆盖层
2. 需要滚动的地方用固定 px 高度
3. 每张卡都有明确的顶部/底部，不依赖 flex 撑高
4. 所有异步操作都有超时兜底 + 可见错误提示
5. 任何依赖系统命令的功能，都准备一个不依赖它的手动兜底入口

---

十三、常见坑

坑 原因 解决
WebUI 一直转圈 桥回调没触发 加超时兜底 + 检查 ksu.exec 是否存在
PID 一直在变 pgrep -f monitor.sh 匹配到桥进程 用 monitor\.sh$ 精确尾匹配
保存配置无反应 base64 -w0 不支持 降级 base64 \| tr -d '\n'
配置读出来是 undefined JSON 缺字段 WebUI 端合并 DEFAULT_STATE
采样间隔不生效 多包串行 sleep 批量 reset → 统一 sleep → 逐个读取
强停没效果 Joyose 被系统自拉起 加验证日志，建议用户关 Joyose 自启
日志太大 无轮转 logger.sh 里加行数截断
开关关不掉 #off 块有 exit 0 移除块内 exit
脚本找不到 硬编码路径写错 所有脚本顶部 MODDIR="..." 用绝对路径
覆盖层不显示内容 WebView 对 fixed+flex 高度计算 bug 改内联展开，固定 px 高度
应用列表加载不出来 桥环境无 PATH / 管道受限 简化命令 + 常用快捷 + 手动输入
