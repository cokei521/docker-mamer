# docker-mamer
Docker 管理脚本集合
✅ 功能说明
list / ls
: 列出所有容器（运行中和已停止的）
start-all
: 启动所有容器
stop-all
: 停止所有容器
restart-all
: 重启所有容器
cleanup
: 清理无用的镜像、容器、网络和卷
stats / st
: 显示容器资源使用情况
update [name]
: 更新指定容器（拉取新镜像）
backup [path]
: 备份所有容器的数据卷
network
: 显示 Docker 网络信息
logs [name]
: 查看容器日志（单个或全部）
exec [name]
: 进入容器交互式终端
export [name]
: 导出容器配置到 JSON
top [name]
: 查看容器内运行的进程
