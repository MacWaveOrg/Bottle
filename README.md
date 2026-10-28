# MacWaveOrg/Bottle

MacWave 4.0 的预编译二进制存放处。以前每条索引记录只有一个指向上游的 `url`，
用户直接从别人的服务器下载；4.0 改成先由 CI 抓下来放进这里，用户只从我们这里取。

这样做解决三件事（这也是 4.0 的全部动机）：

1. 上游被攻破或出现漏洞时，不会直接波及用户——下发的字节是我们验证过的那些；
2. 上游崩溃、删 release、换 CDN 时，用户不会因此装不上；
3. 用户装机量大了也不会给上游造成压力。

## 路径布局

客户端拼地址的方式只有一处，在 MacWave 仓库的 `lib/domains.py`：

```python
def bottle_url(package_name, package_version, arch, filename):
    return f"{require_domain('bottle')}/{package_name}/{package_version}/{arch}/{filename}"
```

也就是：

```
<MACWAVE_BOTTLE_DOMAIN>/<包名>/<版本>/<架构>/<文件名>
```

例如：

```
bottles/jq/1.7.1/arm64/jq-macos-arm64                 →  <域名>/jq/1.7.1/arm64/jq-macos-arm64
bottles/ollama/0.12.3/amd64/ollama-darwin.tgz        →  <域名>/ollama/0.12.3/amd64/ollama-darwin.tgz
```

仓库里保留一层 `bottles/` 只是为了在 GitHub 的网页上好认，域名要指到它下面。

**文件名必须以索引为准。** 索引记录（`infosource`）里有一条 `file:` 字段，客户端拿它当
文件名，也靠它的扩展名决定用 `zip` 还是 `tar.gz` 解压——所以这里放的文件名必须与索引
里写的一字不差，改名就等于换包。

## 与索引的关系

`infosource` 的那条记录仍然保留 `url` 字段，但 **4.0 不读它**——它只用于
`--install-from-source`（绕开这里直接回上游装）和给人看的上游线索。所以：

- 加一个包：先写 `infosource` 的记录（含 `file` 与 `sha256`），再把对应的字节放进来；
- `sha256` 必须是这里那份文件的校验值，不是上游的。

## 一个待定的问题：GHCR

手稿里写的是「GHCR，具体方案略」。这里要先说清楚两者的差别，因为它决定
`bottle_url` 怎么写：

- **普通文件路径**（本仓库 + CDN / Pages / Release 资产 / 任意对象存储）：上面那行
  拼字符串就够用，`MACWAVE_BOTTLE_DOMAIN` 指到一个前缀即可。
- **GHCR（OCI registry）**：它不是按路径取文件，而是 `/v2/<org>/<repo>/manifests/<tag>`
  与 `/v2/<org>/<repo>/blobs/sha256:<digest>` 两套接口。这时 `bottle_url` 不能只拼路径，
  得先取 manifest 再取 blob，客户端也要多一步。

所以：**在定下 GHCR 之前，`bottles/` 这套路径是当前代码唯一能用的形态。**
如果最后选 GHCR，要改的只有 `lib/domains.py` 里的 `bottle_url` 与下载那一步，
索引格式与文件名规则不变。这条留给 4.0 落地时决定，当前代码按普通路径写。

## 目录

```
bottles/
    <包名>/
        <版本>/
            <架构>/           arm64 或 amd64
                <文件名>      与索引 file 字段完全一致
```

空目录不会被 Git 保存，所以下面用 `.gitkeep` 占位；真正的文件由 CI 放进来。
