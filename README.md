# Neo4j on AKS 部署指南

本指南介绍如何在Azure Kubernetes Service (AKS)上使用Helm部署Neo4j社区版。

## 前提条件

- 已创建AKS集群
- 已安装kubectl并配置好集群访问凭证
- 已安装Helm 3.x

## 快速开始

### 1. 添加Neo4j Helm仓库

```bash
helm repo add neo4j https://helm.neo4j.com/neo4j
helm repo update
```

### 2. 部署Neo4j

使用提供的values.yaml进行部署：

```bash
helm install my-neo4j neo4j/neo4j -f values.yaml
```

### 3. 验证部署

检查Pod状态：
```bash
kubectl get pods
```

检查服务状态：
```bash
kubectl get svc
```

## 配置说明

### values.yaml 主要配置

```yaml
neo4j:
  name: "my-neo4j"
  edition: "community"    # 社区版
  password: "your-secure-password"  # 设置密码

resources:
  cpu: "1"    # CPU限制
  memory: "4Gi"  # 内存限制

volumes:
  data:
    mode: "defaultStorageClass"
    size: "10Gi"  # 存储大小

services:
  neo4j:
    enabled: true
    spec:
      type: LoadBalancer  # 服务类型
```

## 访问Neo4j

部署完成后，可以通过以下方式访问：

1. Neo4j Browser:
   - URL: `http://<EXTERNAL-IP>:7474`
   - 默认用户名: neo4j
   - 密码: 在values.yaml中配置的密码

2. Bolt连接:
   - URL: `bolt://<EXTERNAL-IP>:7687`

获取EXTERNAL-IP:
```bash
kubectl get svc my-neo4j
```

## 持久化存储

Neo4j使用AKS的默认StorageClass进行数据持久化，数据存储在10Gi的持久卷中。

## 资源限制

- CPU: 1 core
- 内存: 4Gi

## 常见问题

1. 如果需要修改配置：
```bash
helm upgrade my-neo4j neo4j/neo4j -f values.yaml
```

2. 如果需要卸载：
```bash
helm uninstall my-neo4j
```

## 安全建议

1. 修改默认密码
2. 根据需要配置网络策略
3. 定期备份数据

## 维护操作

### 查看日志
```bash
kubectl logs -f <pod-name>
```

### 进入容器
```bash
kubectl exec -it <pod-name> -- bash
```

### 备份数据
确保定期备份数据目录（/data） 