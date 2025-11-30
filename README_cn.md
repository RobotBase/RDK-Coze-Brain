# 🤖 地瓜机器人 x 扣子：RDK 部署 Coze Studio 实战指南

**适用硬件**：地瓜机器人 RDK 系列（RDK X3 / X3 Module / RDK X5 / RDK S100）  

-----

## 第一步：配置 Docker 环境（最关键的一步）

由于 RDK 运行在 ARM 架构的 Ubuntu 系统上，且国内网络环境特殊，直接安装 Docker 容易遇到 `iptables` 兼容性问题或镜像拉取失败。请严格按照以下步骤操作：

### 1\. 清理旧版本与冲突包

为了防止环境冲突，首先清理系统可能存在的旧版本 Docker 或冲突组件。

```bash
sudo apt remove --purge containerd.io containerd docker-ce docker.io
sudo apt autoremove
sudo rm -rf /var/lib/docker /var/lib/containerd /etc/docker
```

### 2\. 安装 Docker

推荐使用 Ubuntu 官方源进行安装，稳定性更好。

```bash
sudo apt update
sudo apt install docker.io
```

### 3\. 解决 iptables 兼容性并配置国内镜像源

这是一个**必做**的步骤。RDK 系统通常使用 `nftables`，而 Docker 默认依赖 `iptables-legacy`，会导致启动报错。同时，我们需要配置镜像源以加速下载。

请直接执行以下命令，创建配置文件：

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
    "iptables": false,
    "registry-mirrors": [
        "https://docker.1panel.live/",
        "https://docker.1ms.run",
        "https://docker.xuanyuan.me",
        "https://docker.xpg666.xyz/",
        "https://dytt.online",
        "https://lispy.org",
        "https://docker.xiaogenban1993.com",
        "https://docker-0.unsee.tech",
        "https://666860.xyz",
        "https://docker.m.daocloud.io",
        "https://docker.nju.edu.cn",
        "https://hub.rat.dev"
    ]
}
EOF
```

> **配置说明**：
>
> * `"iptables": false`：解决了 `Could not fetch rule set generation id` 的报错。
> * `"registry-mirrors"`：使用了国内常用的 Docker 镜像加速地址，解决下载卡顿问题。

### 4\. 重启 Docker 并验证

应用配置并检查是否安装成功。

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo docker version  # 应该能看到 Client 和 Server 版本信息
```

### 5\. 安装 Docker Compose

Coze Studio 需要通过 Docker Compose 启动。

```bash
sudo apt install docker-compose-plugin
docker compose version  # 验证安装是否成功
```

-----

## 第二步：下载 Coze Studio 项目代码

环境准备好后，我们将 Coze Studio 的代码克隆到本地。

回到用户主目录

```bash
cd ~
```

克隆代码仓库

```bash
git clone https://github.com/coze-dev/coze-studio
```

-----

## 第三步：启动服务

一切就绪，开始部署！

1. 进入 docker 目录

```
cd ~/coze-studio/docker
```

2. 复制环境变量文件

```
cp .env.example .env
```

3. 启动服务 (使用 docker compose)

```
sudo docker compose -f docker-compose.yml up -d
```

此时你会看到终端显示正在 `Pulling`（下载）各种镜像，请耐心等待下载完成。

-----

## 第四步：注册账号与配置模型

当所有容器都显示 `Started` 后，我们通过浏览器进行后续配置：

1. **注册账号**：
    打开浏览器访问 `http://localhost:8888/sign`（或 `http://<RDK的IP地址>:8888/sign`），输入用户名、密码点击注册按钮。

2. **配置模型**（接入豆包 1.6）：
    登录后，访问 `http://localhost:8888/admin/#model-management`（或通过界面导航至模型管理页面）新增模型。

      * *注意：此功能需要 Coze Studio 镜像版本大于等于 0.5.0。*
      * **配置信息获取**：
          * **API Key**: [点击这里创建/查看](https://www.volcengine.com/docs/82379/1541594)
          * **Endpoint ID (推理接入点)**: [点击这里创建](https://console.volcengine.com/ark/region:ark+cn-beijing/endpoint/create?customModelId=) (创建接入点后，页面红框里显示的字符才是你的 Endpoint ID)

3. **开始使用**：
    配置完成后，访问 Coze Studio 主页 `http://localhost:8888/` 即可开始创建 Agent！🚀

-----

## 附录：常见问题与核心故障排查 (Troubleshooting)

如果在部署过程中遇到容器无法启动、服务无法连接或"Internal Server Error"等问题，请参考以下核心解决方案。

### 1\. 核心诊断命令

遇到问题时，请优先执行以下命令查看报错详情，而不是盲目重装。

* **查看容器运行状态**：

    ```bash
    cd ~/coze-studio/docker
    sudo docker compose ps
    ```

    *正常状态下，所有服务的 Status 应为 `Up (healthy)` 或 `Up`。如果有 `Exit` 或 `Unhealthy`，请继续下一步。*

* **查看关键服务日志（核心！！！）**：
    大部分报错（如 500 错误、连接超时）都藏在日志里。

    ```bash
    # 查看后端服务日志（解决 Internal Server Error）
    sudo docker logs --tail 100 coze-server

    # 查看向量库日志（解决 Milvus 启动失败）
    sudo docker logs --tail 100 coze-milvus
    ```

### 2\. 网络与启动报错 (iptables/DNS)

如果日志中出现 `dial tcp ... connection refused`、`context deadline exceeded` 或 `lookup ... failed`，通常是 RDK 的网络转发问题。

* **检查 1：Docker 配置是否禁用 iptables**
    确保 `/etc/docker/daemon.json` 中必须包含 `"iptables": false`，因为 RDK 内核可能缺失某些防火墙模块。

    ```bash
    sudo cat /etc/docker/daemon.json
    # 必须确认包含: "iptables": false
    ```

* **检查 2：开启 IPv4 转发**
    如果禁用了 iptables，必须手动开启内核转发，否则容器无法联网。

    ```bash
    # 临时开启（立即生效）
    sudo sysctl -w net.ipv4.ip_forward=1

    # 验证
    cat /proc/sys/net/ipv4/ip_forward
    # 输出 1 即为正常
    ```

### 3\. Milvus 向量库启动卡住或报错

Milvus 是最复杂的组件，它依赖 Etcd 和 MinIO。如果它报错 `connect to etcd failed` 或 `check blob bucket exist failed`：

* **原因**：通常是因为依赖服务（MinIO/Etcd）的 IP 地址变动，或者它们启动慢于 Milvus。
* **解决方法**：
    不需要修改配置文件，直接重启 Milvus 容器，让它重新获取依赖服务的 IP：

    ```bash
    cd ~/coze-studio/docker
    sudo docker compose restart milvus
    ```

    等待几秒后，再次查看日志确认出现 `Proxy successfully started`。

### 4\. Coze Studio 页面白屏或 500 错误

* **白屏**：通常是浏览器缓存或端口未放行。请尝试强制刷新（Ctrl+F5）或检查 RDK 的 IP 是否正确。
* **500 Internal Server Error**：
    1. 检查是否已在 Web 界面配置了模型（见第四步）。
    2. 检查 `coze-server` 容器是否连接上了 Milvus（见上方"查看关键服务日志"）。
    3. 尝试重启后端服务：

        ```bash
        sudo docker compose restart coze-server
        ```

### 5\. 极端情况：重置所有环境

如果环境彻底乱了（例如 IP 冲突严重），可以执行以下命令彻底清除并重建（**注意：这会删除所有已创建的 Agent 数据**）：

```bash
cd ~/coze-studio/docker
sudo docker compose down -v  # 删除容器和数据卷
sudo systemctl restart docker # 重启 Docker 守护进程
sudo docker compose up -d    # 重新构建
```
