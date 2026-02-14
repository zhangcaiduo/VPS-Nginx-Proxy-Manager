#!/bin/bash
# ================================================================
# “    VPS 包工头面板 Nginx Proxy Manager (文科生专用版)    ” 
#    核心架构：Docker + NPM (反向代理) + he.net (DDNS)
#    特色：可视化管理，端口分离，双重翻墙保险
#    作者：ZhangYang & Gemini
# ================================================================

STATE_FILE="/root/.vps_setup_credentials"
DDNS_SCRIPT="/usr/local/bin/he_net_ddns.sh"

# --- 颜色定义 ---
GREEN='\033[0;32m'
BLUE='\033[0;34m'
RED='\033[0;31m'
YELLOW='\033[0;33m'
CYAN='\033[0;36m'
NC='\033[0m'

# --- 基础检查 ---
check_root() {
    [[ $EUID -ne 0 ]] && echo -e "${RED}请使用 sudo 或 root 运行此脚本！${NC}" && exit 1
}

ensure_docker() {
    if ! command -v docker &> /dev/null; then
        echo -e "${YELLOW}正在安装 Docker 环境...${NC}"
        curl -fsSL https://get.docker.com -o get-docker.sh
        sh get-docker.sh && rm get-docker.sh
        systemctl enable --now docker
        echo -e "${GREEN}Docker 安装完成！${NC}"
    fi
    if ! docker compose version &> /dev/null; then
        apt-get update && apt-get install -y docker-compose-plugin
    fi
}

# --- 核心组件安装 ---

# 1. 配置 he.net DDNS (你的动态域名管家)
setup_he_ddns() {
    clear
    echo -e "${BLUE}--- 配置 he.net 动态域名 (DDNS) ---${NC}"
    echo -e "${YELLOW}请确保你已经在 dns.he.net 添加了 A 记录，并开启了动态更新功能。${NC}"
    read -p "请输入你的域名 (例如 home.zhangyang.com): " DDNS_HOST
    read -p "请输入 he.net 提供的 DDNS Key (不是登录密码): " DDNS_KEY
    
    if [[ -z "$DDNS_HOST" || -z "$DDNS_KEY" ]]; then
        echo -e "${RED}信息不能为空！${NC}"; sleep 2; return
    fi

    cat > $DDNS_SCRIPT <<EOF
#!/bin/bash
# He.net DDNS Update Script
curl -4 "https://dyn.dns.he.net/nic/update?hostname=${DDNS_HOST}&password=${DDNS_KEY}" >> /var/log/he_ddns.log 2>&1
EOF
    chmod +x $DDNS_SCRIPT
    
    # 添加到定时任务 (每5分钟执行一次)
    (crontab -l 2>/dev/null | grep -v "$DDNS_SCRIPT"; echo "*/5 * * * * $DDNS_SCRIPT") | crontab -
    
    echo -e "${GREEN}✅ DDNS 配置成功！每5分钟自动汇报一次 IP。${NC}"
    echo -e "${GREEN}正在立即执行一次更新...${NC}"
    bash $DDNS_SCRIPT
    read -n 1 -s -r -p "按任意键继续..."
}

# 2. 安装 Nginx Proxy Manager (大门)
install_npm() {
    ensure_docker
    mkdir -p /root/npm_data
    cat > /root/npm_data/docker-compose.yml <<EOF
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
EOF
    cd /root/npm_data && docker compose up -d
    echo -e "${GREEN}✅ Nginx Proxy Manager 已启动！${NC}"
    echo -e "${CYAN}管理后台: http://$(curl -s4 ifconfig.me):81${NC}"
    echo -e "${CYAN}默认邮箱: admin@example.com${NC}"
    echo -e "${CYAN}默认密码: changeme${NC}"
    echo -e "${YELLOW}请立即登录并修改密码！以后所有应用的域名绑定都在这里进行。${NC}"
    read -n 1 -s -r -p "按任意键继续..."
}

# --- 应用部署 (房间) ---

# 3. Nextcloud (网盘)
install_nextcloud() {
    ensure_docker
    mkdir -p /root/nextcloud_data
    # 使用 SQLite 简化部署，对于个人使用足够，且不需要维护单独的数据库容器
    cat > /root/nextcloud_data/docker-compose.yml <<EOF
services:
  app:
    image: nextcloud:latest
    restart: unless-stopped
    ports:
      - '8888:80'  # 内部端口 8888
    volumes:
      - ./html:/var/www/html
EOF
    cd /root/nextcloud_data && docker compose up -d
    echo -e "${GREEN}✅ Nextcloud 已启动！${NC}"
    echo -e "${YELLOW}请去 NPM 后台，添加反向代理：${NC}"
    echo -e "   域名: pan.你的域名.com"
    echo -e "   转发IP: 172.17.0.1 (Docker网关IP)"
    echo -e "   转发端口: 8888"
    read -n 1 -s -r -p "按任意键继续..."
}

# 4. WordPress (博客)
install_wordpress() {
    ensure_docker
    mkdir -p /root/wordpress_data
    cat > /root/wordpress_data/docker-compose.yml <<EOF
services:
  db:
    image: mariadb:10.6
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: password
    volumes:
      - ./db_data:/var/lib/mysql
  wordpress:
    image: wordpress:latest
    restart: unless-stopped
    ports:
      - '8890:80' # 内部端口 8890
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: password
      WORDPRESS_DB_NAME: wordpress
EOF
    cd /root/wordpress_data && docker compose up -d
    echo -e "${GREEN}✅ WordPress 已启动！${NC}"
    echo -e "${YELLOW}请去 NPM 后台，添加反向代理：${NC}"
    echo -e "   转发端口: 8890"
    read -n 1 -s -r -p "按任意键继续..."
}

# 5. Jellyfin (影院)
install_jellyfin() {
    ensure_docker
    mkdir -p /root/jellyfin_data /mnt/media
    cat > /root/jellyfin_data/docker-compose.yml <<EOF
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    restart: unless-stopped
    ports:
      - '8096:8096'
    volumes:
      - ./config:/config
      - /mnt/media:/media
EOF
    cd /root/jellyfin_data && docker compose up -d
    echo -e "${GREEN}✅ Jellyfin 已启动！${NC}"
    echo -e "${YELLOW}媒体文件请放入: /mnt/media${NC}"
    echo -e "${YELLOW}NPM 转发端口: 8096${NC}"
    read -n 1 -s -r -p "按任意键继续..."
}

# 6. Navidrome (音乐)
install_navidrome() {
    ensure_docker
    mkdir -p /root/navidrome_data /mnt/music
    cat > /root/navidrome_data/docker-compose.yml <<EOF
services:
  navidrome:
    image: deluan/navidrome:latest
    restart: unless-stopped
    ports:
      - '4533:4533'
    volumes:
      - ./data:/data
      - /mnt/music:/music
EOF
    cd /root/navidrome_data && docker compose up -d
    echo -e "${GREEN}✅ Navidrome 已启动！${NC}"
    echo -e "${YELLOW}音乐文件请放入: /mnt/music${NC}"
    echo -e "${YELLOW}NPM 转发端口: 4533${NC}"
    read -n 1 -s -r -p "按任意键继续..."
}

# 7. Uptime Kuma (监控)
install_kuma() {
    ensure_docker
    mkdir -p /root/kuma_data
    cat > /root/kuma_data/docker-compose.yml <<EOF
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    restart: always
    ports:
      - '3001:3001'
    volumes:
      - ./data:/app/data
EOF
    cd /root/kuma_data && docker compose up -d
    echo -e "${GREEN}✅ Uptime Kuma 已启动！${NC}"
    echo -e "${YELLOW}NPM 转发端口: 3001${NC}"
    read -n 1 -s -r -p "按任意键继续..."
}

# 8. Syncthing (同步)
install_syncthing() {
    ensure_docker
    mkdir -p /root/syncthing_data
    cat > /root/syncthing_data/docker-compose.yml <<EOF
services:
  syncthing:
    image: syncthing/syncthing
    hostname: my-syncthing
    environment:
      - PUID=0
      - PGID=0
    volumes:
      - ./data:/var/syncthing
    ports:
      - 8384:8384 # Web UI
      - 22000:22000/tcp # Sync TCP
      - 22000:22000/udp # Sync UDP
      - 21027:21027/udp # Discovery
    restart: unless-stopped
EOF
    cd /root/syncthing_data && docker compose up -d
    echo -e "${GREEN}✅ Syncthing 已启动！${NC}"
    echo -e "${YELLOW}NPM 转发端口: 8384 (Web管理界面)${NC}"
    read -n 1 -s -r -p "按任意键继续..."
}

# 9. Glances (实时监控)
install_glances() {
    ensure_docker
    docker run -d --restart="always" -p 61208:61208 -e GLANCES_OPT="-w" -v /var/run/docker.sock:/var/run/docker.sock:ro --pid host nicolargo/glances
    echo -e "${GREEN}✅ Glances 已启动！${NC}"
    echo -e "${YELLOW}NPM 转发端口: 61208${NC}"
    read -n 1 -s -r -p "按任意键继续..."
}

# --- 翻墙工具 (侧门) ---

install_3xui() {
    echo -e "${BLUE}--- 部署 3X-UI 面板 (推荐) ---${NC}"
    echo -e "${YELLOW}正在调用官方脚本安装...${NC}"
    bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
    echo -e "${GREEN}3X-UI 安装/升级完成。请使用脚本提示的端口和密码登录。${NC}"
    echo -e "${RED}注意：请在 3X-UI 面板设置中，避开 80 和 443 端口！建议使用 2053, 8443 等端口。${NC}"
    read -n 1 -s -r -p "按任意键继续..."
}

install_hysteria() {
    echo -e "${BLUE}--- 部署 Hysteria 2 (备用神器) ---${NC}"
    echo -e "${YELLOW}正在调用官方一键脚本...${NC}"
    bash <(curl -fsSL https://get.hy2.sh/)
    echo -e "${GREEN}Hysteria 2 安装完成。${NC}"
    echo -e "${RED}配置提示：${NC}"
    echo -e "1. 配置文件在 /etc/hysteria/config.yaml"
    echo -e "2. 同样，请务必修改端口，不要占用 443 (除非你没装 NPM)。建议使用 UDP 端口 4444。"
    read -n 1 -s -r -p "按任意键继续..."
}

# --- 安防 ---
install_fail2ban() {
    apt-get update && apt-get install -y fail2ban
    systemctl enable --now fail2ban
    echo -e "${GREEN}✅ Fail2ban 已启动，默认保护 SSH (22端口)。${NC}"
    read -n 1 -s -r -p "按任意键继续..."
}

# --- 主菜单 ---
show_menu() {
    clear
    echo -e "${GREEN}================================================================${NC}"
    echo -e "${BLUE}   VPS 包工头 (Nginx Proxy Manager 可视化版)  v8.0  by ZhangYang${NC}"
    echo -e "${GREEN}================================================================${NC}"
    echo -e " ${CYAN}--- 基础设施 (必装) ---${NC}"
    echo -e " 1) 更新系统 (apt update)"
    echo -e " 2) 配置 he.net DDNS (动态域名)"
    echo -e " 3) 部署 NPM 反向代理面板 (大门)"
    echo -e " ${CYAN}--- 应用部署 (房间) ---${NC}"
    echo -e " 4) 部署 Nextcloud (网盘)"
    echo -e " 5) 部署 WordPress (博客)"
    echo -e " 6) 部署 Jellyfin (影院)"
    echo -e " 7) 部署 Navidrome (音乐)"
    echo -e " 8) 部署 Syncthing (文件同步)"
    echo -e " 9) 部署 Uptime Kuma (监控面板)"
    echo -e " 10) 部署 Glances (资源监控)"
    echo -e " ${CYAN}--- 翻墙工具 (侧门 - 端口双保险) ---${NC}"
    echo -e " 11) 部署 3X-UI 面板 (主力推荐)"
    echo -e " 12) 部署 Hysteria 2 (备用暴力协议)"
    echo -e " ${CYAN}--- 安防与维护 ---${NC}"
    echo -e " 13) 部署 Fail2ban (防暴力破解)"
    echo -e " 14) 添加 SSH 密钥"
    echo -e " 99) 退出"
    echo -e "${GREEN}================================================================${NC}"
}

main() {
    check_root
    while true; do
        show_menu
        read -p "请输入选项: " choice
        case $choice in
            1) apt update && apt upgrade -y ;;
            2) setup_he_ddns ;;
            3) install_npm ;;
            4) install_nextcloud ;;
            5) install_wordpress ;;
            6) install_jellyfin ;;
            7) install_navidrome ;;
            8) install_syncthing ;;
            9) install_kuma ;;
            10) install_glances ;;
            11) install_3xui ;;
            12) install_hysteria ;;
            13) install_fail2ban ;;
            14) 
                echo -e "请把你的公钥粘贴到下面，按回车结束:"
                read pubkey
                mkdir -p ~/.ssh && echo "$pubkey" >> ~/.ssh/authorized_keys
                echo "✅ 密钥添加成功！"
                sleep 2
                ;;
            99) exit 0 ;;
            *) echo -e "${RED}无效选项${NC}"; sleep 1 ;;
        esac
    done
}

main
