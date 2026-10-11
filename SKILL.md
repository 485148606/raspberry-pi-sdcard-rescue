---
name: 树莓派 SD 卡离线救援
description: >-
  【树莓派 SD 卡离线救援】>- 【树莓派 SD 卡离线救援】树莓派进不去时的 SD 卡离线救援法。当忘记树莓派密码 / SSH 连不上 / 需要重置密码或注入 SSH 公钥时，把 SD 卡插到 Windows 电脑，直接改 FAT 启动分区上的配置，让树莓派下次开机自动执行救援脚本。覆盖 Raspberry Pi Imager 配置残留的取证、yescrypt 哈希离线校验、cloud-init 与 systemd.run 双通道注入、以及一系列必踩的坑（CRLF、lock_passwd 误伤、FAT32 无执行位）。触发词：树莓派密码忘了、树莓派连不上、重置树莓派密码、给树莓派装SSH密钥、raspberry pi rescue、SD卡救援、pi 密码找回。
agent_created: true
---

# 树莓派 SD 卡离线救援

忘记树莓派密码、或 SSH 彻底进不去时，**不用重刷系统**。把 SD 卡拔下来插电脑，改启动分区上的几个文本文件，开机自动把公钥装好、密码重置掉。

## 适用前提

- 树莓派能开机、卡能插到电脑上（Windows 直接能读写 FAT32 启动分区）
- 卡的启动分区（`bootfs`，内含 `bcm27*.dtb` / `kernel*.img`）——Windows 会给它一个盘符
- 机型无关，但 **Pi 3B / 3B+ 及更早机型没有默认 USB 启动**（Pi 4/5 才有），所以离线救援**只能走 SD 卡**；判断机型可看启动分区里有没有对应的 `bcm2710-rpi-3-b.dtb`（Pi 3）等 dtb 文件

## 第 0 步：先别急着救援，能拷的就先拷出来

树莓派密码不一定是真的忘了。按这个顺序找，很多时候不用动卡：

1. **Raspberry Pi Imager 记着上次的设置**（它默认勾"记住"，所以哪怕用户认为"烧录时没填任何东西"，配置照样生效）。
   注册表：`HKCU\Software\Raspberry Pi\Raspberry Pi Imager\imagecustomization`
   里面有 `hostname`、`sshUserName`、`sshUserPassword`、`wifiSSID`、`wifiPasswordCrypt`。
   ```
   reg query "HKCU\Software\Raspberry Pi\Raspberry Pi Imager" /s
   ```
2. **验证这套配置确实是生效的那套**：用 mDNS 查主机名。`ping <hostname>.local` 能解析到树莓派 IP 就说明定制生效了（`raspberrypi.local` 解析失败 = 主机名被改过）。
3. **WiFi 明文密码**从本机就能拿（用户常复用）：
   ```
   netsh wlan show profile name="SSID" key=clear   # 看"关键内容"
   ```
4. **`sshUserPassword` 是 yescrypt 哈希（`$y$...`），明文永不落盘。** Imager 源码 `customization_generator.cpp` 里 `sshUserPasswordPlain` 只活在运行时内存，`setPersistedCustomisationSetting` 只持久化哈希。别指望在磁盘上翻出明文。
5. 要撞哈希的话：`pyescrypt` 能离线校验 MCF 格式，不用碰网络。
   ```python
   import pyescrypt
   Y = pyescrypt.Yescrypt(mode=pyescrypt.Mode.MCF)   # 必须指定 MCF，否则报 "MCF string passed to a JSON instance"
   Y.compare(b'candidate', b'$y$jB5$...')            # 不抛异常 = 命中；抛 WrongPassword = 不对
   ```
   单次约 7ms，12 核并行约 700 次/秒。**务必先自检**：`Y.digest(b'x', salt=b'...')` 生成一个再 `compare`，确认能命中，否则会拿到假阴性。
   中文弱口令字典（可用 `ghproxy.net` 中继 raw.githubusercontent）：
   `TheKingOfDuck/fuzzDicts/passwordDict/` 下的 `top19576.txt`、`某集团下发的弱口令字典.txt`、`birth.txt`。

## 第 1 步：认清卡用的是哪种引导机制

看启动分区根目录里有什么：

| 看到 | 机制 | 救援走哪条 |
|---|---|---|
| `user-data` / `meta-data` / `network-config` | **cloud-init**（较新 pi-gen 镜像） | 改这几个文件 + 改 `instance_id` |
| `cmdline.txt` 里有 `systemd.run=` | systemd firstrun | 直接换 `firstrun.sh` |
| 只有 `config.txt` / `cmdline.txt` | 裸镜像 | 用 `systemd.run` 注入（下面第 2 步） |

**启动分区运行时挂在哪**：`config.txt` 里若有 `/boot/firmware/overlays/README` 这类注释 → 挂在 `/boot/firmware`（Bookworm 及以后）。老镜像挂 `/boot`。

## 第 2 步：注入（双通道，互为兜底）

**两条通道都写上，且设成同一个密码 / 同一把公钥**，哪条生效都能进。

### 通道 B 优先（cloud-init）—— 实测这一条才是真正管用的

⚠️ **实测结论，先看这条**：在一台 RPi OS 13 (trixie) 的 cloud-init 镜像上，`systemd.run` **压根没被执行**（引导后回去看：`firstrun.sh` 原封不动还在、`rescue.log` 从未生成），而 **cloud-init 的 `runcmd` 一次就把公钥和密码搞定了**。
所以：**cloud-init 镜像是主通道，`systemd.run` 最多算锦上添花，别把希望压在它身上。**
判断顺序也建议反过来：先确认卡上有没有 `user-data`——有就走 cloud-init，没有才去折腾 `systemd.run`。

### 通道 B：cloud-init（改 instance_id 逼它重跑）

- `meta-data`：把 `instance_id:` 改成新值。原注释明确写了改它会触发重新走首次配置。
- `user-data`：
  - `users[0].passwd:` 换成新密码的 yescrypt 哈希
  - `users[0]` 下加 `ssh_authorized_keys:` 列表塞公钥
  - 追加顶层 `chpasswd:`（`expire: false`）和 `runcmd:`（root 身份再兜一遍装公钥+改密码）
- `cmdline.txt` 追加 ` ds=nocloud;i=<新id>`，让 cloud-init 的 `check_instance_id()` 能对上新实例（Imager 自己也是这么干的）。

**强烈建议两处都设**：`users[].passwd`（哈希）和顶层 `chpasswd.list`（明文），互为冗余。**runcmd 是最实的一道保险**——实测就是它救的场。

### 通道 A：`systemd.run`（内核参数，可能不触发，仅作补充）

`cmdline.txt` **是单行、结尾不要换行**。在末尾追加：

```
 ds=nocloud;i=<新id> systemd.run=/bin/sh /boot/firmware/firstrun.sh
```

🚨 **千万不要加 `systemd.unit=kernel-command-line.target`。** 这个踩过的坑详见下面"坑 0"——它是 Imager 给**老镜像**用的，照搬到 Bookworm 会让树莓派**彻底失联**（绿灯常亮、无网络、无 SSH、僵在最小启动目标）。也不要加 `systemd.run_success_action=reboot`：不加时脚本跑完系统继续正常启动，反而更安全、少一次重启。

- **必须写成 `/bin/sh 脚本路径`**，不要只写脚本路径。FAT32 没有可执行位，直接执行会失败；套一层 `/bin/sh` 绕开。
- 参数会被 systemd 按空白切词，`/bin/sh` 和路径都不含 `=`，不会被误当成别的内核参数。
- 不加 `systemd.unit=` 时，systemd 会在**正常启动流程走完之后**才执行脚本——此时 `/boot/firmware` 已由 fstab 挂载、根文件系统可写，脚本才能真的跑起来。脚本失败也无任何副作用（系统已经正常起来了）。

`firstrun.sh` 写在同一分区根目录，**必须 LF 换行**：

```sh
#!/bin/sh
# 关键：先摘掉自己再干活 —— 中途无论怎么失败都不会重启循环
BOOT=""
for d in /boot/firmware /boot; do
  if [ -f "$d/cmdline.txt" ]; then BOOT="$d"; break; fi
done
[ -z "$BOOT" ] && BOOT=/boot/firmware

# ---- 第一步：先清理 cmdline，杜绝任何重启循环可能 ----
sed -i 's| systemd.run[^ ]*||g'   "$BOOT/cmdline.txt" 2>/dev/null
sed -i 's| /boot/firmware/firstrun.sh||g' "$BOOT/cmdline.txt" 2>/dev/null
sed -i 's| /boot/firstrun.sh||g'  "$BOOT/cmdline.txt" 2>/dev/null
sed -i 's| ds=nocloud;[^ ]*||g'   "$BOOT/cmdline.txt" 2>/dev/null
rm -f "$BOOT/firstrun.sh"

# ---- 第二步：干活，全程写日志回启动分区（这步是排错的生命线）----
{
echo "=== rescue start $(date) ==="
echo "BOOT=$BOOT"
mkdir -p /home/pi/.ssh
touch /home/pi/.ssh/authorized_keys
grep -q "my-rescue-tag" /home/pi/.ssh/authorized_keys || echo '<公钥，注释带 my-rescue-tag>' >> /home/pi/.ssh/authorized_keys
chmod 700 /home/pi/.ssh && chmod 600 /home/pi/.ssh/authorized_keys
chown -R pi:pi /home/pi/.ssh
echo "key lines: $(wc -l < /home/pi/.ssh/authorized_keys)"

echo 'pi:<新密码>' | chpasswd
echo "chpasswd rc=$?"
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sed -i 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
systemctl restart ssh 2>/dev/null || systemctl restart sshd 2>/dev/null
echo "=== rescue done $(date) ==="
} >"$BOOT/rescue.log" 2>&1

exit 0
```

清理用 `s| systemd.run.*||g` 能一把删掉后面所有参数——所以**必须把 `systemd.run` 追加在行尾**。

### 生成哈希

用 pyescrypt 生成本机可校验的 `$y$` 哈希（盐随意，16 字符起）：

```python
import pyescrypt
Y = pyescrypt.Yescrypt(mode=pyescrypt.Mode.MCF)
h = Y.digest(b'<新密码>', salt=b'<16+字符盐>')
Y.compare(b'<新密码>', h)   # 必须不抛异常
```

## 第 3 步：必有校验（这一步救过命）

改完**逐项断言**再拔卡：

```python
import yaml, pyescrypt
d = yaml.safe_load(open('F:/user-data', encoding='utf-8'))
u = d['users'][0]
assert u['lock_passwd'] is False                      # ← 见坑 2
assert 'wxy...' in d['chpasswd']['list']
assert 'wxy...' not in str(d).replace(...)            # 确认没有旧密码残留
Y = pyescrypt.Yescrypt(mode=pyescrypt.Mode.MCF)
Y.compare(b'<新密码>', u['passwd'].encode())          # 哈希对得上
fr = open('F:/firstrun.sh', 'rb').read()
assert b'\r' not in fr                                # ← 见坑 1
# 还要 grep 全卡确认旧密码 0 残留：
# grep -c "旧密码" /f/user-data /f/firstrun.sh   → 全部 0
```

## 四个必踩的坑

**坑 0（最狠，会让树莓派彻底失联）：照抄 Imager 的 `systemd.unit=kernel-command-line.target`。**
实测在 Bookworm/Trixie 的 cloud-init 镜像上，加了它之后：树莓派**绿灯常亮、ping 不通、SSH 不通、全网段扫不到 MAC**，看起来像 "没连上 WiFi"，其实是**系统停在最小启动目标，网络和服务压根没起来**。
机理：该参数把默认 target 改成 `kernel-command-line.target`（几乎没有依赖），系统只启动到那儿就停。Imager 敢用是因为它的脚本**必定成功并自删 cmdline**，跑完立刻重启进正常系统；而**这个极早期阶段 `/boot/firmware` 尚未挂载**（Bookworm 是开机后由 fstab 挂上去的，老的 RPi OS 才是开机就在 `/boot`），脚本文件找不到 → 执行失败 → 又没进正常启动 → 僵死。
**正确做法**：只写 `systemd.run=/bin/sh <脚本路径>`，**不加 `systemd.unit=`，也不加 `run_success_action=reboot`**，让脚本在正常启动完成后执行。
**取证方法**（判断脚本到底有没有跑）：把卡拔回来插电脑，看三件事——① `rescue.log` 有没有生成；② `firstrun.sh` 是否还在；③ `cmdline.txt` 的 mtime 有没有变。**三者全都没变 = 脚本压根没被执行**，别在密码/公钥上找问题，直接去查引导参数。

1. **CRLF 换行会让 `firstrun.sh` 直接跑不起来。** 用 Python 写文件时**不要** `.replace("\n", "\r\n")`；FAT 不做换行转换，写什么就是什么。用 `open(p,'wb')` + `encode()` 写 LF。校验 `b'\r' not in content`。
2. **正则 `passwd:\s*\S+` 会误伤 `lock_passwd: false`**，把它换成哈希 → cloud-init 视作真值 → **账户被锁死**。必须锚定行首缩进：`re.sub(r"(?m)^  passwd:\s*\S+$", ...)`，改完 `assert 'lock_passwd: false' in content`。
3. **改密码要改全，漏一处就白干。** 一共四处：`user-data` 的 `passwd:` 哈希、`user-data` 的 `chpasswd.list`、`user-data` 的 `runcmd`、`firstrun.sh` 的 `chpasswd` 行。改完必须 grep 全卡扫旧密码。

## 第 4 步：交付与收尾

- 拔卡前先在 Windows 侧**安全弹出**盘符（托盘"安全删除硬件"）。
- 备份原始文件（`cmdline.txt` / `user-data` / `meta-data` / `network-config` / `config.txt`）到工作区，改完把新版本也复制一份留档，MD5 对比确认落盘。
- 告诉用户：插卡回板 → 开机 → **等 1~3 分钟** → 然后 SSH 进来。
- 优先用 `ssh pi@<hostname>.local`（mDNS/avahi）而不是硬编码 IP，避免 DHCP 换地址；要固定可用路由器按 MAC 做静态绑定。

### 收尾（改成走 SSH，别再让用户拔卡）

进去之后**启动分区在运行时挂在 `/boot/firmware` 且可写**，直接远程清理，一次都不用再拆机：

```sh
# RPi OS 默认 sudo 要密码，没配 NOPASSWD → 必须用 sudo -S 透传
PW='<新密码>'
echo "$PW" | sudo -S cp -a /boot/firmware/{cmdline.txt,meta-data,user-data,network-config} ~/boot_backup_$(date +%F)/
scp cmdline.txt meta-data user-data pi@<host>:/tmp/     # 从 Windows 侧传清理版
echo "$PW" | sudo -S cp /tmp/{cmdline.txt,meta-data,user-data} /boot/firmware/
echo "$PW" | sudo -S rm -f /boot/firmware/firstrun.sh /boot/firmware/rescue.log
echo "$PW" | sudo -S sync
```

清理时**保留 `user-data` 里的新密码哈希和公钥**（否则下次 cloud-init 重跑会把密码打回旧的未知哈希），但可以把明文 `chpasswd.list` / `runcmd` 拿掉——原始设计本来就只存哈希，别把明文密码留在 FAT 分区上。

**收尾后必须重启验证一遍**：`sudo reboot` → 等它自己回到线上（Pi 3B 约 50 秒）→ 再 SSH 连一次。**改完配置不重启测一遍等于没验证。**

## 排错

- 开机后连不上、且**绿灯常亮** → 先怀疑引导参数（见"坑 0"），别在密码/公钥上找问题。
- 要判断脚本到底有没有跑：把卡拔回来插电脑，看 ① `rescue.log` 有没有生成；② `firstrun.sh` 是否还在；③ `cmdline.txt` 的 mtime 有没有变。**三者全没变 = 脚本压根没被执行。**
- **`systemd.run` 无效是常态**（本案例就是），但 cloud-init 的 `runcmd` 生效了 → 卡拔回来发现 `rescue.log` 不存在、公钥却已装好，就是这个情况，属正常，别当成故障。
- 日志里有 `chpasswd rc=0` 但 SSH 还连不上 → 确认 `sshd` 起来了（RPi OS 的服务名是 **`ssh`**，不是 `sshd`）。
- 确认能进之后，别再靠密码——公钥已经装好了，`ssh -i <私钥> pi@<host>` 免密直连。
