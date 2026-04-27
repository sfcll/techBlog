# Raspberry Pi 5 使用 Docker 搭建个人服务器环境 Docker / Portainer / Nginx / acme.sh SSL / Tomcat / MariaDB / PostgreSQL / WireGuard VPN
1. 安装 Docker
    1.1 更新系统
        sudo apt update
        sudo apt upgrade -y
    1.2 安装 Docker
        curl -fsSL https://get.docker.com -o get-docker.sh
        sudo sh get-docker.sh
    1.3 确认：
        docker --version
    1.4 当前用户加入 docker 组
        sudo usermod -aG docker $USER
        sudo reboot

2. 创建 Docker 网络
    所有主要服务放在同一个 Docker 网络中。
    docker network create sfmed-network
    docker network ls

3. 安装 Portainer
    3.1 创建 Portainer 数据卷
        docker volume create portainer_data
    3.2 启动 Portainer
        docker run -d \
        --name portainer \
        --network sfmed-network \
        -p 9000:9000 \
        -p 9443:9443 \
        -v /var/run/docker.sock:/var/run/docker.sock \
        -v portainer_data:/data \
        --restart unless-stopped \
        portainer/portainer-ce:latest
    访问：
        http://树莓派IP:9000
    设置管理员账户。
    后续如果通过 Nginx 代理 Portainer，可以重建 Portainer 去掉 -p 9000:9000 和 -p 9443:9443。

4. 准备 Nginx 目录
    mkdir -p ~/docker/nginx/conf.d
    mkdir -p ~/docker/nginx/html
    mkdir -p ~/docker/nginx/certs

    创建测试页面：
    echo "Nginx OK" > ~/docker/nginx/html/index.html

5. 申请通配符 SSL 证书
    这里使用 acme.sh + 阿里云 DNS API
    证书覆盖： xxx.com *.xxx.com

    5.1 安装 acme.sh
        curl https://get.acme.sh | sh
        source ~/.bashrc

        如果默认 CA 是 ZeroSSL，建议切换到 Let’s Encrypt：
        acme.sh --set-default-ca --server letsencrypt

        注册账户：
        acme.sh --register-account -m 你的邮箱@example.com --server letsencrypt
    
    5.2 设置阿里云 AccessKey
        export Ali_Key="你的AccessKey"
        export Ali_Secret="你的AccessKeySecret"

        建议写入：
        nano ~/.bashrc

        追加：
        export Ali_Key="你的AccessKey"
        export Ali_Secret="你的AccessKeySecret"

        生效：
        source ~/.bashrc

    5.3 申请证书
        acme.sh --issue \
        --dns dns_ali \
        -d sf-med.cn \
        -d *.sf-med.cn \
        --server letsencrypt
        成功后会看到证书签发成功。

    5.4 安装证书到 Nginx 目录
        如果权限不足，先执行：
        sudo chown -R $USER:$USER ~/docker/nginx/certs
        安装证书：
        acme.sh --install-cert -d sf-med.cn \
        --key-file       ~/docker/nginx/certs/privkey.pem \
        --fullchain-file ~/docker/nginx/certs/fullchain.pem \
        --reloadcmd     "docker exec nginx nginx -s reload"
        说明：
        privkey.pem    私钥
        fullchain.pem  完整证书链

        acme.sh 会自动创建续期任务，到期前自动续期，并执行 Nginx reload。

6. 配置 Nginx
    nano ~/docker/nginx/conf.d/default.conf

7. 启动 Nginx 容器
    docker run -d \
    --name nginx \
    --network sfmed-network \
    -p 80:80 \
    -p 443:443 \
    -v ~/docker/nginx/conf.d:/etc/nginx/conf.d \
    -v ~/docker/nginx/html:/usr/share/nginx/html \
    -v ~/docker/nginx/certs:/etc/nginx/certs \
    --restart unless-stopped \
    nginx:latest

8. 部署 Tomcat
    8.1 创建目录
        mkdir -p ~/docker/tomcat/webapps
        mkdir -p ~/docker/tomcat/logs

        把 WAR 文件放到：
        ~/docker/tomcat/webapps/

    8.2 启动 Tomcat
        docker run -d \
        --name tomcat \
        --network sfmed-network \
        -v ~/docker/tomcat/webapps:/usr/local/tomcat/webapps \
        -v ~/docker/tomcat/logs:/usr/local/tomcat/logs \
        --restart unless-stopped \
        tomcat:9.0-jdk17-temurin

9. 部署 MariaDB
    9.1 创建目录
        mkdir -p ~/docker/mariadb/data
        mkdir -p ~/docker/mariadb/conf

        如果权限报错：
        sudo chown -R $USER:$USER ~/docker/mariadb

    9.2 创建 MariaDB 配置
        nano ~/docker/mariadb/conf/my.cnf

        [mysqld]
        bind-address = 0.0.0.0
        character-set-server = utf8mb4
        collation-server = utf8mb4_unicode_ci

        [client]
        default-character-set = utf8mb4
    
    9.3 启动 MariaDB
        这里开放 3306，但后面用防火墙限制只允许 VPN 访问。
        docker run -d \
        --name mariadb \
        --network sfmed-network \
        -p 3306:3306 \
        -e MARIADB_ROOT_PASSWORD=你的MariaDB密码 \
        -e MARIADB_DATABASE=appdb \
        -e MARIADB_USER=appuser \
        -e MARIADB_PASSWORD=你的MariaDB密码 \
        -v ~/docker/mariadb/data:/var/lib/mysql \
        -v ~/docker/mariadb/conf:/etc/mysql/conf.d \
        --restart unless-stopped \
        mariadb:10.11

        进入 MariaDB：
        docker exec -it mariadb mariadb -u root -p

        查看：
        SHOW DATABASES;
        SHOW VARIABLES LIKE 'bind_address';

11. 部署 PostgreSQL
    11.1 创建目录
        mkdir -p ~/docker/postgres/data

    11.2 启动 PostgreSQL
        docker run -d \
        --name postgres \
        --network sfmed-network \
        -p 5432:5432 \
        -e POSTGRES_USER=postgres \
        -e POSTGRES_PASSWORD=你的Postgres密码 \
        -e POSTGRES_DB=appdb \
        -v ~/docker/postgres/data:/var/lib/postgresql/data \
        --restart unless-stopped \
        postgres:15 \
        -c listen_addresses='*'

12. 配置 PostgreSQL 允许 VPN 网段访问
    编辑：
    nano ~/docker/postgres/data/pg_hba.conf
    最后追加：
    host all all 10.13.13.0/24 md5
    编辑：
    nano ~/docker/postgres/data/postgresql.conf
    确认：
    listen_addresses = '*'
    重启：
    docker restart postgres
    测试：
    docker exec -it postgres psql -U postgres

13. 部署 WireGuard VPN
    13.1 创建目录
        mkdir -p ~/docker/wireguard

    13.2 启动 WireGuard
        docker run -d \
        --name wireguard \
        --network host \
        -e PUID=1000 \
        -e PGID=1000 \
        -e TZ=Asia/Tokyo \
        -e SERVERURL=vpn.xxx.com \
        -e SERVERPORT=51820 \
        -e PEERS=1 \
        -e PEERDNS=1.1.1.1 \
        -e INTERNAL_SUBNET=10.13.13.0 \
        -e ALLOWEDIPS=0.0.0.0/0 \
        -v ~/docker/wireguard:/config \
        -p 51820:51820/udp \
        --cap-add=NET_ADMIN \
        --cap-add=SYS_MODULE \
        --restart unless-stopped \
        lscr.io/linuxserver/wireguard

    13.3 路由器端口转发
        UDP 51820 → 树莓派内网 IP

14. 开启 Raspberry Pi IP 转发
    如果没有 /etc/sysctl.conf，推荐使用 /etc/sysctl.d/
    sudo nano /etc/sysctl.d/99-sysctl.conf
    写入：
    net.ipv4.ip_forward=1
    应用：
    sudo sysctl --system
    确认：
    cat /proc/sys/net/ipv4/ip_forward
    返回：
    1

15. 配置 NAT，让 VPN 可以上网
    确认树莓派使用的出口网卡。
    ip a
    一般有线是：
    eth0
    添加 NAT：
    sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    安装持久化工具：
    sudo apt install iptables-persistent -y
    保存：
    sudo netfilter-persistent save

16. 获取 WireGuard 客户端配置
    查看配置：
    cat ~/docker/wireguard/peer1/peer1.conf
    
    确认里面有：
    Endpoint = vpn.xxx.com:51820
    AllowedIPs = 0.0.0.0/0
    DNS = 1.1.1.1

    导入到：
    Windows WireGuard
    iPhone WireGuard
    Android WireGuard
    Mac WireGuard

17. 防火墙：只允许 VPN 访问数据库
    现在数据库端口已经映射到宿主机：
    3306 MariaDB
    5432 PostgreSQL

    为了不让公网访问数据库，只允许 VPN 网段访问。
    VPN 网段：
    10.13.13.0/24

    17.1 MariaDB 只允许 VPN
        sudo iptables -A INPUT -p tcp --dport 3306 -s 10.13.13.0/24 -j ACCEPT
        sudo iptables -A INPUT -p tcp --dport 3306 -j DROP

    17.2 PostgreSQL 只允许 VPN
        sudo iptables -A INPUT -p tcp --dport 5432 -s 10.13.13.0/24 -j ACCEPT
        sudo iptables -A INPUT -p tcp --dport 5432 -j DROP

        保存：
        sudo netfilter-persistent save

        查看规则：
        sudo iptables -L -n --line-numbers
        sudo iptables -t nat -L -n --line-numbers

        注意不要删除：
        MASQUERADE -> eth0 这是 VPN FQ需要的。

18. 备份建议
    建议定期备份以下目录：

    ~/docker/nginx
    ~/docker/tomcat
    ~/docker/mariadb
    ~/docker/postgres
    ~/docker/wireguard
