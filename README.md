# racknerd 部署n8n：1 GB 够不够、一键装还是 Docker 装、一年花多少钱？（附套餐表与 Restic 备份方案）

RackNerd 给 n8n 标的最低配置是 **2 vCPU / 2 GB 内存**，官方 FAQ 还补了一句：生产环境要接 Postgres 和 AI 工具，建议直接选 4 GB 档。安装有两条路：在 `ctrl` 面板里一键装，15–45 分钟出结果，代价是整块盘被重装；或者 SSH 进去用 Docker Compose 加 Caddy 自己搭，多花半小时，换来端口、证书、备份全在自己手里。价格能核到的是 **1 GB 年付 $21.99、2 GB 年付 $35.99**，最终以下单页显示的金额和周期为准。

下面把配置口径、两条安装路径的实际差别、SQLite 换 Postgres 的时机、备份怎么留，一项项讲清楚。

> **选购速览**
> 多数人买 2 GB 那档（$35.99/年，折合每月约 3 美元）。它正好落在官方标注的 2 vCPU / 2 GB 上，还留得下从第一天就跑 PostgreSQL 的余量。只想跑几个轻量工作流、以后升级也认了，1 GB（$21.99/年）能把 n8n 跑起来。
> 👉 [查看 RackNerd 当前套餐与结账价](https://bit.ly/RacKnerd)

## 套餐对比：两档有价格依据，4 GB 现场确认

| 套餐 | 内存 / vCPU | 适合谁 | 价格（年付） | 下单 |
| --- | --- | --- | --- | --- |
| 1 GB KVM VPS | 1 GB / 1 核 | 试水、几个轻量工作流、SQLite 单机 | $21.99/年 | [选 1 GB 套餐](https://bit.ly/RacKnerd) |
| 2 GB KVM VPS | 2 GB / 2 核 | 官方标注的 n8n 起步线，可同机跑 Postgres | $35.99/年 | [选 2 GB 套餐](https://bit.ly/RacKnerd) |
| 4 GB 及以上 | 官方 FAQ 建议的生产档 | 接 Postgres + AI 工具、queue mode 多进程 | 公开报价未核到，以结账页为准 | [去结账页确认 4 GB 档价格](https://bit.ly/RacKnerd) |

两个价格来自第三方整理页，不是官方价目表截图，所以下单前请以购物车金额和计费周期为准。4 GB 档我只写官方 FAQ 的配置建议，不替你猜价格和硬盘容量。

平台层面的信息是明确的：全部为 KVM 虚拟化，含完整 root SSH，存储是 RAID-10 SSD，每档都给 1 Gbps 端口，可选 Debian、Ubuntu、AlmaLinux、Rocky、Fedora 或自定义 ISO，机房分布在多个地区。洛杉矶和法国机房还能提交工单申请最多 100 个免费 IPv6，跑多实例或按客户隔离 n8n 时用得上。

## n8n 到底吃多少内存？两个口径分开看

软件本身很轻：空闲时占用约 320 MB，但跑起真实工作流后实际占用落在 1–2 GB。n8n 的扩容建议里，内存优先级高于 CPU，拿不准时加内存而不是加核。

RackNerd 官方给一键镜像标的 2 vCPU / 2 GB 属于“跑起来之后还有余量”的推荐值，跟软件最低可行配置是两回事，别把两个数混成一个。存储方面，数据库本身占 512 MB 到 4 GB，SSD 的随机读写直接影响 n8n 界面和执行的响应速度，这也是 RAID-10 SSD 比多给几十 GB 空间更值钱的地方。

## 两条部署路径：一键装省事，手动装可控

### ctrl 一键安装：快，但会重装整块盘

下单后进 `ctrl` 控制面板，打开 One-Click Apps，选 n8n，剩下的交给它。安装耗时约 15–45 分钟，完成后 root SSH 和应用登录凭据会显示在面板里。

官方页面写得很直白：一键安装会**永久擦除 VPS 上已有的数据，包括操作系统、应用和文件**，然后部署一套全新镜像。它不是“在现有系统上加装一个软件”，而是整盘重装。新买的空机器无所谓，已经在跑别的服务的机器千万别点。面板里同类的可选应用还有 Portainer、Coolify、Webmin、CyberPanel、WireGuard (wg-easy)、Ollama、LibreChat、AnythingLLM、Hermes Agent、AdminBolt 等，如果以后想把 Docker 管理面板或 WireGuard 装在同一台机器上，可以一次规划好，避免反复重装。

### Docker Compose + Caddy：多花半小时，换可控和可备份

手动方案的核心组件只有三个：n8n 容器、Caddy 反向代理、一个挂到宿主机的数据目录。

镜像用官方的 `docker.n8n.io/n8nio/n8n:latest`，容器监听 5678，数据卷挂到 `/home/node/.n8n`。compose 里几个环境变量值得逐个确认：

yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    restart: always
    environment:
      - N8N_HOST=${N8N_HOST}
      - WEBHOOK_URL=${WEBHOOK_URL}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - DB_SQLITE_POOL_SIZE=5
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_PROXY_HOPS=1
    volumes:
      - /opt/docker-data/n8n/data:/home/node/.n8n


三个细节直接决定后面顺不顺：

**`N8N_PROXY_HOPS=1`** 告诉 n8n 它在一个反向代理后面。漏掉这一行，webhook 回调和登录跳转可能拼出错误的协议或地址，表现是页面能开、但 webhook 收不到或跳转回 http。

**不发布 5678 端口。** 容器只在内部 Docker 网络里暴露 5678，宿主机只让 Caddy 占用 80 和 443。n8n 本身不直接暴露在公网，攻击面小一圈。

**数据集中在 `/opt/docker-data/<服务名>/`。** 所有服务的配置和数据都收在这一个目录下，以后接 Restic 只要指向 `/opt/docker-data`，n8n 的工作流、凭据、设置一次性全覆盖，不用逐个容器去找卷。

Caddy 那边配一行就够：


n8n.yourdomain.com {
    reverse_proxy n8n:5678
}


域名用 Cloudflare 的 A 记录指向 VPS，Caddy 看到 Caddyfile 里的域名会自动申请 Let's Encrypt 证书并负责续期。四种 `.env` 变量里最不能丢的是 `N8N_ENCRYPTION_KEY`，它用来加解密存下来的凭据，换掉或丢失，之前的 API key、数据库密码就解不开了。生成一串长随机值，抄一份到密码管理器里。

首次进后台可以顺手领一个免费 license key，能解锁若干功能，不额外收费。以后升级就是 `docker compose pull` 加重建容器，数据在宿主机目录里不受影响，完事 `docker image prune` 清掉旧镜像。

遇到容器起不来、日志报权限错误，多半是数据目录属主不对：

bash
sudo chown -R $USER:$USER /opt/docker-data/n8n/data


## SQLite 还是 Postgres：分界线不在工作流数量

1 GB 和 2 GB 的真实差别，是能不能从第一天就把 PostgreSQL 放在同一台机器上。SQLite 起步完全够用，单文件、零配置、备份就是拷目录；等工作流变多、并发执行上来，或者你想跑 queue mode，再换 Postgres。

queue mode 是 n8n 的多进程模式：一个主进程负责调度和 UI，若干 worker 进程执行任务，中间靠 Redis 通信。这套组合加上 Postgres，四个服务挤在一台机器上，4 GB 内存才不至于捉襟见肘。有人在树莓派 5（8 GB 内存）上跑过完整的一套：n8n 主进程 + worker、PostgreSQL 16、Redis、Caddy，再加 Tailscale 和监控。这个配置能跑，也说明 queue mode 的重量级。

单机够不够，先看你的工作流是定时触发还是高频 webhook。定时任务为主，1 GB 加 SQLite 能撑很久；webhook 常年有流量、还要接 AI 节点，直接上 2 GB，别在内存上省那 14 美元。

## 上线前值得做完的几件事

买的是 unmanaged KVM，安全配置得自己做。最低限度：面板开两步验证，建一个非 root 的 sudo 用户，SSH 改密钥登录并禁用密码登录和 root 直连，UFW 只放行 SSH、HTTP、HTTPS，装 Fail2Ban 封禁暴力破解 IP，开无人值守安全更新。这些做完再装 Docker。

备份用 Restic 推到 Cloudflare R2，S3 兼容、不收出网费，标准存储每月 10 GB 免费额度对一套 n8n 数据很宽裕。保留策略可以这样：

bash
restic backup /opt/docker-data
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune


恢复时停掉容器，把数据拷回 `/opt/docker-data/n8n`，再启动。因为数据本来就集中在一个目录，这一步不需要回忆每个卷挂在哪。

还有一个值得抄的路由思路：把 n8n 拆成两个入口，管理界面走私有网络（Tailscale 之类的内网域名），公网域名只放 webhook 路径，其余请求一律返回 403。仪表盘不暴露在公网，webhook 照常触发，这套做法在 VPS 上同样成立，比给后台加一层密码更彻底。

## 一年到底花多少钱

社区版在 Fair Code 许可下免费，不限工作流数量、步骤数、执行次数和用户数。你付的只有机器钱：$21.99 一年折合约 **$1.83/月**，$35.99 一年折合约 3 美元/月。

作为参照，n8n Cloud 起步价 $24/月，含 2,500 次执行（这个价格来自第三方页面转述，仅作比价用）。同样的自动化引擎，自托管一年的支出还不到云版两个月的费用。差额的来源也很清楚：云版卖的是托管、升级、可用性，自托管把这些活接了过来，机器重启、证书续期、备份验证都得你自己盯着。

如果你能接受这份运维责任，这笔账很划算；如果你连 SSH 都不想碰，省下的钱会以别的方式还回去。

👉 [按年付价格开通 RackNerd VPS](https://bit.ly/RacKnerd)

## 常见问题

**一键安装会删掉我机器上的东西吗？**

会。官方明确写着永久擦除操作系统、应用和文件，然后部署全新镜像。新机器随便装，有数据的机器先备份或改用 Docker 手动部署。

**1 GB 内存能跑 n8n 吗？**

能跑，空闲占用只要 320 MB 左右。但跑起真实工作流后实际占用会到 1–2 GB，1 GB 的机器没有余量，也不适合同机再跑 Postgres。轻量用途可以，长期用建议 2 GB。

**装完网页打不开，先查什么？**

依次确认三件事：域名 A 记录是否已指向 VPS、Caddy 是否成功申请到证书（看容器日志）、n8n 容器的数据目录属主是否正确。反向代理场景下还要确认 `N8N_PROXY_HOPS=1` 和 `N8N_PROTOCOL=https` 都设了。

**HTTPS 要另外买证书吗？**

不用。Caddy 会自动向 Let's Encrypt 申请并续期，前提是域名已经解析到这台机器、80 和 443 端口在防火墙里放行。

**以后不够用了怎么升级？**

RackNerd 的套餐升级是一次重启的事，不需要迁移数据。数据都在 `/opt/docker-data` 下，跟着系统盘走。

## 最后怎么选

只想试水、跑几个定时同步类工作流，买 **1 GB（$21.99/年）**，SQLite 起步，不够再升。

要接 Postgres、webhook 有稳定流量、打算长期当生产工具用，直接上 **2 GB（$35.99/年）**，这是官方标注的起步线，也是多数人一次买对的选择。

要跑 queue mode、把 worker、Postgres、Redis 放同一台机器，或者同一台机器上还想塞 Ollama、LibreChat 这类吃内存的应用，看 **4 GB 档**，配置和报价在下单页现场确认。

下单前确认两件事：购物车里的最终金额与计费周期（年付价格会随活动变动），以及这笔钱的性质。RackNerd 官方知识库的退款条目写明不提供 money back guarantee，请按不可退的年度支出做预算。

👉 [前往 RackNerd 选择适合跑 n8n 的套餐](https://bit.ly/RacKnerd)
