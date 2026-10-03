# Lin-Shizuku 模块开发指南（官方版）

> 本文档是 Lin-Shizuku 宿主的**模块开发规范**。基于宿主自身实现整理，适用于所有在 Lin-Shizuku 上运行的模块（服务管理 / 系统调优 / 进程监控 / 工具类）。
>
> Lin 模块与 Magisk / KernelSU 模块本质不同：**不刷系统分区、不依赖内核**，而是通过 Shizuku 提供的 root/ADB 权限执行脚本，用宿主内置 WebView 加载 WebUI 与用户交互。**模块的能力边界 = shell 能做什么**。

---

## 一、模块模型

### 1.1 宿主与模块的分工

```
┌─────────────────────────── Lin-Shizuku 宿主 ───────────────────────────┐
│  Shizuku 服务连接与提权        模块安装 / 卸载 / 启停                  │
│  模块 WebUI 加载（WebView）     ksu 桥（shell 命令执行）               │
│  兼容注入层（Promise/探测/布局）  模块列表与开关页                       │
└────────────────────────────────────────────────────────────────────────┘
                                   │  ksu.exec(cmd, opts, cb)
                                   ▼
/data/local/tmp/Lin-Shizuku/module/<module_id>/
├── module.json          声明（权威，含 webui 入口）
├── run.sh               开关入口（#on / #off 块）
├── webroot/index.html   WebUI 入口
└── scripts/…            业务脚本（宿主只负责执行，不关心内部实现）
```

**结论**：宿主只做三件事——**安装目录管理、开关执行 run.sh、加载 WebUI 并执行桥命令**。模块的启动/停止/业务逻辑全部由模块自己的脚本负责。

### 1.2 模块生命周期

```
安装（解压 zip → 校验 module.json → 放入模块目录）
   ↓
模块页开关 [开] → 抽取 run.sh #on 块 → 模块启动脚本（start）
   ↓
运行中：WebUI 轮询 status.sh 展示状态 / 用户改配置 → write.sh
   ↓
模块页开关 [关] → 抽取 run.sh #off 块 → 模块停止脚本（stop）
   ↓
卸载（删除模块目录，运行时文件一并清除）
```

关键约定：

- **开关是唯一启停来源**，WebUI 不提供启停按钮（避免绕过宿主生命周期）
- 宿主不会"开机自启模块"——开机只自启 Shizuku 服务；模块如需开机恢复，由模块自身在启动脚本中实现
- 安装/卸载是纯文件操作，**无自动执行钩子**；模块自己的 `uninstall.sh` 是手动清理入口

---

## 二、目录规范

### 2.1 标准目录模板

```
/data/local/tmp/Lin-Shizuku/module/<module_id>/
├── module.json              # 模块声明（必需）
├── module.prop              # Magisk/KernelSU 兼容元数据（推荐）
├── run.sh                   # 开关入口（必需）
├── uninstall.sh             # 可选：手动清理入口
├── README.md                # 可选
│
├── config/
│   ├── settings.json        # WebUI 权威配置（JSON，结构化）
│   └── module.conf          # 运行时扁平键值（脚本读取快，由写配置时派生）
│
├── scripts/
│   ├── core/                # start.sh / stop.sh / restart.sh
│   ├── config/              # read.sh / write.sh
│   ├── utils/               # logger.sh / status.sh
│   └── <main>.sh            # 主业务逻辑（由 core/start.sh 拉起）
│
├── webroot/                 # WebUI（提供 WebUI 时必需）
│   └── index.html           # 入口，宿主按 module.json 的 webui 字段加载
│
└── .runtime/                # 运行时文件（脚本自建，不入包）
    ├── .pid                 # 进程 PID
    ├── .state               # 状态键值
    ├── .log                 # 运行日志
    └── .cooldown            # 冷却时间戳（按需）
```

### 2.2 硬性约定

1. `module.json` 必须存在，`webui` 字段指向 WebUI 入口
2. `run.sh` 必须存在，且按规范提供 `#on` / `#off` 块
3. WebUI 入口必须是 `webroot/index.html`（或其下相对路径）
4. **所有脚本内部路径硬编码为绝对路径**——宿主桥执行命令时不继承 `$PWD`，相对路径必然出错
5. 模块目录名 = `module_id`，命名规则见元数据章节

---

## 三、元数据

### 3.1 module.json（权威声明）

```json
{
  "id": "my_module",
  "name": "模块显示名称",
  "version": "v1.0",
  "versionCode": 1,
  "author": "作者",
  "description": "一句话说明能力",
  "webui": "webroot/index.html"
}
```

字段规则：

| 字段 | 必填 | 规则 |
|---|---|---|
| `id` | ✅ | `^[a-zA-Z][a-zA-Z0-9._-]+$`，不能以数字或 `-` 开头；全小写推荐 |
| `name` | ✅ | 模块列表显示名，可中文 |
| `version` | ✅ | 展示用版本串 |
| `versionCode` | ✅ | 整数，用于版本比较 |
| `author` | ✅ | 作者标识 |
| `description` | ✅ | 模块列表副标题 |
| `webui` | ✅* | WebUI 入口相对路径（不提供 WebUI 时省略） |

### 3.2 module.prop（兼容层）

```ini
id=my_module
name=模块显示名称
version=v1.0
versionCode=1
author=作者
description=一句话说明能力
```

- 与 module.json 字段冗余，兼容 Magisk/KernelSU 工具链的读取习惯
- 宿主读取时 **module.json 优先**，缺失再回退 module.prop
- 文件编码 UTF-8、LF 换行、无 BOM

---

## 四、启停机制：run.sh

模块页开关切换时，宿主从 run.sh 中**抽取并单独执行** `#on` / `#off` 块。

### 4.1 标准模板

```sh
#!/system/bin/sh
# Lin-Shizuku 模块开关入口
# 模块页开 → 执行 #on 块；关 → 执行 #off 块

# ---- 手动执行入口（终端 sh run.sh 默认走这里） ----
MODDIR="/data/local/tmp/Lin-Shizuku/module/<module_id>"
sh "$MODDIR/scripts/core/start.sh"
exit 0

#on
MODDIR="/data/local/tmp/Lin-Shizuku/module/<module_id>"
sh "$MODDIR/scripts/core/start.sh"

#off
MODDIR="/data/local/tmp/Lin-Shizuku/module/<module_id>"
sh "$MODDIR/scripts/core/stop.sh"
```

### 4.2 规则

1. `#on` / `#off` 块会被**单独抽取执行**，块内必须自带 `MODDIR`，**不得依赖块外变量**
2. 块内**不要写 `exit`**——会提前终止宿主抽取的脚本
3. 手动入口放在文件最前 + `exit 0`，避免手动执行误入 `#on` 块
4. `#on`/`#off` 块各自只做一件事：调 `core/start.sh` / `core/stop.sh`，业务逻辑全部下沉到 scripts

---

## 五、WebUI 与 ksu 桥协议

### 5.1 桥对象

宿主向 WebView 注入 `window.ksu`：

```js
ksu.exec(cmd, opts, callbackName)   // 异步执行 shell 命令
ksu.execSync(cmd)                   // 同步执行，返回 stdout 字符串
ksu.moduleInfo()                    // { moduleDir, moduleId, name, version }
ksu.moduleEnabled()                 // 模块是否启用
ksu.toast(msg)                      // 设备弹提示
```

`ksu.exec` 回调约定：

```js
window[callbackName](errno, stdout, stderr)
```

- `errno`：真实进程退出码（0 成功；非 0 失败；-1 超时）
- `stdout` / `stderr`：分离输出
- 宿主执行走 `sh -c` 并注入 PATH，**管道/重定向可用**

### 5.2 宿主注入的增强 API

宿主在页面加载完成后自动注入，模块可直接使用：

**`ksu.execPromise(cmd, opts)` → Promise**

```js
const { errno, stdout, stderr } = await ksu.execPromise("cat /proc/version");
if (errno !== 0) { /* 处理失败 */ }
```

**`window.__lin_webview_info`（兼容探测）**

```js
{
  webviewVersion: "120.0.6099.230",   // WebView 内核 Chrome 版本
  fixedSupported: true,               // position:fixed 是否可靠
  vhSupported: true,                  // vh 单位是否可靠
  innerHeight: 891,
  devicePixelRatio: 2.75
}
```

模块可据此条件降级（见第六章）。

### 5.3 通用 sh() 封装模板（推荐直接抄）

```js
var MODDIR = "/data/local/tmp/Lin-Shizuku/module/<module_id>";

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

要点：**20s 超时兜底**（桥异常时回调永不返回）+ **done 防重**（桥可能多次触发）+ **delete window[n]**（防泄漏）。

### 5.4 数据交换模式

| 场景 | 方式 | 说明 |
|---|---|---|
| 读配置 | `read.sh all_b64` → base64 解码 JSON | 防特殊字符破坏 |
| 写配置 | `write.sh set_b64 '<base64 JSON>'` | 防 shell 注入 |
| 读状态 | `status.sh` → 标准 JSON | WebUI 每 5s 轮询 |
| 读日志 | `tail -n 30 <模块>/.log` | 追加展示 |

`status.sh` 推荐输出：

```json
{
  "running": true,
  "pid": "1234",
  "lastEvent": "最近一次动作描述",
  "eventTime": "2026-10-01 12:34:56",
  "logSize": 12345,
  "cooling": false,
  "cooldownRemain": 0
}
```

---

## 六、WebView 兼容层（宿主已内置）

宿主 WebView 已做以下处理，**模块作者无需重复实现**：

| 项目 | 宿主配置 |
|---|---|
| Viewport | `useWideViewPort=true` + `loadWithOverviewMode=false` + `setInitialScale(0)`，遵守页面 viewport meta |
| Mixed content | ALWAYS_ALLOW（file:// 页面可加载资源） |
| 文件访问 | allowFileAccess + 跨源 file:// 访问开启 |
| 硬件加速 | 应用级开启，未用软件渲染 |
| 深色模式 | 不强制覆盖，尊重页面 `prefers-color-scheme` |
| vh 兜底 | 注入 `JS_VH_POLYFILL`：遍历元素把 `height/min-height/max-height/top/bottom` 中的 vh 换算为真实 px（节流 + MutationObserver） |
| 布局兜底 | 注入 `JS_LAYOUT_FIX`：仅对「body flex 列 + flex:1 内部滚动容器」页面强制 html/body 高度 = innerHeight 并修正滚动区高度；**其他页面完全不动** |
| 注入顺序 | execPromise → 兼容探测 → vh polyfill → 布局修复 |

### 6.1 模块作者应遵守的兼容原则

1. **滚动高度用固定 px**，不要用 `vh` / `max-height:xxvh`（WebView 各版本 vh 计算不一致）
2. **优先文档流布局**，避免 `position:fixed` + flex + vh 组合的"底部覆盖层"（高度计算 bug，内容会塌陷为 0）
3. 必须用覆盖层时：让**整个 overlay 滚动**（`position:fixed; inset:0; overflow-y:auto`），不要在固定高度容器里再嵌滚动容器
4. 需要"置顶/置底"时用 `scrollIntoView()` 滚动内联区块，而不是覆盖层
5. 持久化走脚本侧 `settings.json`，**不要依赖 localStorage**
6. 时间/数字格式化手写，不依赖 `Intl`；数据交换走 `ksu.exec`，不走网络
7. 可选：读 `window.__lin_webview_info` 做条件分支（如 `fixedSupported=false` 时自动改内联展开）

---

## 七、脚本分层与核心模板

### 7.1 分层

```
scripts/
├── core/       生命周期：start（幂等）/ stop（精确匹配）/ restart
├── config/     配置读写：read（get/all/all_b64）/ write（set_b64）
├── utils/      通用：logger（轮转日志）/ status（JSON 状态）
└── <main>.sh   主业务循环（由 core/start.sh 以 nohup setsid 拉起）
```

### 7.2 start.sh（幂等 + 心跳）

```sh
MODDIR="/data/local/tmp/Lin-Shizuku/module/<module_id>"
PID_FILE="$MODDIR/.runtime/.pid"

read_pid() { cat "$1" 2>/dev/null | tr -d '[:space:]'; }

# 幂等：已有存活实例直接返回（kill -0 + 心跳新鲜度 120s 双重判定）
if [ -f "$PID_FILE" ]; then
    old_pid=$(read_pid "$PID_FILE")
    if [ -n "$old_pid" ] && kill -0 "$old_pid" 2>/dev/null; then
        mtime=$(stat -c %Y "$PID_FILE" 2>/dev/null || echo 0)
        now=$(date +%s 2>/dev/null || echo 0)
        if [ "$mtime" -le 0 ] 2>/dev/null || [ $((now - mtime)) -lt 120 ] 2>/dev/null; then
            echo "already-running"; exit 0
        fi
    fi
fi

# 清理冷启动残留 → setsid 脱离进程组 + nohup 防挂断 → 拉起主逻辑
rm -f "$MODDIR/.runtime/.cooldown" 2>/dev/null
nohup setsid sh "$MODDIR/scripts/<main>.sh" >/dev/null 2>&1 < /dev/null &

# 轮询等待 PID 落盘（最多 5s）
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

### 7.3 stop.sh（精确匹配防误杀）

```sh
MODDIR="/data/local/tmp/Lin-Shizuku/module/<module_id>"
PID_FILE="$MODDIR/.runtime/.pid"

# 精确匹配主进程（cmdline 以 /scripts/<main>.sh 结尾），排除 sh -c 桥接进程
kill_main() {
    if command -v pgrep >/dev/null 2>&1; then
        for p in $(pgrep -f "/scripts/<main>\.sh$" 2>/dev/null); do
            kill "$p" 2>/dev/null
        done
    fi
}

[ -f "$PID_FILE" ] && kill "$(read_pid "$PID_FILE")" 2>/dev/null
kill_main
rm -f "$PID_FILE"
echo "stopped"
```

### 7.4 logger.sh（轮转）

```sh
LOG_FILE="$MODDIR/.runtime/.log"

log() {
    [ -z "$1" ] && return 0
    ts=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$ts] $1" >> "$LOG_FILE" 2>/dev/null
    n=$(wc -l < "$LOG_FILE" 2>/dev/null)
    if [ "$n" -gt 400 ]; then
        tail -n 200 "$LOG_FILE" > "$LOG_FILE.tmp" 2>/dev/null \
            && mv "$LOG_FILE.tmp" "$LOG_FILE" 2>/dev/null
    fi
}
```

### 7.5 主循环骨架

```sh
#!/system/bin/sh
MODDIR="/data/local/tmp/Lin-Shizuku/module/<module_id>"
PID_FILE="$MODDIR/.runtime/.pid"
STATE_FILE="$MODDIR/.runtime/.state"

. "$MODDIR/scripts/utils/logger.sh"

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
    log "启动 (pid=$$)"
    setstate running 1
    while :; do
        touch "$PID_FILE"            # 心跳
        # ... 业务逻辑 ...
        sleep 1
    done
}

# 幂等防重（同 start.sh 判定）
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

# 原子写 PID + 信号清理
echo $$ > "$PID_FILE.tmp" && mv -f "$PID_FILE.tmp" "$PID_FILE"
trap 'rm -f "$PID_FILE"; exit 0' TERM INT
main
```

---

## 八、配置管理

### 8.1 双份配置设计

| 文件 | 格式 | 读者 | 说明 |
|---|---|---|---|
| `config/settings.json` | JSON | WebUI | **权威源**，含结构化数组（如监控列表） |
| `config/module.conf` | 扁平键值 | 主脚本 | write.sh 从 JSON 派生，脚本读取快 |

### 8.2 write.sh 流程

```
base64 解码 JSON
    ↓
结构校验（关键字段存在性）
    ↓
原子写 settings.json（临时文件 + mv）
    ↓
grep -oE 严格提取字段 → 派生 module.conf
    ↓
若主进程运行中 → 调用 restart.sh（不擅自拉起、不重复启动）
```

### 8.3 JSON 提取模板（兼容空格变体）

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

---

## 九、安全与稳健性清单

| 风险 | 防护 |
|---|---|
| 命令注入 | WebUI → 脚本用 base64 传 JSON；脚本内对用户输入（如包名）做白名单校验 |
| 路径穿越 | 不接收路径参数；标识符严格正则 |
| 重复启动 | `kill -0` + 心跳新鲜度（120s）双重判定 |
| PID 漂移 | `pgrep -f "/scripts/<main>\.sh$"` 精确尾匹配，排除 sh -c 桥进程 |
| 误杀进程 | 精确匹配 cmdline 后再 kill |
| 配置半写 | 临时文件 + mv 原子替换 |
| 日志膨胀 | 超 400 行截 200 行 |
| base64 兼容 | `base64 -w0` 失败降级 `base64 \| tr -d '\n'` |
| 桥超时 | `sh()` 封装 20s 超时 + done 防重 |
| WebUI 字段缺失 | 合并 DEFAULT_STATE + 逐字段兜底 |
| 子 shell break 失效 | `IFS='\|'` + `for x in $list`，不用 `\| while` |
| 环境差异 | `#!/system/bin/sh`，不依赖 bash 特性 |

---

## 十、开发与调试

### 10.1 开发顺序

1. 定 `module.json`（id/name/webui）
2. 搭骨架：`run.sh` 的 `#on`/`#off` 块先做 echo 验证
3. 写核心脚本，终端独立调试通过后接 WebUI
4. 写配置读写脚本，`echo '...' | base64 -w0` 手动验证
5. 写 `status.sh`，用 `python -m json.tool` 校验输出
6. 最后写 WebUI：先状态显示，后配置编辑

### 10.2 调试命令

```bash
sh -n scripts/<main>.sh                                    # 语法自检
sh scripts/core/start.sh                                   # 手动启动
sh scripts/utils/status.sh                                 # 状态 JSON
sh scripts/config/read.sh all_b64 | base64 -d              # 读配置
echo '{"key":"value"}' | base64 -w0                        # 手动造 payload
tail -f .runtime/.log                                      # 看日志
pgrep -af "scripts/<main>.sh"                              # 看进程
```

WebUI 桥调试：`sh()` 封装里 `console.log` 桥出入参，用 `chrome://inspect` 远程调试 WebView。

### 10.3 实测清单

- [ ] 模块页开关开 → start 返回 `started <PID>`
- [ ] 模块页开关关 → stop 返回 `stopped`
- [ ] 重复点开关 → PID 不变（幂等）
- [ ] WebUI 保存配置 → write 返回 OK，module.conf 更新
- [ ] WebUI 刷新状态 → status.sh 返回合法 JSON
- [ ] 长列表滚动到底、键盘不遮挡按钮、深色模式对比度正常
- [ ] 卸载 → uninstall.sh 清理运行时文件

---

## 十一、发布与分发

### 11.1 打包

模块以 **zip** 分发（含 webroot 与 scripts，不含 `.runtime/`）：

```bash
cd <模块根目录>
zip -r <module_id>.zip . -x '.runtime/*'
```

### 11.2 安装 / 更新 / 卸载

- **安装**：宿主模块页 → 安装模块 → 选 zip。同 id 覆盖安装（保留 config，清理 .runtime）
- **更新**：直接覆盖安装新版 zip，配置保留
- **卸载**：模块页卸载 → 删除目录（先手动执行 uninstall.sh 清理进程与运行时文件）

### 11.3 版本

- 升级改 `versionCode` + `version`；覆盖安装以 versionCode 判断新旧
- 保持 id 不变才可覆盖安装；改 id = 新模块

---

## 十二、常见问题

| 问题 | 原因 | 解决 |
|---|---|---|
| WebUI 一直转圈 | 桥回调没触发 | 检查 `typeof ksu.exec`；sh() 加超时兜底 |
| PID 一直在变 | pgrep 匹配到桥进程 | `monitor\.sh$` 精确尾匹配 |
| 保存配置无反应 | `base64 -w0` 不支持 | 降级 `base64 \| tr -d '\n'` |
| 配置读出来 undefined | JSON 缺字段 | WebUI 合并 DEFAULT_STATE |
| 开关关不掉 | `#off` 块含 `exit` | 移除块内 exit |
| 脚本找不到 | 相对路径 / PWD 未继承 | 全部硬编码绝对路径 |
| 覆盖层内容塌陷 | fixed+flex+vh 高度计算 bug | 内联展开 + 固定 px 高度 |
| 应用列表加载不出来 | 桥环境命令受限 | 简化命令 + 快捷标签 + 手动输入兜底 |
| 命令输出被截断 | stdout 通道超限 | 用行截断（head -n），不用字节截断 |
