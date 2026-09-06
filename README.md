# ACR 镜像同步流水线

用 GitHub Actions 把任意公共镜像同步到自己的阿里云 ACR，
**补齐阿里云镜像加速器覆盖不到的部分**。

> 本仓库是家庭实验室镜像加速方案的一部分。真实的 ACR 地址和命名空间
> 通过 GitHub Variables 注入，不写在文件里。

## 为什么需要它

阿里云的镜像加速器只同步了 Docker Hub 的一个子集。实测 `mysql:8.4`、
`eclipse-temurin:21-jdk` 都返回 `not found` —— 这不是网络问题，
是阿里云那边就没有，靠调配置绕不过去。

GitHub 的 runner 直连 Docker Hub，没有覆盖问题，也不经过自己的任何机器：

```
仓库 push  →  Actions runner  →  skopeo copy  →  你的 ACR
                                                   │
                                      家里节点 ─────┘
```

## 一、建仓库

把本目录的两个文件放到一个新的 GitHub 仓库根目录：

```
你的仓库/
├── .github/workflows/sync-images.yml
└── images.txt
```

> **仓库可以设为 public。** 敏感信息全部走 Secrets / Variables，
> 文件里不含任何地址和凭证。public 仓库的 Actions 分钟数无限制，
> private 仓库每月只有 2000 分钟免费额度。

## 二、配置 Variables（非敏感，明文）

仓库 → Settings → Secrets and variables → Actions → **Variables** 标签 → New variable

| 名称 | 值 |
|---|---|
| `ACR_REGISTRY` | `crpi-xxxxxxxxxxxx.cn-shanghai.personal.cr.aliyuncs.com` |
| `ACR_NAMESPACE` | `你的命名空间` |

> 用**公网地址**（不带 `-vpc`）—— GitHub 的 runner 在阿里云外面，走不了内网。

## 三、配置 Secrets（敏感，加密存储）

同一页面 → **Secrets** 标签 → New repository secret

| 名称 | 值 | 必需 |
|---|---|---|
| `ACR_USERNAME` | ACR 访问凭证用户名，建议用 RAM 只读子账号 | ✅ |
| `ACR_PASSWORD` | ACR 访问凭证的固定密码 | ✅ |
| `DOCKERHUB_USERNAME` | Docker Hub 用户名 | 建议 |
| `DOCKERHUB_TOKEN` | Docker Hub Access Token（不是登录密码） | 建议 |

**为什么建议配 Docker Hub 凭证**：匿名拉取限额是 100 次 / 6 小时 / IP，
GitHub runner 的出口 IP 是所有人共享的，很容易撞上限额并报
`toomanyrequests`。登录后提到 200 次 / 6 小时。

> Docker Hub Token 在 hub.docker.com → Account Settings → Personal access tokens 生成，
> 权限选 **Public Repo Read-only** 就够。

⚠️ **ACR 凭证必须能写**（要 push），所以只读的 `AliyunContainerRegistryReadOnlyAccess`
在这里不够用。建议单独建一个只有 ACR 读写权限的 RAM 子账号，
和节点上用的只读子账号分开。

## 四、用法

### 批量：改 images.txt

```
mysql:8.4
redis:7
nacos/nacos-server:v2.4.3
```

push 上去，Actions 自动跑。目标名默认是源镜像去掉 registry 和 org 前缀：

| 写法 | 同步后的地址 |
|---|---|
| `nginx:latest` | `<ACR>/<NS>/nginx:latest` |
| `rancher/local-path-provisioner:v0.0.37` | `<ACR>/<NS>/local-path-provisioner:v0.0.37` |
| `registry.k8s.io/coredns/coredns:v1.14.2` | `<ACR>/<NS>/coredns:v1.14.2` |

需要改名时写第二列：

```
registry.k8s.io/metrics-server/metrics-server:v0.7.2   metrics-server:v0.7.2
```

### 单个：手动触发

Actions 页面 → 左侧 `sync-images` → **Run workflow** →
在输入框填镜像名（如 `mysql:8.4`）→ Run。

不改文件、不 push，适合临时需要一个镜像的场景。

### 跑完看结果

Actions 的运行页面底部有个 **Summary** 表格，列出每个镜像的同步结果
和家里节点该用的拉取地址。

## 五、家里节点拉取

```bash
sudo crictl pull crpi-xxxxxxxxxxxx.cn-shanghai.personal.cr.aliyuncs.com/你的命名空间/mysql:8.4
```

k8s 里直接写这个地址：

```yaml
spec:
  containers:
    - name: app
      image: crpi-xxxxxxxxxxxx.cn-shanghai.personal.cr.aliyuncs.com/你的命名空间/mysql:8.4
```

> 私有仓库需要 `imagePullSecrets`，配置见本地文档第 6 章。
> **Secret 是命名空间隔离的**，每个新命名空间都要重建一次。

## 六、几个要知道的事

**① 推送方向仍然跨境，会慢**

GitHub 的 runner 不可能在国内，push 到阿里云上海要跨境。单个大镜像可能要几分钟到十几分钟。

**但这不重要** —— 整个过程是异步的，你不用等它。提交完 `images.txt` 就可以去做别的，
跑完了家里直接高速拉。这和「拉镜像时人在那儿干等」是完全不同的体验。

**② 加速器方案仍然保留**

加速器有的镜像（`alpine`、`mysql:8`、`nginx` 等），用 ECS 上的 `sync-image` 脚本更快 ——
全程在阿里云内部，几十秒完事。

**两套并存**：加速器有的走 ECS，没有的走 Actions。

**③ 重复同步是安全的**

skopeo 会比对 digest，已存在的层直接跳过。images.txt 里的老镜像每次都会被检查一遍，
但不会重复传输数据。

**④ 上游镜像更新了怎么办**

同一个 tag 上游更新后，重跑一次 Actions 就会同步新版本（digest 变了，skopeo 会重新传）。
可以在 workflow 里加 `schedule` 定时跑，但**不建议** —— 测试环境的镜像版本
最好是固定的，自动更新会让「昨天还好好的」变成难以排查的问题。

**⑤ ACR 个人版额度**

3 个命名空间、300 个仓库。一个镜像一个仓库的话，300 个足够家庭实验室用很久。
