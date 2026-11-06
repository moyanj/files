# xDroid 在 ArchLinux 上的安装方法与内核模块适配指南

> 环境：ArchLinux，自编译内核 linux-moyan 7.1.9，xDroid 13.2.380（麟卓）。
> 本文档记录完整安装方法，含全部脚本，可独立复现。
>
> 前置说明：
> - 本文脚本均为 Python 标准库实现（ctypes + os + shutil），无第三方依赖，
>   因此不建 uv venv，直接用系统 python3 以 root 运行（需要操作 /usr/src、
>   /var/lib/dkms 等 root 路径）。这是对「Python 必须用 uv」规范的有意偏离。
> - 涉及的内核模块、udev 规则、systemd 单元均已在本机实测通过。

---

## 0. 结论速览

| 模块 | 是否需要动手 | 最终方案 |
|---|---|---|
| `ashmem_xdroid` | **需要** | 运行时捕获厂商源码 → 修补 API 变更 → 编译安装 |
| `binder_xdroid` | 不需要 | 内核内置 `rust_binder` + binderfs 动态分配设备节点 |
| `asg_xdroid` | 不需要 | goldfish 模拟器地址空间模块，容器运行时不使用 |

整个适配只有 **一个 7 行补丁**（ashmem.c），外加两个捕获/注入脚本。

---

## 1. 问题分析

### 1.1 xDroid 是什么

xDroid 不是虚拟机，而是「Android 13 用户态 + LXC 容器」：

```
Linux 宿主机（ArchLinux）
└── LXC 容器 = Android 13 用户态（squashfs 镜像 android13.img，1.75 GB）
    ├── /system  /vendor  /apex
    └── Android 应用 = 普通 Linux 进程，窗口由 xDroidUI 托管
         ↑ 依赖内核提供
         binder(IPC)  ashmem(共享内存)  asg(模拟器地址空间)
```

容器内运行 `surfaceflinger`、`system_server`、`webview_zygote` 等真实
Android 系统进程，说明这是完整的 Android 运行时，不是模拟器。

### 1.2 为什么在 ArchLinux 上装不上

官方安装器按顺序做三件事：

1. 解包用户态文件到 `/opt/xdroid/`（这部分能成功）
2. **动态从服务器下载内核模块源码**到 `/usr/src/`，用 DKMS 编译
   `ashmem_xdroid` / `binder_xdroid` / `asg_xdroid` 三个模块
3. 编译成功才继续部署 systemd 单元、配置文件等

第 2 步在 6.1+ 内核上必然失败：

```
ashmem.c:413:64: error: 'shmem_file_setup' 的第 3 个实参类型不兼容
  413 |   vmfile = shmem_file_setup(name, asma->size, vma->vm_flags);
      | 传入 vm_flags_t (unsigned long)，期望 vma_flags_t
```

Linux 6.1 把 VMA 标志从 `vm_flags_t` 位图重构为 `vma_flags_t` 结构体。
失败后安装器中断，后续文件不再部署，**且源码立即被清理**。

### 1.3 核心困难：源码不在安装包里

安装包（962 MiB 的自解压脚本 + gzip tar）里没有任何 `.c` 文件，
`android13.img`（squashfs）里也没有。源码由安装器运行时从服务器拉取，
解压到 `/usr/src/`，存活仅约 30 秒（12:05:01 落地，12:05:32 被删）。

因此必须「边安装边捕获」。

---

## 2. 方法总览

```
准备阶段                     安装阶段（约 30~60 秒窗口）
─────────────────           ──────────────────────────────
capture_ko.py     ───────┐
(inotify 监视源码落地)     │
                         ├──→ 运行官方安装器 ./xDroidInstall-*.run
inject_ashmem.py  ───────┘         │
(把解包的 ashmem.c                  ├─ 解包用户态 → /opt/xdroid
 替换为已修补版本,                  ├─ 释放 ashmem 源码 → /usr/src/ashmem/
 让它编译真实成功)                  │    ├─ capture: 秒级备份原版
                                   │    └─ inject:  覆盖为修补版 → 编译成功
                                   ├─ 释放 asg 源码  → capture 抓下
                                   ├─ binder → 走 binderfs，无需源码
                                   └─ 部署 systemd / 配置 → 安装完成
```

- **capture_ko.py**：inotify 递归监视 `/usr/src` 与 `/var/lib/dkms`，
  源码一落地立刻复制走，附带内容去重和每 2 秒定时重扫兜底。
- **inject_ashmem.py**：安装器在 ashmem 失败后会中断，不再解包后续模块。
  本脚本把解包出的 ashmem 源码替换为已修补版，让编译**真实成功**，
  安装器于是继续推进 asg、binder 阶段，给 capture 创造捕获机会。
  必须覆盖三处路径防竞态（解包处、dkms source 处、dkms build 处）。

> 为什么不用「拦截 /usr/bin/dkms」：安装器会自行还原被替换的 dkms，
> 包装器不可靠。改在文件系统层监听，绕开与安装器的对抗。

---

## 3. 脚本一：源码捕获 capture_ko.py

保存为 `capture_ko.py`，`sudo python3 capture_ko.py 3600` 运行。

```python
#!/usr/bin/env python3
"""监视 xDroid 安装过程中释放的内核模块源码并立即备份。

背景:
  xDroid 安装器运行时把 ashmem/asg/binder 的 C 源码释放到
  /usr/src/<name>/ 交给 DKMS 编译, 失败后约 30 秒内清理。
  本脚本用 inotify 递归监听 /usr/src 与 /var/lib/dkms,
  任何 .c/.h/Makefile 出现即复制到快照目录, 在清理前抢下源码。

用法:
  sudo python3 capture_ko.py [持续秒数, 默认 3600]
"""

from __future__ import annotations

import ctypes
import ctypes.util
import os
import select
import shutil
import signal
import struct
import sys
import time
from pathlib import Path

WATCH_ROOTS: list[Path] = [Path("/usr/src"), Path("/var/lib/dkms")]
SNAPSHOT_ROOT = Path.home() / "apps" / "xdroid" / "_ko_snapshot"
DURATION_S = int(sys.argv[1]) if len(sys.argv) > 1 else 3600

# 只捕获 xDroid 相关路径。
# 注意: 不能写 "binder_" —— 源码目录是 /usr/src/binder/binder.c,
# 路径里没有下划线, 带下划线会把整个目录漏掉(实测踩过的坑)。
KEYWORDS = ("xdroid", "ashmem", "asg", "binder")

INTERESTING_SUFFIX = {".c", ".h", ".conf", ".mk", ".ko", ".mod", ".mod.c"}
INTERESTING_NAMES = {"Makefile", "Kbuild", "dkms.conf", "clean", "install"}

IN_CREATE = 0x00000100
IN_MODIFY = 0x00000002
IN_CLOSE_WRITE = 0x00000008
IN_MOVED_TO = 0x00000080
IN_DELETE = 0x00000200
WATCH_MASK = IN_CREATE | IN_MODIFY | IN_CLOSE_WRITE | IN_MOVED_TO | IN_DELETE

libc = ctypes.CDLL(ctypes.util.find_library("c"), use_errno=True)
fd = libc.inotify_init()
if fd < 0:
    raise SystemExit("inotify_init 失败, 需要 root 或 CAP_SYS_ADMIN")

wd_paths: dict[int, Path] = {}


def _add_watch(path: Path) -> int:
    wd = libc.inotify_add_watch(fd, str(path).encode(), WATCH_MASK)
    if wd < 0:
        raise OSError(ctypes.get_errno(), "inotify_add_watch", str(path))
    return wd


def add_tree(root: Path) -> int:
    """递归注册 root 下所有已存在的目录。"""
    count = 0
    for dirpath, _dirnames, _files in os.walk(root):
        p = Path(dirpath)
        try:
            wd = _add_watch(p)
        except OSError:
            continue
        wd_paths[wd] = p
        count += 1
    return count


SNAPSHOT_ROOT.mkdir(parents=True, exist_ok=True)


def should_capture(path: Path) -> bool:
    """路径属于 xDroid 相关 + 是源码/构建产物。"""
    low = str(path).lower()
    if not any(k in low for k in KEYWORDS):
        return False
    if path.name in INTERESTING_NAMES:
        return True
    return path.suffix in INTERESTING_SUFFIX


_seen: dict[str, tuple[int, int]] = {}


def snapshot(path: Path, why: str) -> None:
    """复制文件到快照目录, 保留相对路径; 内容未变则跳过去重。"""
    try:
        st = path.stat()
    except OSError:
        return
    sig = (st.st_size, st.st_mtime_ns)
    key = str(path)
    if _seen.get(key) == sig:
        return

    rel = path.relative_to("/") if str(path).startswith("/") else Path(path.name)
    dest = SNAPSHOT_ROOT / rel
    dest.parent.mkdir(parents=True, exist_ok=True)

    # 与已有副本内容相同则只更新记录, 不再写盘
    if dest.exists() and dest.stat().st_size == st.st_size:
        try:
            if dest.read_bytes() == path.read_bytes():
                _seen[key] = sig
                return
        except OSError:
            pass
        dest = dest.with_name(f"{dest.name}.{time.strftime('%H%M%S')}")

    try:
        shutil.copy2(path, dest)
    except OSError as exc:
        print(f"[warn] 复制失败 {path}: {exc}", flush=True)
        return
    _seen[key] = sig
    print(f"[capture] {why} {path} -> {dest}", flush=True)


def scan_existing(root: Path) -> None:
    """补扫已存在的源码文件(防止脚本启动时源码已落地)。"""
    for dirpath, _dirnames, filenames in os.walk(root):
        for fn in filenames:
            p = Path(dirpath) / fn
            if should_capture(p):
                snapshot(p, "[预扫]")


def rescan_targets() -> None:
    """定向重扫 xDroid 源码目录, 兜底 inotify 可能遗漏的事件。

    inotify 的目录 watch 在"整个目录被快速创建并填充"时可能来不及注册,
    定时重扫可确保不丢文件。只扫已知目标目录, 避免遍历无关源码。
    """
    bases: list[Path] = []
    for name in ("ashmem", "asg", "binder"):
        p = Path("/usr/src") / name
        if p.is_dir():
            bases.append(p)
    bases += [p for p in Path("/usr/src").glob("*xdroid*") if p.is_dir()]
    bases += [p for p in Path("/var/lib/dkms").glob("*xdroid*") if p.is_dir()]
    for b in bases:
        for dirpath, _dirnames, filenames in os.walk(b):
            for fn in filenames:
                p = Path(dirpath) / fn
                if should_capture(p):
                    snapshot(p, "[重扫]")


stop = False


def _on_signal(_sig, _frm):
    global stop
    stop = True


signal.signal(signal.SIGINT, _on_signal)
signal.signal(signal.SIGTERM, _on_signal)

total = 0
for r in WATCH_ROOTS:
    if r.is_dir():
        total += add_tree(r)
        scan_existing(r)

print(f"[init] 已注册 {total} 个目录 watch, 快照目录 {SNAPSHOT_ROOT}", flush=True)
print(f"[init] 监听时长 {DURATION_S}s, Ctrl-C 可提前退出", flush=True)

EVENT_HDR = struct.Struct("iIII")  # wd, mask, cookie, len
buf = b""
last_rescan = 0.0
deadline = time.monotonic() + DURATION_S

while not stop and time.monotonic() < deadline:
    now = time.monotonic()
    if now - last_rescan >= 2.0:  # 每 2 秒定向重扫兜底
        last_rescan = now
        try:
            rescan_targets()
        except OSError as exc:
            print(f"[warn] 重扫失败: {exc}", flush=True)

    timeout = min(1.0, max(0.0, deadline - time.monotonic()))
    rlist, _, _ = select.select([fd], [], [], timeout)
    if fd not in rlist:
        continue
    try:
        buf += os.read(fd, 64 * 1024)
    except OSError:
        continue
    while len(buf) >= EVENT_HDR.size:
        wd, mask, _cookie, length = EVENT_HDR.unpack_from(buf)
        if len(buf) < EVENT_HDR.size + length:
            break
        raw = buf[EVENT_HDR.size:EVENT_HDR.size + length]
        buf = buf[EVENT_HDR.size + length:]
        name = raw.split(b"\0", 1)[0].decode("utf-8", "replace")
        base = wd_paths.get(wd)
        if base is None:
            continue
        target = base / name

        if mask & IN_CREATE and target.is_dir():
            # 新目录: 注册 watch; 若属于 xdroid 则立即补扫
            add_tree(target)
            if any(k in str(target).lower() for k in KEYWORDS):
                scan_existing(target)
        if mask & IN_DELETE:
            continue
        if name and target.exists() and should_capture(target):
            # CLOSE_WRITE 表示写完, 最适合抓取完整内容
            why = "[CLOSE]" if mask & IN_CLOSE_WRITE else "[WRITE]"
            snapshot(target, why)

print(f"[done] 监听结束, 快照位于 {SNAPSHOT_ROOT}", flush=True)
```

---

## 4. 脚本二：源码注入 inject_ashmem.py

保存为 `inject_ashmem.py`。它需要「已修补的 ashmem 源码目录」作为注入源
（见第 5 节），`sudo python3 inject_ashmem.py 3600` 运行。

```python
#!/usr/bin/env python3
"""把安装器解包的 ashmem 源码替换为已修补版本, 使编译真实成功。

为什么需要:
  安装器解包的 ashmem.c 用 Linux 5.x 的 shmem_file_setup 签名, 在 6.1+
  内核必然编译失败; 一旦失败, 安装器可能不再继续解包 asg/binder 源码,
  导致无法捕获。把解包结果换成本地修补版, 安装器就会继续推进。

  原始版本已由 capture_ko.py 完整备份, 不受影响。

用法:
  sudo python3 inject_ashmem.py [持续秒数, 默认 3600]
"""

from __future__ import annotations

import hashlib
import shutil
import sys
import time
from pathlib import Path

# 本地已修补的 ashmem 源码目录(见第 5 节, 从捕获快照复制后打补丁)
SRC = Path.home() / "apps" / "xdroid" / "xdroid-ko"
LOG = Path("/tmp/inject-ashmem.log")
DURATION_S = int(sys.argv[1]) if len(sys.argv) > 1 else 3600

FILES = ["ashmem.c", "ashmem.h", "deps.c", "Makefile", "uapi/ashmem.h"]


def log(msg: str) -> None:
    line = f"[inject] {msg}"
    print(line, flush=True)
    try:
        with LOG.open("a") as fh:
            fh.write(line + "\n")
    except OSError:
        pass


def digest(p: Path) -> str | None:
    try:
        return hashlib.sha256(p.read_bytes()).hexdigest()
    except OSError:
        return None


def target_dirs() -> list[Path]:
    """所有可能出现 ashmem 源码的目录。

    安装流程会多次复制源码, 任一处漏改都会导致编译的仍是原版:
      1. 解包到 /usr/src/ashmem/
      2. dkms add 复制到 /var/lib/dkms/ashmem_xdroid/<ver>/source/
      3. dkms build 在 /var/lib/dkms/ashmem_xdroid/<ver>/build/ 就地编译
    """
    dirs: list[Path] = []
    src = Path("/usr/src/ashmem")
    if src.is_dir():
        dirs.append(src)
    dkms = Path("/var/lib/dkms")
    if dkms.is_dir():
        for base in dkms.glob("*ashmem*"):
            for sub in ("source", "build"):
                for cand in (base / sub,
                             *(p / sub for p in base.iterdir() if p.is_dir())):
                    if cand.is_dir():
                        dirs.append(cand)
    return dirs


def inject_once() -> int:
    """对所有目标目录执行一轮注入, 返回本轮覆盖的文件数。"""
    n = 0
    for target_dir in target_dirs():
        for rel in FILES:
            src = SRC / rel
            if not src.is_file():
                continue
            dst = target_dir / rel
            if digest(dst) != digest(src):  # 不存在或内容不同 → 覆盖
                dst.parent.mkdir(parents=True, exist_ok=True)
                try:
                    shutil.copy2(src, dst)
                    log(f"已替换 {dst} (原文件已由 capture_ko 备份)")
                    n += 1
                except OSError as exc:
                    log(f"替换失败 {dst}: {exc}")
    return n


def main() -> int:
    log(f"启动, 修补源 {SRC}, 时长 {DURATION_S}s, 动态扫描目标目录")
    deadline = time.monotonic() + DURATION_S
    total = 0
    while time.monotonic() < deadline:
        try:
            total += inject_once()
        except OSError as exc:
            log(f"轮次异常: {exc}")
        time.sleep(0.3)
    log(f"结束, 共替换 {total} 次")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

---

## 5. ashmem 补丁（唯一需要改的代码）

### 5.1 准备修补源目录

先用捕获到的原始源码建一个工作副本：

```bash
mkdir -p ~/apps/xdroid/xdroid-ko
cp -r ~/apps/xdroid/_ko_snapshot/usr/src/ashmem/. ~/apps/xdroid/xdroid-ko/
```

> 注意甄别版本：快照目录里 `ashmem.c` 若含 `legacy_to_vma_flags`
> 说明是已被注入过的修补版，原始版通常带时间戳后缀
> （如 `ashmem.c.135950`，且 `grep legacy_to_vma_flags` 无结果）。
> 打补丁要以**原始版**为底。

### 5.2 补丁内容

`ashmem.c` 第 413 行附近，原版：

```c
		/* ... and allocate the backing shmem file */
		vmfile = shmem_file_setup(name, asma->size, vma->vm_flags);
```

改为：

```c
		/* ... and allocate the backing shmem file */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 1, 0)
		/*
		 * Linux 6.1 把 VMA 标志位从 vm_flags_t 位图重构为 vma_flags_t
		 * 结构体, shmem_file_setup() 的第三个参数随之改为 vma_flags_t。
		 * 厂商源码停留在 5.x 的位图写法, 这里用内核提供的官方兼容桥
		 * legacy_to_vma_flags() 做一次转换。
		 */
		vmfile = shmem_file_setup(name, asma->size,
					  legacy_to_vma_flags(vma->vm_flags));
#else
		vmfile = shmem_file_setup(name, asma->size, vma->vm_flags);
#endif
```

依据：内核 6.1+ 在 `include/linux/mm_types.h` 提供官方兼容桥：

```c
static __always_inline vma_flags_t legacy_to_vma_flags(vm_flags_t flags)
{
	vma_flags_t ret = EMPTY_VMA_FLAGS;
	vma_flags_overwrite_word(&ret, flags);
	return ret;
}
```

> deps.c 的 4 个警告（`-Wmissing-prototypes`、`-Wunused-result`）是厂商
> 源码原有写法，不影响构建，不必处理。
>
> deps.c 的作用说明：5.7+ 起 `kallsyms_lookup_name` 不再导出，厂商用
> `sprint_symbol` 在内核内存中暴力扫描其地址，进而拿到 `shmem_zero_setup`、
> `vfs_read` 等未导出符号。该机制在 7.1 内核上实测仍然有效。

### 5.3 编译验证（可选，安装前可先手动试编）

```bash
cd ~/apps/xdroid/xdroid-ko
make KERNEL_SRC=/usr/lib/modules/$(uname -r)/build
# 成功标志: 生成 ashmem_xdroid.ko
```

---

## 6. 完整操作流程

```bash
# ── 0. 准备 ─────────────────────────────────────────────
cd ~/apps/xdroid
# 清理旧的 dkms 残留注册(如有)
for m in ashmem_xdroid asg_xdroid binder_xdroid; do
  sudo dkms remove "$m/2.0" --all 2>/dev/null
done

# ── 1. 启动捕获与注入(setsid 脱离会话, 避免后台任务结束连带被杀) ──
sudo setsid nohup python3 capture_ko.py 3600    > /tmp/capture.log 2>&1 < /dev/null &
sudo setsid nohup python3 inject_ashmem.py 3600 > /tmp/inject.log  2>&1 < /dev/null &

# 确认进程存活
pgrep -af 'capture_ko|inject_ashmem'

# ── 2. 运行官方安装器(普通用户, 不要 sudo) ──────────────
./xDroidInstall-x86_64-v13.2.380-20250306.run

# ── 3. 安装完成后检查捕获结果 ────────────────────────────
ls _ko_snapshot/usr/src/
# 预期: ashmem/  asg/  (binder 走 binderfs, 无源码属正常)

# ── 4. 停掉后台脚本 ─────────────────────────────────────
for p in $(pgrep -f 'capture_ko[.]py'); do sudo kill "$p"; done
for p in $(pgrep -f 'inject_ashmem[.]py'); do sudo kill "$p"; done
```

> 关键细节：`setsid` 不可省。直接用 `nohup ... &` 放在 shell 后台任务里，
> 任务结束时子进程会被连带清理（实测日志会出现 `[done] 监听结束`，
> 进程实际已死，导致第一次捕获失败）。

---

## 7. ashmem 模块的安装与 udev 修正

安装器用 DKMS 成功编译后会自动安装模块。如需手动安装：

```bash
sudo install -m 644 ashmem_xdroid.ko /usr/lib/modules/$(uname -r)/extra/
sudo depmod -a
sudo modprobe ashmem_xdroid
ls -l /dev/ashmem_xdroid    # 应存在
```

### udev 规则缺陷与修正

厂商规则 `/etc/udev/rules.d/99-xdroid.rules` 有 bug：

```conf
KERNEL=="ashmem_xdroid*", NAME="%k", MODE="0666"
```

这些是 **misc 设备**（sysfs 路径 `devices/virtual/misc/ashmem_xdroid`）。
udev 的 `KERNEL` 匹配的是 uevent 的 kernel 字段即父设备名 `misc`，
所以规则**永远不匹配**，节点保持默认 `0600 root-only`。

修正为（先 `sudo cp 99-xdroid.rules 99-xdroid.rules.orig` 备份）：

```conf
SUBSYSTEM=="misc", KERNEL=="ashmem_xdroid",   NAME="ashmem_xdroid",   MODE="0666"
SUBSYSTEM=="misc", KERNEL=="binder_xdroid",   NAME="binder_xdroid",   MODE="0666"
SUBSYSTEM=="misc", KERNEL=="hwbinder_xdroid", NAME="hwbinder_xdroid", MODE="0666"
SUBSYSTEM=="misc", KERNEL=="vndbinder_xdroid", NAME="vndbinder_xdroid", MODE="0666"
SUBSYSTEM=="misc", KERNEL=="asg_xdroid",      NAME="asg_xdroid",      MODE="0666"

KERNEL=="loop*", ENV{UDISKS_IGNORE}="1"
ENV{ID_FS_TYPE}=="squashfs", ENV{UDISKS_IGNORE}="1"
```

生效：

```bash
sudo udevadm control --reload
sudo udevadm trigger --subsystem-match=misc
sudo udevadm settle
ls -l /dev/ashmem_xdroid    # crw-rw-rw-
```

> 安全提示：`MODE="0666"` 意味着本机任意用户可读写容器共享内存与 binder
> 通道，是已知攻击面。遵循厂商行为以保证功能；如需收紧可改 `0660` + 组授权。

---

## 8. binder 与 asg 为什么不用编译

### 8.1 binder 走 binderfs

内核已内置 Rust 版 binder 驱动（`CONFIG_ANDROID_BINDER_IPC_RUST=y`），
`rust_binder` 是 builtin，`rust_devices` 参数运行时只读且被置空，
所以没有静态 `/dev/binder*` 节点。

但安装器实际使用 **binderfs** 在容器启动时动态分配：

```
/dev/binderfs/
  binder-control      242:0
  xdroid-binder1      242:1
  xdroid-hwbinder1    242:2
  xdroid-vndbinder1   242:3
```

节点名正是 xDroid 期望的形式，**内核侧零改动**。

#### 备用方案：外部编译 C 版 binder_linux（实测不需要，存档备用）

内核 `drivers/android/binder.c` 的设备名直接来自 `devices` 模块参数，
无白名单校验：

```c
while ((device_name = strsep(&device_tmp, ",")))
        ret = init_binder_device(device_name);
```

把 `binder.c binder_alloc.c binder_netlink.c` 及头文件复制到外部目录，
配合下面的 Makefile 即可编成可加载模块，加载时传
`devices=binder_xdroid,hwbinder_xdroid,vndbinder_xdroid` 获得私有节点名：

```makefile
# 外部目录编译 Android Binder IPC 驱动 (binder_linux)
KERNEL_SRC ?= /lib/modules/$(shell uname -r)/build
MODULE_NAME := binder_linux

ccflags-y += -I$(src)

obj-m += $(MODULE_NAME).o
# 与内核 drivers/android/Makefile 一致: 去掉 binderfs 与 Rust 版
$(MODULE_NAME)-y := binder.o binder_alloc.o binder_netlink.o

.PHONY: all clean install load

all:
	$(MAKE) -C $(KERNEL_SRC) M=$(PWD) modules

clean:
	$(MAKE) -C $(KERNEL_SRC) M=$(PWD) clean

install: all
	install -d /usr/lib/modules/$(shell uname -r)/extra
	install -m 644 $(MODULE_NAME).ko /usr/lib/modules/$(shell uname -r)/extra/
	depmod -a

load: install
	modprobe $(MODULE_NAME)
	@ls -l /dev/binder* /dev/hwbinder* /dev/vndbinder* 2>/dev/null || true
```

推荐用 `/etc/modprobe.d/xdroid-binder.conf` 固化参数：

```conf
options binder_linux devices=binder_xdroid,hwbinder_xdroid,vndbinder_xdroid
```

### 8.2 asg 未被使用

`asg_defs.h` 全文：

```c
#ifndef __ASG_DEFS_H
#define GOLDFISH_ADDRESS_SPACE_AREA_SIZE            (16ULL << 30ULL)
#endif
```

`goldfish_address_space.h` 定义设备名 `"asg_xdroid"` 与分配/ping 的 ioctl
结构，用途是限制 Android 模拟器（goldfish）的虚拟地址空间（16 GB）。
xDroid 以容器方式运行不走模拟器路径，该模块未被加载，Android 正常工作。

如需试装（源码已在快照中）：

```bash
cd _ko_snapshot/usr/src/asg
make && sudo insmod asg_xdroid.ko
```

---

## 9. 安装不完整的连带修复（仅在安装中断过时需要）

若安装器曾在 ashmem 阶段失败，用户态文件可能缺一半，症状：

```
xAppCenter: error while loading shared libraries: libssl.so.1.0.0
```

从载荷补齐（库分布在 `usr/bin` 与 `usr/lib/x86_64-linux-gnu` 两处）：

```bash
# 解出载荷中的库
tar xzf xdroid.tar.gz -C /tmp/fix --strip-components=1 \
    --wildcards './usr/lib/x86_64-linux-gnu/lib*'

sudo mkdir -p /opt/xdroid/lib/x86_64-linux-gnu
sudo cp -a /tmp/fix/usr/lib/x86_64-linux-gnu/. /opt/xdroid/lib/x86_64-linux-gnu/

# 验证
LD_LIBRARY_PATH=/opt/xdroid/usr/bin:/opt/xdroid/lib/x86_64-linux-gnu \
  ldd /opt/xdroid/usr/bin/xAppCenter | grep 'not found'   # 无输出即正常
```

其余缺项（systemd 单元、桌面入口）在载荷根目录与安装包 `data/` 里均有：

```bash
sudo cp xdroid-manager.service xdroid-upgradebackend.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo cp data/xAppCenter.desktop /usr/share/applications/
sudo cp data/xAppCenter.png /usr/share/icons/
```

---

## 10. 厂商包的其他缺陷（备忘）

1. **引导脚本 `rm -rf` 未加引号**：`rm -rf ${DIR_INSTALL}`，`$HOME` 为空时
   路径退化，存在误删风险。
2. **无载荷完整性校验**：无 MD5/CRC，篡改不会被发现。
3. **仅支持 RPM/DEB 系**：检测 `apt-get`/`yum`/`zypper`/`pacman`，
   在 Arch 上官方流程无法完整跑通，必须手工补齐。
4. **安装器会还原被替换的 `/usr/bin/dkms`**：包装器拦截不可靠，
   应改用文件系统层（inotify）监听。
5. **udev 规则永不生效**（见第 7 节）。

---

## 11. 验证清单（安装完成后）

```bash
# 模块
lsmod | grep ashmem_xdroid                          # 已加载
ls -l /dev/ashmem_xdroid                            # crw-rw-rw-
ls /dev/binderfs/                                   # xdroid-binder1 等

# 服务
systemctl is-active xdroid-manager.service          # active
ps aux | grep -E 'xDroidServiceManager|lxc monitor' # 均有进程

# 容器内 Android 进程(以下任一种方式)
sudo nsenter -t $(pgrep -f 'lxc monitor' | head -1) -n -m -p -- \
  ps -eo pid,comm | grep -E 'system_server|surfaceflinger|xdroidd'

# 网络
ip addr show xdroid-0                               # 192.168.252.1/24
```

全部通过即表示 Android 容器完整运行（可再打开 xAppCenter 装一个 App 验证）。

---

## 12. 文件清单

```
~/apps/xdroid/
├── capture_ko.py              源码捕获脚本（第 3 节）
├── inject_ashmem.py           源码注入脚本（第 4 节）
├── xdroid-ko/                 修补版 ashmem 源码（第 5 节，注入源）
│   ├── ashmem.c (已打补丁)
│   ├── deps.c  ashmem.h  Makefile  uapi/ashmem.h
├── binder-ko/Makefile         C 版 binder 外部构建脚本（第 8.1 节，备用）
├── _ko_snapshot/usr/src/      捕获的原始源码（务必保留）
│   ├── ashmem/   ashmem.c 1017 行 + deps.c 178 行
│   └── asg/      goldfish_address_space.c 35 KB
└── xDroidInstall-*.run        官方安装包
```
