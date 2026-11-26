# MacWaveOrg/Bottle

Summerwave（以及 4.x 的 MacWave）的**预编译产物存放处**。以前每条索引记录只有一个指向上游的
`url`，用户直接从别人的服务器下载；4.0 起改成先由 CI 取下来放进这里，用户只从我们这里取。

这样解决三件事：

1. 上游被攻破或出现漏洞时，不会直接波及用户 —— 下发的字节是我们处理并验证过的；
2. 上游崩溃、删 release、换 CDN 时，用户不会因此装不上；
3. 用户装机量大了也不会给上游造成压力。

> **1.0 有一处根本变化**：产物**不再要求与上游逐字节一致**。重定位与前缀改写都在发布前
> 做完（见下面的「产物契约」）。因此索引里的 `sha256` 描述的是**这里发布的这份字节**，
> 不再等于上游公布的值。

---

## 一、存放形态：一个包一个 Release

**资产放在 Release 里，不放在仓库文件里。** 硬限制：git 单文件超过 100 MiB 直接拒绝推送，
而 `ollama` 单个产物已经 172.9 MB。Release 单文件上限 2 GiB，官方不限总量与带宽。

```
tag    = 包名
资产名 = <包名>@<版本>.<os>-<arch><扩展名>
```

客户端拼出来的地址：

```
https://github.com/MacWaveOrg/Bottle/releases/download/jq/jq%401.8.2.macos-arm64.tar.gz
                                                     └┬┘ └──────────┬──────────┘
                                                     tag          资产名
```

为什么是这个形状 —— 只有一条标准：**国内镜像站要能用 rsync + nginx 原样提供**。清华、中科大、
阿里镜像的都是普通文件树，没有一个能模拟容器 registry 那套按 digest 取字节的接口。而 GitHub 的
下载路径恰好是 `/<tag>/<资产>`，所以镜像站只要保留域名之后那段相对路径：

```
默认：https://github.com/MacWaveOrg/Bottle/releases/download/jq/jq%401.8.2.macos-arm64.tar.gz
镜像：https://mirror.example.cn/MacWave/bottle/jq/jq%401.8.2.macos-arm64.tar.gz
                                  ↑ 域名之后完全相同
```

镜像路径里的 `MacWave/` 是给「一个镜像域名被多个产品共用」留的位置。
换镜像就是换一个环境变量（`MACWAVE_BOTTLE_DOMAIN`），镜像站不需要理解任何协议。

## 二、平台标签写在名字里（1.0）

单远端、不按平台拆仓库，所以**架构必须进文件名**。标签词表是封闭的：

| 一侧 | 取值 |
| --- | --- |
| os | `macos`（本仓库）；`linux` 属于将来的 Linux 侧，共用同一张词表 |
| arch | `arm64` / `amd64`（`x86_64` 在入口就归一成 `amd64`） |

```
jq@1.8.2.macos-arm64.tar.gz
jq@1.8.2.macos-amd64.tar.gz
```

标签由 `config/platforminfo.bottle_tag()` 生成，客户端拼地址、索引生成器、打包工具、迁移工具
全都调同一个函数 —— **改词表只改一处**，且两边有测试盯着。

**旧名字（`jq@1.8.2.arm64`）在过渡期并存。** 4.x 客户端的地址是硬拼旧形状的，所以改名期间
新旧两份都在（`scripts/migrate_index_layout.py --emit-gh-script` 默认保留旧名）。等 4.x 退场后
再清理。

### 支持范围（对外承诺）

| 架构 | 最低系统 | 谁来构建 |
| --- | --- | --- |
| `arm64` | **macOS 15 Sequoia** | Sequoia 本机 / `macos-15` runner |
| `amd64` | **macOS 12 Monterey** | 一台 Monterey 12 的 Intel 机器（SDK 决定地板） |

为什么按架构分而不是全库一个值：任何 Apple Silicon Mac 都跑得动 15，这个地板排除不了任何硬件，
只排除「能升但不升」的人；Intel 侧的地板由构建机的 SDK 决定，天然是 12，白捡了停更老 Intel Mac
的兼容面。构建侧的地板表与实现见 `MacWaveOrg/Ports` 的 `scripts/build-env.sh`。

每条版本记录里的 `min_os` 是**产物自己的声明**，由 `vtool -show-build` 读出；客户端据此警告并
询问（`-F` 可强制）。**产物没写 `-mmacosx-version-min` 时 `minos` 就等于构建机的系统版本** ——
这正是 `aria2c@1.37.0` 曾经「声明支持 12、实际要求 15」的原因。

## 三、产物契约（这里放的东西必须满足）

这一节是**对外承诺**：客户端与镜像都按它工作。

### 1. `sha256` 描述「这份字节」

索引记录里的 `sha256` 必须等于这个资产的字节哈希。客户端下载后校验，不符即拒绝安装。
它**不必**等于上游公布的值 —— 见下一条。

### 2. 允许、并且经常需要重打包

为了做到「客户端只下载 + 解包」，发布前会做三件事（`scripts/prepare_bottle.py`）：

| 做什么 | 为什么 |
| --- | --- |
| Mach-O 引用改成 `@loader_path/…` 相对形式，并重新 ad-hoc 签名 | 依赖装在 `Shore/<依赖>/<版本>/`，绝对路径与 `@rpath` 都会指错 |
| conda 的占位前缀（`info/has_prefix`）换成安装后的绝对路径 `<安装根>/Shore/<名>/<版本>` | 不换的话 `.pc` 的 `prefix`、`icu-config` 的 `default_prefix` 全指向上游构建机 |
| 重打成 `tar.bz2`，归档根 = 版本目录的内容（`bin/` `lib/` `include/` …） | 客户端解包后直接落在 `Shore/<名>/<版本>/` |

因此**「字节原样搬运」这条 4.x 的规则已经作废**。取而代之的是下面两条硬规则。

### 3. 产物里不许有「构建时才有」的路径

- **临时路径**：打包会解到 `mktemp -d`，那个目录打包完就没了，写进去就是死路；
- **构建机路径**：`/Users/…`、`/opt/homebrew/…`、`/tmp/…` 这类，运行时会读的都算违规。

`prepare_bottle.py` 打完包会**重读归档逐成员搜**本轮 `work`/`stage` 路径，命中就删产物并拒绝
继续；`has_prefix` 点过名的文件里若还留着占位符，同样拒绝出包。

**不算违规**的只有两类：`info/` 里的构建记录（`paths.json`、`recipe/`、清单侧的 `has_prefix`），
以及静态库 / 调试信息里的 `-fdebug-prefix-map=<老路径>=<新名字>` —— 判据是「运行时会去打开它吗」。

### 4. Mach-O 引用只有两种合法形态

- 系统库：`/usr/lib/…`、`/System/…`、`/Library/Apple/…`
- 其余一律 `@loader_path/…`，且**必须真的解析得到**（相对 staging 根算，staging 根 ≙ 安装根）

绝对路径即使指向 `/opt/macwave/…` 也不接受：安装位置虽然写死，但测试与沙箱要把包装进临时根再跑，
烧死绝对路径等于把「装完真跑一次」这条验证链拆掉。

### 5. 同一个包的两个架构都必须在

缺哪个架构就让那个平台明说「没有你这个平台的产物」，**不要**拿另一个架构的字节顶上 ——
用户会拿到 `exec format error`。

## 四、四条铁律

**一、只增不删。** 旧版本必须永远可下载。多版本共存是产品前提 —— 删掉一个资产，就等于让
`wave install <包>@<旧版本>` 永久失效。

**二、同名不覆盖。** 资产一旦发布就不再改动。需要重建同一版本时**换版本后缀**，例如
`1.7.1-1`（排序上大于 `1.7.1`，客户端的版本解析器已正确处理）。

**三、可复现。** 同一个上游、同一份 recipe、同一个地板，重打出来的 `sha256` 应当一致；不一致
就说明构建脚本漏了确定性开关。重打只读索引与上游，不读构建机的任何本地状态。

**四、发布与回填必须一起做。** 重打必然改字节，所以上传资产之后要立刻把新的 `file` / `sha256`
写回 `infosource` 的记录。两者错位的话用户下到的字节校验不过 —— 症状是「装到一半失败」，
而索引本身看起来完全正常。

## 五、与索引的关系

- 版本记录里的 `file` 决定**扩展名**（客户端据此选解压器），必须先于字节存在；
- `sha256` 是**这里的字节**的校验值（重打过就不再是上游的值）；
- `url` 只在 `--install-from-source` 时被读，正常安装完全不看它 —— 它是溯源线索；
- 重打过的记录会多两行：`relocated: "loader_path"`、`install: "tree"`。

平台标签与记录名的对应关系见 MacWave 仓库的 `docs/DATA-LAYER.md`。

## 六、谁往里放东西

- **重打**：`scripts/prepare_bottle.py`（`--all` 批量、`--cache` 复用下载、`--tsv` 输出待回填的
  `file`/`sha256`、`--resume` 断点续跑）。它**只下载、重打、打印**，不上传也不改索引。
- **上传**：`scripts/publish_payloads.py --staging <树> --write`，同名同大小跳过。
- **镜像同步**：镜像机上 `scripts/sync_payloads.py --source payload`，只补新增。

需要重打的情形通常只有两个：换了安装布局（1.0 的 `Shore/`）、或者上游产物带坏路径。

## 七、闸门（发布前必过）

| 闸门 | 查什么 |
| --- | --- |
| `prepare_bottle.py` 自带审计 | 重读归档：临时路径、`has_prefix` 占位符残留 |
| `relocate_rpath.py --check` | 每条引用都解析得到、每个 Mach-O 都有签名 |
| `Ports/scripts/selfbuilt_check.sh` | 架构、签名、绝对引用、文本里的构建目录、`minos` 地板 |
| `scripts/versions_index_test.sh` | 镜像的 `versions` 清单与实际产物一致（离线可跑） |

## 八、镜像站怎么同步

在镜像服务器上：

```bash
python3 scripts/sync_payloads.py --infosource <infosource 检出> \
  --out /www/wwwroot/mirror.example.cn/MacWave \
  --source payload --jobs 6
```

`--source payload` 表示从这里的 Release 取，而不是打扰上游。重复运行只补新增。
完整步骤见 MacWave 仓库的 `docs/MIRROR-SETUP.md`。

## 特殊软件包：`wave` / `macwave` / `linuxwave`

这三个名字容易和项目本身搞混，这里一次说清。tag 就是包名，所以「Bottle 里有没有这个 tag」
= 「有没有这个软件包」。

| 名字 | 这里有这个 tag 吗 | 是什么 | 装出来的命令 |
| --- | --- | --- | --- |
| `wave` | **没有，也不会有** | 包管理器自己的命令名（MacWave 与 LinuxWave 共用同一个 `wave`） | `wave` |
| `macwave` | **没有，也不会有** | macOS 包管理器本体，走 `MacWaveOrg/Selfupdate` 的自动更新，不进 Bottle | `wave` |
| `linuxwave` | **有** | 第三方软件包：[orhun/linuxwave](https://github.com/orhun/linuxwave)，Zig 写的音乐生成器，把熵变成 wav，MIT | `linuxwave` |
| `waveterm` | 没有 | Wave Terminal，Electron 桌面程序 | — |

要点：

- `linuxwave` 与 LinuxWave 包管理器同名纯属巧合。包管理器的命令叫 `wave`，软件包的命令叫
  `linuxwave`；包管理器本体不进这个仓库，所以不会出现「包管理器本体」的 tag。
- `wave` **永远不做包名** —— 名字里带 wave 的工具要按它真实的命令命名（例如 `waveterm`）。
  原因很具体：安装软件包时客户端会在 `links/` 下建同名快捷方式，如果某个包叫 `wave`，
  它就会和包管理器自己的命令抢同一个名字。
- **不收 GUI 应用。** 这里只放命令行软件包，Electron / 带 `.app` 的桌面程序（Wave Terminal
  就是这一类）不在收录范围内，所以也不会有它们的 tag。
