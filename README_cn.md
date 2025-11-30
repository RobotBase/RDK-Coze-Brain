# 🤖 地瓜机器人 x 扣子：RDK 部署 Coze Studio 实战指南

**适用硬件**：地瓜机器人 RDK 系列（RDK X3 / X3 Module / RDK X5 / RDK S100）  

-----

## 第一步：配置 Docker 环境（最关键的一步）

由于 RDK 运行在 ARM 架构的 Ubuntu 系统上，且国内网络环境特殊，直接安装 Docker 容易遇到 `iptables` 兼容性问题或镜像拉取失败。请严格按照以下步骤操作：

### 1\. 清理旧版本与冲突包

为了防止环境冲突，首先清理系统可能存在的旧版本 Docker 或冲突组件。

sudo apt remove --purge containerd.io containerd docker-ce docker.io
sudo apt autoremove
sudo rm -rf /var/lib/docker /var/lib/containerd /etc/docker

### 2\. 安装 Docker

推荐使用 Ubuntu 官方源进行安装，稳定性更好。

sudo apt update
sudo apt install docker.io

### 3\. 解决 iptables 兼容性并配置国内镜像源

这是一个**必做**的步骤。RDK 系统通常使用 `nftables`，而 Docker 默认依赖 `iptables-legacy`，会导致启动报错。同时，我们需要配置镜像源以加速下载。

请直接执行以下命令，创建配置文件：

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
    "iptables": false,
    "registry-mirrors": [
        "https://dockerproxy.com",
        "https://docker.m.daocloud.io",
        "https://hub-mirror.c.163.com",
        "https://mirror.baidubce.com",
        "https://docker.nju.edu.cn",
        "https://registry.docker-cn.com"
    ]
}
EOF

> **配置说明**：
>
> * `"iptables": false`：解决了 `Could not fetch rule set generation id` 的报错。
> * `"registry-mirrors"`：使用了国内常用的 Docker 镜像加速地址，解决下载卡顿问题。

### 4\. 重启 Docker 并验证

应用配置并检查是否安装成功。

sudo systemctl daemon-reload
sudo systemctl restart docker
sudo docker version  # 应该能看到 Client 和 Server 版本信息

### 5\. 安装 Docker Compose

Coze Studio 需要通过 Docker Compose 启动。

sudo apt install docker-compose-plugin
docker compose version  # 验证安装是否成功

-----

## 第二步：下载 Coze Studio 项目代码

环境准备好后，我们将 Coze Studio 的代码克隆到本地。

# 回到用户主目录

cd ~

# 克隆代码仓库

git clone <https://github.com/coze-dev/coze-studio>

-----

## 第三步：配置大模型（接入豆包 1.6）

Coze Studio 只是一个平台，正式启动前，我们必须为它配置“大脑”（大模型）。这里我们以**火山引擎的豆包模型（Doubao-1.6）**为例。

### 1\. 准备配置文件

进入项目目录，复制一份模板文件：

cd coze-studio
cp backend/conf/model/template/model_template_ark_doubao-seed-1.6.yaml backend/conf/model/ark_doubao-seed-1.6.yaml

### 2\. 获取 API Key 和 Endpoint ID

你需要前往火山引擎控制台获取以下两个关键信息：

* **API Key**: [点击这里创建/查看](https://www.volcengine.com/docs/82379/1541594)
* **Endpoint ID (推理接入点)**: [点击这里创建](https://console.volcengine.com/ark/region:ark+cn-beijing/endpoint/create?customModelId=)
  * *注意：创建接入点后，页面红框里显示的字符才是你的 Endpoint ID。*

### 3\. 修改配置文件

使用编辑器（如 vim 或 nano）修改 `backend/conf/model/ark_doubao-seed-1.6.yaml`，仅需修改以下 3 个参数：

id: 123                          # 必须是非 0 整数，全局唯一即可
meta:
  conn_config:
    api_key: "你的火山引擎API_Key"    # 填入上面获取的 API Key
    model: "你的Endpoint_ID"         # 填入上面获取的 Endpoint ID

-----

## 第四步：启动服务

一切就绪，开始部署！

# 1. 进入 docker 目录

cd ~/coze-studio/docker

# 2. 复制环境变量文件

cp .env.example .env

# 3. 启动服务 (使用 docker compose)

sudo docker compose -f docker-compose.yml up -d

此时你会看到终端显示正在 `Pulling`（下载）各种镜像，请耐心等待下载完成。

-----

## 第五步：开始使用

当所有容器都显示 `Started` 后：

1. 打开浏览器，访问 RDK 的 IP 地址或本地地址：`http://localhost:8888`
2. 输入默认的邮箱和密码进行登录。
3. 现在，你已经成功在地瓜 RDK 上部署了 Coze Studio！🚀

-----
