# MacWaveOrg/Bottle

MacWave 4.0 的预编译二进制存放处。以前每条索引记录只有一个指向上游的 `url`，用户直接从
别人的服务器下载；4.0 改成先由 CI 取下来放进这里，用户只从我们这里取。

这样做解决三件事（这也是 4.0 的全部动机）：

1. 上游被攻破或出现漏洞时，不会直接波及用户 —— 下发的字节是我们验证过的那些；
2. 上游崩溃、删 release、换 CDN 时，用户不会因此装不上；
3. 用户装机量大了也不会给上游造成压力。

## 存放形态：一个包一个 Release

**资产放在 Release 里，不放在仓库文件里。** 原因是硬限制：git 单文件超过 100 MiB 会被直接
拒绝推送，而 `ollama-darwin.tgz` 已经是 **172.9 MB**。Release 的单文件上限是 2 GiB，官方也
明确不限制总量与下载带宽。

```
tag   = 包名
资产名 = <包名>@<版本>.<架构><扩展名>
```

客户端拼出来的地址：

```
https://github.com/MacWaveOrg/Bottle/releases/download/jq/jq%401.7.1.arm64
                                                     └┬┘ └──────┬──────┘
                                                     tag      资产名
```

为什么是这个形状 —— 只有一条标准：**国内镜像站要能用 rsync + nginx 原样提供**。清华、
中科大、阿里镜像的都是普通文件树，没有一个能模拟容器 registry 那套按 digest 取字节的接口。
而 GitHub 的下载路径恰好是 `/<tag>/<资产>`，所以镜像站只要保留域名之后那段相对路径：

```
默认：https://github.com/MacWaveOrg/Bottle/releases/download/jq/jq%401.7.1.arm64
镜像：https://mirror.example.cn/MacWave/bottle/jq/jq%401.7.1.arm64
                                  ↑ 域名之后完全相同
```

镜像路径里的 `MacWave/` 不是随意的：一个镜像域名要能被多个产品共用，LinuxWave 将来会放在
同一台机器的 `/LinuxWave`，谁也不独占域名根。所以镜像站的目录结构与 `MACWAVE_BOTTLE_DOMAIN`
的值都带这一层。

换镜像就是换一个环境变量（`MACWAVE_BOTTLE_DOMAIN`），镜像站不需要理解任何协议。

## 三条不可违反的规则

**一、只增不删。** 旧版本必须永远可下载。多版本共存是 MacWave 的设计前提 —— 删掉一个资产，
就等于让 `wave install <包>@<旧版本>` 永久失效。

**二、同名不覆盖。** 资产一旦发布就不再改动。需要重建同一版本时**换版本后缀**，例如
`1.7.1-1`（排序上大于 `1.7.1`，客户端的版本解析器已正确处理）。

**三、字节原样搬运，不重打包。** 放进来的是上游那份文件，一个字节都不改。因此：

- 校验值与上游公布的一致，索引里可以直接填官方 SHA256，不必自己下载再算；
- 不存在「我们的副本与上游不同」这种只能靠信任解释的问题。

代价是扩展名不统一（`zip` / `tar.gz` / `tar.bz2` / `tar.xz` / `conda` / 裸可执行文件都有）。
所以客户端从索引的 `file` 字段（即上游文件名）取扩展名，只改文件名不改内容；`pkgunzip.sh`
早就按扩展名支持这些格式，4.0 一行都没加。

## 架构写在文件名里

`arm64` 与 `amd64` 是两条同名同版本的索引记录，所以架构必须进文件名，否则会撞名：

```
jq@1.7.1.arm64
jq@1.7.1.amd64
```

## 与索引的关系

`infosource` 的版本记录仍然保留 `url` 字段，但 **4.0 不读它** —— 它只用于
`--install-from-source`（绕开这里直接回上游装）和给人看的上游线索。所以：

- 加一个包：先写 `infosource` 的记录（含 `file` 与 `sha256`），再由 CI 搬字节到这里；
- `sha256` 是这份文件的校验值，通常就等于上游公布的值。

## 谁往里放东西

`MacWaveOrg/MacWave` 的 `.github/workflows/sync-payloads.yml`，跑在 GitHub 托管的 Runner 上 ——
这是唯一能稳定访问上游的地方（从大陆访问 `github.com` 实测三次里两次连 TCP 都建不起来）。

它调用两个脚本：

- `scripts/sync_payloads.py` —— 抓上游、按索引里的 sha256 校验、按上述规则命名；
- `scripts/publish_payloads.py` —— 建 Release（tag = 包名）并上传，同名同大小跳过。

手动跑一次单包验证：Actions → Sync payloads → Run workflow，`only` 填 `jq`。

## 镜像站怎么同步

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
