Lin-Shizuku 模块开发指南

基于 Joyose掉帧自动优化_v1.0 实测结构整理。Lin 模块与 Magisk / KernelSU 模块在设计哲学上不同：它不刷系统分区、不依赖内核，而是通过 Shizuku 提供的 root 权限 执行脚本，用 WebUI 与用户交互。适合做「运行时进程监控 / 系统调优 / 服务管理」类模块。

---

一、Lin 模块 vs Magisk / KernelSU

维度 Magisk / KernelSU Lin-Shizuku
运行环境 内核级 / 系统分区 Shizuku（root 或 ADB 权限）
安装位置 /data/adb/modules/<id>/ /data/local/tmp/Lin-Shizuku/module/<id>/
系统修改 systemless overlayfs / magic mount 不支持，只做运行时操作
启停机制 post-fs-data.sh / service.sh 自动执行 模块页开关 → run.sh 的 #on / #off 块
开机自启 有 无（40 轮 APK 只自启 Shizuku）
安装钩子 customize.sh 自动执行 无（安装 = 解压 + module.json 声明）
卸载钩子 uninstall.sh 自动执行 无（uninstall.sh 是手动清理入口）
WebUI 需 KernelSU 的 kernelsu npm 包 经 Lin 桥 ksu.exec(cmd, opts, cb)
元数据 module.prop module.prop + module.json 双份

结论：Lin 模块本质是「Shizuku 权限下运行的脚本 + WebUI 控制台」，能力边界 = shell 能做什么。

---

二、目录结构

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

硬性约定：

· module.prop 和 module.json 都必须存在
· WebUI 必须在 webroot/index.html
· 所有脚本路径必须硬编码为绝对路径（Lin 桥执行时不继承 $PWD）

---

三、module.prop

```
id=joyse_jank_opt
name=Joyse掉帧自动优化
version=v1.0
versionCode=3
author=Lin
description=锁帧/大幅掉帧强停一次Joyse，冷却1分钟防负优化；可自定义游戏监控列表，WebUI控制台
```

规则：

· id 必须匹配 ^[a-zA-Z][a-zA-Z0-9._-]+$（不能以数字、短横线开头）
· versionCode 必须是整数
· 换行必须用 LF，不要 CRLF
· 不要有 BOM

---

四、module.json

```json
{
  "id": "joyse_jank_opt",
  "name": "Joyse掉帧自动优化",
  "version": "v1.0",
  "versionCode": 3,
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

1. 桥注入

Lin 管理器把 JS 桥注入到 WebView 的 window.ksu：

```js
ksu.exec(cmd, opts, callbackName)
```

实际签名（与 KernelSU 官方 kernelsu npm 包不同）：

· cmd：shell 命令字符串
· opts："{}" 或选项字符串（Lin 版本传空对象）
· callbackName：回调函数名，Lin 通过 window[callbackName](errno, stdout, stderr) 回调

2. 通用封装（推荐直接抄）

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

3. 与脚本交互的三种模式

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

4. 状态字段约定

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
子 shell break 失效 `IFS='
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

十二、常见坑

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

