# **Docker 自动化管理脚本大全**

 **Docker 自动化管理脚本**，可帮助您更高效、安全地管理容器和镜像。它支持容器启停、更新、日志、备份、进入容器、导出配置等功能，基本覆盖日常 Docker 运维场景。

* * *

#  **脚本代码**

保存为 docker-manager.sh 并赋予执行权限：

```shell

#!/bin/bash

# Docker 管理脚本集合（增强版）
# 用法: ./docker-manager.sh [command] [options]

set -e
# 彩色输出
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color
# 显示使用说明
usage() {
    echo -e "${BLUE}Docker 管理脚本${NC}"
    echo "用法: $0 [command]"
    echo ""
    echo "命令:"
    echo "  list          列出所有容器"
    echo "  start-all     启动所有容器"
    echo "  stop-all      停止所有容器"
    echo "  restart-all   重启所有容器"
    echo "  cleanup       清理无用镜像和容器"
    echo "  stats         显示容器资源使用情况"
    echo "  update [name] 更新指定容器（拉取新镜像并重启）"
    echo "  backup [path] 备份所有容器的数据卷（默认: ./backups）"
    echo "  network       显示网络信息"
    echo "  logs [name]   查看容器日志（默认显示所有容器日志）"
    echo "  exec [name]   进入容器交互式终端"
    echo "  export [name] 导出容器配置到 JSON"
    echo "  top [name]    查看容器内进程"
    echo ""
}

# 检查 Docker 是否可用
check_docker() {
    if ! command -v docker &> /dev/null; then
        echo -e "${RED}错误:${NC} 未找到 Docker，请先安装 Docker"
        exit 1
    fi
}

# 列出容器
list_containers() {
    echo -e "${YELLOW}正在运行的容器:${NC}"
    docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"

    echo -e "\n${YELLOW}所有容器:${NC}"
    docker ps -a --format "table {{.ID}}\t{{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
}

# 启动所有容器
start_all_containers() {
    containers=$(docker ps -aq)
    if [ -z "$containers" ]; then
        echo "没有容器可启动"
        return
    fi
    echo "正在启动所有容器..."
    docker start $containers
    echo -e "${GREEN}所有容器已启动${NC}"
}

# 停止所有容器
stop_all_containers() {
    containers=$(docker ps -q)
    if [ -z "$containers" ]; then
        echo "没有正在运行的容器"
        return
    fi
    echo "正在停止所有容器..."
    docker stop $containers
    echo -e "${GREEN}所有容器已停止${NC}"
}

# 重启所有容器
restart_all_containers() {
    containers=$(docker ps -q)
    if [ -z "$containers" ]; then
        echo "没有正在运行的容器"
        return
    fi
    echo "正在重启所有容器..."
    docker restart $containers
    echo -e "${GREEN}所有容器已重启${NC}"
}

# 清理无用资源
cleanup_docker() {
    echo "清理无用容器..."
    docker container prune -f

    echo "清理无用镜像..."
    docker image prune -af

    echo "清理无用网络..."
    docker network prune -f

    echo "清理无用卷..."
    docker volume prune -f

    echo -e "${GREEN}清理完成${NC}"
}

# 显示资源使用情况
show_stats() {
    docker stats --no-stream
}

# 更新容器
update_container() {
    if [ -z "$1" ]; then
        echo "错误: 需要指定容器名称"
        exit 1
    fi

    container_name=$1
    echo "正在更新容器: $container_name"

    # 获取镜像
    image_name=$(docker inspect --format='{{.Config.Image}}' $container_name)

    echo "拉取最新镜像: $image_name"
    docker pull $image_name

    echo "停止并删除容器: $container_name"
    docker stop $container_name
    docker rm $container_name

    echo -e "${YELLOW}请手动重新运行容器，因为运行参数可能不同${NC}"
    echo "示例: docker run -d --name $container_name [其他参数] $image_name"
}

# 备份数据卷
backup_volumes() {
    backup_path=${1:-./backups}
    mkdir -p $backup_path

    echo "备份数据卷到: $backup_path"

    for container in $(docker ps -aq); do
        container_name=$(docker inspect --format='{{.Name}}' $container | sed 's/\///g')
        volumes=$(docker inspect -f '{{range .Mounts}}{{if .Name}}{{.Name}} {{end}}{{end}}' $container_name)

        if [ ! -z "$volumes" ]; then
            echo "备份容器 $container_name 的数据卷..."
            for volume in $volumes; do
                echo "备份卷: $volume"
                docker run --rm -v $volume:/source -v $backup_path:/backup alpine \
                    tar -czf /backup/${container_name}_${volume}_$(date +%Y%m%d_%H%M%S).tar.gz -C /source .
            done
        fi
    done

    echo -e "${GREEN}备份完成${NC}"
}

# 显示网络信息
show_network() {
    echo "Docker 网络:"
    docker network ls

    echo -e "\n容器网络详情:"
    for network in $(docker network ls -q); do
        echo "网络 $(docker network inspect -f '{{.Name}}' $network):"
        docker network inspect -f '{{range .Containers}}{{.Name}} {{end}}' $network
        echo ""
    done
}

# 查看容器日志
view_logs() {
    if [ -z "$1" ]; then
        echo "显示所有容器日志 (Ctrl+C 退出)..."
        docker-compose logs -f || docker logs -f $(docker ps -q)
    else
        container_name=$1
        echo "显示容器 $container_name 日志 (Ctrl+C 退出)..."
        docker logs -f $container_name
    fi
}

# 进入容器
exec_container() {
    if [ -z "$1" ]; then
        echo "错误: 需要指定容器名称"
        exit 1
    fi
    docker exec -it $1 /bin/bash || docker exec -it $1 sh
}

# 导出容器配置
export_container() {
    if [ -z "$1" ]; then
        echo "错误: 需要指定容器名称"
        exit 1
    fi
    docker inspect $1 > ${1}_config.json
    echo "容器配置已导出到 ${1}_config.json"
}

# 查看容器进程
top_container() {
    if [ -z "$1" ]; then
        echo "错误: 需要指定容器名称"
        exit 1
    fi
    docker top $1
}

# 主程序
check_docker

case "${1:-}" in
    list|ls)        list_containers ;;
    start-all)      start_all_containers ;;
    stop-all)       stop_all_containers ;;
    restart-all)    restart_all_containers ;;
    cleanup)        cleanup_docker ;;
    stats|st)       show_stats ;;
    update)         update_container "$2" ;;
    backup)         backup_volumes "$2" ;;
    network)        show_network ;;
    logs)           view_logs "$2" ;;
    exec)           exec_container "$2" ;;
    export)         export_container "$2" ;;
    top)            top_container "$2" ;;
    *)              usage; exit 1 ;;
esac

exit 0
```

* * *

#  **使用说明**

1.  保存脚本为 docker-manager.sh
    
2.  添加执行权限：
    

    
    chmod +x docker-manager.sh
    

3.  运行脚本：
    

    ./docker-manager.sh [command]

* * *

# **✅ 功能说明**

-   **list / ls**
    
    : 列出所有容器（运行中和已停止的）
    
-   **start-all**
    
    : 启动所有容器
    
-   **stop-all**
    
    : 停止所有容器
    
-   **restart-all**
    
    : 重启所有容器
    
-   **cleanup**
    
    : 清理无用的镜像、容器、网络和卷
    
-   **stats / st**
    
    : 显示容器资源使用情况
    
-   **update \[name\]**
    
    : 更新指定容器（拉取新镜像）
    
-   **backup \[path\]**
    
    : 备份所有容器的数据卷
    
-   **network**
    
    : 显示 Docker 网络信息
    
-   **logs \[name\]**
    
    : 查看容器日志（单个或全部）
    
-   **exec \[name\]**
    
    : 进入容器交互式终端
    
-   **export \[name\]**
    
    : 导出容器配置到 JSON
    
-   **top \[name\]**
    
    : 查看容器内运行的进程
    

* * *

#  **额外实用命令**

    
    # 查看容器资源使用情况（实时）
    docker stats
    
    # 查看镜像层大小
    docker image ls --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
    
    # 一键停止并删除所有容器
    docker stop $(docker ps -aq) && docker rm $(docker ps -aq)
    
    # 删除所有未使用的镜像
    docker image prune -a
    
    # 显示磁盘使用情况
    docker system df
    
    # 查看容器 IP 地址
    docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name
    
    # 在容器中执行命令
    docker exec -it container_name /bin/bash
    
    

* * *

⚡ 有了这份增强版 docker-manager.sh，你基本可以像用小型 CLI 工具一样轻松管理整个 Docker 环境。
