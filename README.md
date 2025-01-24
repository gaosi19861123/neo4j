# Neo4j Kubernetes 部署

本项目包含了在 Kubernetes 集群中部署 Neo4j 数据库的配置文件和说明。

## 项目结构

```
.
├── README.md
├── secrets.yaml    # 包含 Neo4j 认证信息的 Secret 配置
└── values.yaml     # Neo4j Helm Chart 的配置文件
```

## 配置说明

### 密码配置 (secrets.yaml)

Secret 配置文件包含了 Neo4j 的认证信息：
- Secret 名称：`neo4j-auth-secret`
- 命名空间：`neo4j-system`
- 认证格式：`username/password` 的 base64 编码

### Neo4j 配置 (values.yaml)

主要配置包括：
- 数据库名称：`neo4j-release`
- 版本：Community Edition
- 资源配置：
  - CPU: 1 核
  - 内存: 4Gi
- 存储配置：
  - 使用默认存储类
  - 存储大小：10Gi
- 服务配置：
  - 类型：LoadBalancer
  - 端口：
    - HTTP (7474)
    - HTTPS (7473，默认关闭)
    - Bolt (7687)

## 部署步骤

1. 创建命名空间：
```bash
kubectl create namespace neo4j-system
```

2. 部署 Secret：
```bash
kubectl apply -f secrets.yaml
```

3. 使用 Helm 部署 Neo4j：
```bash
helm install neo4j-release neo4j/neo4j -f values.yaml -n neo4j-system
```

## 服务访问

部署完成后会创建三个服务：

1. `neo4j-release`（ClusterIP）
   - 用途：集群内部访问
   - 端口：7474 (HTTP), 7687 (Bolt)

2. `neo4j-release-admin`（ClusterIP）
   - 用途：管理接口
   - 端口：7474 (HTTP), 7687 (Bolt)

3. `neo4j-release-lb-neo4j`（LoadBalancer）
   - 用途：外部访问
   - 端口映射：
     - 7474 (HTTP Browser)
     - 7473 (HTTPS，如果启用)
     - 7687 (Bolt Protocol)

### 访问方式

1. 通过外部 LoadBalancer：
   - Neo4j Browser: `http://<EXTERNAL-IP>:7474`
   - Bolt 连接: `neo4j://<EXTERNAL-IP>:7687`

2. 集群内部访问：
   - Neo4j Browser: `http://neo4j-release.neo4j-system.svc.cluster.local:7474`
   - Bolt 连接: `neo4j://neo4j-release.neo4j-system.svc.cluster.local:7687`

获取外部 IP：
```bash
kubectl get svc neo4j-release-lb-neo4j -n neo4j-system
```

## 验证部署

1. 检查所有资源状态：
```bash
kubectl get pods,svc -n neo4j-system
```

2. 测试连接：
```bash
kubectl run --rm -it --namespace "neo4j-system" --image "neo4j:5.26.1" cypher-shell \
  -- cypher-shell -a "neo4j://neo4j-release.neo4j-system.svc.cluster.local:7687" -u neo4j -p "$(kubectl get secret neo4j-auth-secret -o go-template='{{.data.NEO4J_AUTH | base64decode }}' | cut -d '/' -f2)"
```

## 注意事项

1. 确保 Kubernetes 集群有足够的资源
2. 存储类必须支持所需的存储大小
3. 如果需要更改密码，需要更新 Secret 并重启 Pod
4. 生产环境建议开启 HTTPS

## 删除服务

完整删除服务及其相关资源：

```bash
# 1. 删除 Helm release
helm uninstall neo4j-release

# 2. 删除 Secret
kubectl delete secret neo4j-auth-secret

# 3. 删除持久卷声明（PVC）
kubectl delete pvc data-neo4j-0
```

验证删除状态：
```bash
# 验证所有资源是否已删除
kubectl get pods,svc,secret,pvc | grep neo4j
```

注意事项：
- 删除 PVC 将永久删除数据
- 如需保留数据，请在删除前进行备份
- 确保删除前没有其他服务依赖于此 Neo4j 实例

## 注意事项

1. 安全建议：
   - 在生产环境中使用强密码
   - 考虑启用 HTTPS
   - 根据需要配置网络策略

2. 维护建议：
   - 定期备份数据
   - 监控资源使用情况
   - 及时更新版本

## 故障排除

如果遇到问题，可以查看 Pod 日志：
```bash
kubectl logs -l app=neo4j
```

## 回滚操作

如需回滚到之前版本：
```bash
# 查看历史版本
helm history neo4j-release

# 回滚到指定版本
helm rollback neo4j-release [版本号]
``` 