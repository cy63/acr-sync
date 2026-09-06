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

| 名称 | 值 | 说明 |
|---|---|---|
| `ACR_REGISTRY` | `crpi-xxxxxxxxxxxx.cn-shanghai.personal.cr.aliyuncs.com` | **公网地址**，不带 `-vpc` —— runner 在阿里云外面，走不了内网 |
| `ACR_NAMESPACE` | `你的命名空间` | |
| `ACR_REPO` | `你的仓库名` | 所有镜像共用这一个仓库 |

> ⚠️ **为什么所有镜像挤在一个仓库里**：ACR 个人版的仓库数量有上限（本例是 3 个），
> 不够一个镜像一个仓库。所以改成**单仓库 + tag 区分**：
> `mysql:8.4` 同步后是 `<仓库>:mysql-8.4`。

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

tag 自动推导：**去掉 registry / org 前缀，冒号换成短横线**。

| 写法 | 同步后的地址 |
|---|---|
| `mysql:8.4` | `<ACR>/<NS>/<REPO>:mysql-8.4` |
| `nginx` | `<ACR>/<NS>/<REPO>:nginx-latest` |
| `rancher/local-path-provisioner:v0.0.37` | `<ACR>/<NS>/<REPO>:local-path-provisioner-v0.0.37` |
| `registry.k8s.io/coredns/coredns:v1.14.2` | `<ACR>/<NS>/<REPO>:coredns-v1.14.2` |

需要自己定 tag 时写第二列：

```
registry.k8s.io/metrics-server/metrics-server:v0.7.2   metrics-server-v0.7.2
```

> ⚠️ **代价**：镜像原本的版本号被塞进了 tag 里，同一镜像的多个版本会并列成
> `mysql-8.4`、`mysql-8.0`……看着不如 Docker Hub 的结构清楚。
> 这是仓库数量受限下的妥协，不是推荐做法。

### 单个：手动触发

Actions 页面 → 左侧 `sync-images` → **Run workflow** →
在输入框填镜像名（如 `mysql:8.4`）→ Run。

不改文件、不 push，适合临时需要一个镜像的场景。

### 跑完看结果

Actions 的运行页面底部有个 **Summary** 表格，列出每个镜像的同步结果
和家里节点该用的拉取地址。

## 五、家里节点拉取

```bash
sudo crictl pull crpi-xxxxxxxxxxxx.cn-shanghai.personal.cr.aliyuncs.com/你的命名空间/你的仓库:mysql-8.4
```

k8s 里直接写这个地址：

```yaml
spec:
  containers:
    - name: app
      image: crpi-xxxxxxxxxxxx.cn-shanghai.personal.cr.aliyuncs.com/你的命名空间/你的仓库:mysql-8.4
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

命名空间和仓库数量都有上限（实测仓库上限为 3）。这就是本流水线采用
**单仓库 + tag 区分** 的原因。tag 数量没有限制，够用。
