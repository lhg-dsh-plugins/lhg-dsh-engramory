# 本版修复（BUGFIX）

## Bug：dsh 守卫对目标文件无界同步读取——FIFO 挂死宿主、/dev/zero 撑爆内存（上游 issue #21）

上游 issue：https://github.com/tinqiao-oss/engramory/issues/21（open，报告基于同一提交 500cf5d / 0.2.5）

`adapters/dsh/plugin/index.js` 的守卫（`ctx.tools.guard()`，同步运行在宿主主线程上）
在判断一次 write/edit 是否会突破索引上限时，用 `readIfPossible()` 对目标文件直接
`readFileSync(filePath)`，无 `stat`、无「是否普通文件」检查、无大小上限。未配置
`indexPath` 时，任意目录下任意名为 `MEMORY.md` 的文件被写入/编辑都会触发这次读取：

- `MEMORY.md` 是 FIFO → `readFileSync` 永远阻塞，宿主事件循环挂死；
- `MEMORY.md` 符号链接到 `/dev/zero`（git 会原样检出的链接）→ 无限读入，
  进程 RSS 涨破 1 GB 被杀；
- 普通大文件（如 64 MB）→ 每次编辑都全量读入内存，只为和 25,600 字节的上限比较。

## 证据链

复现脚本（工作区 `repro.sh` + `repro_child.mjs`：以 test.js 同款 Proxy ctx 挂载守卫，
子进程带 timeout / ulimit 看门狗，对同一路径分别跑 write/edit）：

修复前（上游 0.2.5 @ 500cf5d）：

```
MEMORY.md -> /dev/zero (write) | OOM 中止，exit 134（堆内存耗尽，ulimit -v 2GB 内）
MEMORY.md 为 FIFO (write)      | 6s timeout，exit 124（守卫永不返回）
普通小文件 MEMORY.md (write)   | RETURNED 0ms，allow（对照组正常）
64MB 普通文件 (edit)           | RETURNED 285ms（全量读入 64MB 后才返回）
```

修复后（0.2.6）：

```
MEMORY.md -> /dev/zero (write) | RETURNED 1ms，allow（按不可读处理，fail-open）
MEMORY.md 为 FIFO (write)      | RETURNED 0ms，allow
普通小文件 MEMORY.md (write)   | RETURNED 1ms，allow（行为不变）
64MB 普通文件 (edit)           | RETURNED 0ms，拒绝「already over the cap (67108864 bytes > 25600)…」
64MB 文件上写小内容 (write)    | RETURNED 0ms，allow（缩小重写仍放行，压实路径不堵死）
```

新增的 3 条回归测试（`adapters/dsh/plugin/test.js`）经 git stash 对照：
未修复代码跑同一套件 39.8s 仍未完成（FIFO 挂死被 timeout 杀），修复后 0.32s 全过。

## 根因

`readIfPossible()` 把「读不到就当空文件」这一 fail-open 语义建立在「`readFileSync`
要么成功要么抛错」的假设上；但对非普通文件它两者都不是——FIFO 阻塞、设备文件
无限量供数——且对普通文件读取量与守卫真正需要的信息（是否已超上限、当前字节数、
前缀内的行数）完全不成比例。

## 改动

仅 `adapters/dsh/plugin/index.js`（核心修复），另加配套测试/证据：

- `readIfPossible()` → `readCurrent(filePath, settings)`：先 `statSync`；
  非普通文件（FIFO、设备、套接字等）直接按不可读处理（返回 undefined，守卫
  fail-open，与缺失文件语义一致）；普通文件不超过 `maxBytes + 1` 时整读，
  超过时只用 `openSync/readSync` 读前 `maxBytes + 1` 字节，返回
  `{ buf, size, truncated }`，其中 `size` 是 stat 的真实字节数。
- `verdict()`：当前字节数改取 stat 的 `size`（截断读不会再把大文件的缩小重写
  误判成增长）；行数取已读缓冲的计数（截断时为前缀内的下限）。
- 局部编辑路径：截断读无法模拟编辑结果，文件此时必然已超字节上限，走原有
  「已超限 + 无法测量 → 拒绝盲改并提示整文件压实」的规则，拒绝理由报真实大小。
- `adapters/dsh/plugin/test.js`：新增 3 条回归测试（非普通文件不读入、FIFO 不阻塞、
  超大文件按 stat 尺寸判定且缩小重写放行）。
- `tests/dsh_e2e/results.json`：插件代码变更后，上游自有规则要求兼容声明须有
  「当前代码 + 当前版本」的真实 dsh e2e 通过记录（`tests/test_dsh_plugin.py` 钉死），
  故用 `tests/dsh_e2e/run.py --record` 在真实 dsh 0.1.5-rc.3 / 0.1.7-alpha.1 /
  0.1.7-alpha.2 / 0.1.7-rc.1 上重跑并全部 PASS（新 runtime digest
  sha256:047b27a7…）。
- `package.json` 与 `adapters/dsh/plugin/package.json`（两处被上游同步测试钉死）：
  version 0.2.5 → 0.2.6，repository.url 改为本 fork 仓库。其余字段不动。

## 复验

| 检查 | 修复前基线 | 修复后 |
|---|---|---|
| 复现脚本（/dev/zero、FIFO、小文件、64MB） | OOM exit 134 / 挂死 exit 124 / 正常 / 285ms 全量读 | 1ms 返回 / 0ms 返回 / 正常 / 0ms 按真实尺寸判定 |
| `node --test adapters/dsh/plugin/test.js` | 28/28 通过（新测试加入后未修复代码挂死） | 31/31 通过 |
| `python3 tests/test_dsh_plugin.py` | ALL PASS（0.2.5 原始代码） | ALL PASS（0.2.6，含新录制的 4 条 e2e 证据校验） |
| 其余 Python 套件（index_guard/state_lock/codex_hooks/codex_init/uninstall/protocol_sync/trace） | 全部通过 | 全部通过 |
| `python3 tests/test_tools.py` | 1 FAILED | 1 FAILED（与基线同一条，见下） |

基线既有失败（修复前后相同、与本修复无关）：`test_tools.py` 的
`test_doctor_reports_unenumerable_directory`——该用例靠 chmod 000 制造不可枚举
目录，但本机测试以 root 运行，root 不受权限位限制，目录仍可枚举，故断言失败。
在非 root 环境（上游 CI）该用例通过。

## 边界

- 截断读时行数是「前 maxBytes+1 字节内行数」的下限：一个已超字节上限、且单行
  极长的文件，若新写内容行数超过该下限但未超过文件真实总行数，行数维度的判定
  可能偏严（拒绝）；字节维度判定始终精确（stat 尺寸），且整文件缩小重写永远
  放行，压实脱困路径不受影响。这是按 issue #21 建议（大文件只读 maxBytes+1）
  取的内存/阻塞上界，代价仅限于这一极端组合。
- 本修复只动 dsh 插件（JS）；Python 守卫 `hooks/engramory_index_guard.py` 不在
  issue #21 报告范围内，未改动。
- 上游 issue #20（符号链接/相对路径下 indexPath 身份判定）是独立问题，本版未处理。
