# Neo4j 集群部署在 Azure Kubernetes Service (AKS)

本文档详细说明如何在Azure Kubernetes Service (AKS)上部署Neo4j企业版集群。

## 目录

- [前提条件](#前提条件)
- [部署架构](#部署架构)
- [存储配置](#存储配置)
- [集群配置](#集群配置)
- [部署步骤](#部署步骤)
- [访问Neo4j](#访问neo4j)
- [维护指南](#维护指南)
- [故障排除](#故障排除)

## 前提条件

- Azure订阅账号
- 安装Azure CLI
- 安装kubectl
- 安装Helm 3.x（可选）
- Neo4j企业版许可证
- 基本的Kubernetes知识

## 部署架构

本部署方案包含以下组件：

- AKS集群（3个节点）
- Neo4j企业版 5.14.0 集群模式
- Azure Files Premium存储（ReadWriteMany）
- LoadBalancer服务
- Headless Service用于集群通信
- 配置管理（ConfigMap和Secret）

## 存储配置

### 1. Azure Files Premium存储类
```yaml
# 支持多节点读写的存储类配置
storageClassName: azurefile-premium
accessModes: [ "ReadWriteMany" ]
parameters:
  skuName: Premium_LRS
mountOptions:
  - dir_mode=0777
  - file_mode=0777
```

### 2. 存储卷声明模板
```yaml
volumeClaimTemplates:
- metadata:
    name: neo4j-data
  spec:
    accessModes: [ "ReadWriteMany" ]
    storageClassName: "azurefile-premium"
    resources:
      requests:
        storage: 20Gi
```

## 集群配置

### 1. 集群通信端口
```yaml
ports:
- port: 7474  # HTTP
- port: 7687  # Bolt
- port: 6362  # Backup
- port: 7688  # Cluster
- port: 5000  # Discovery
```

### 2. 集群发现配置
```yaml
env:
- name: NEO4J_initial_discovery_members
  value: "neo4j-0.neo4j.default.svc.cluster.local:5000,neo4j-1.neo4j.default.svc.cluster.local:5000,neo4j-2.neo4j.default.svc.cluster.local:5000"
- name: NEO4J_dbms_mode
  value: "CORE"
```

### 3. 服务配置
- LoadBalancer服务：对外暴露服务
- Headless服务：用于集群内部通信

## 部署步骤

1. 创建存储类：
```bash
kubectl apply -f k8s/storage-class.yaml
```

2. 部署服务：
```bash
kubectl apply -f k8s/neo4j-service.yaml
```

3. 部署配置：
```bash
kubectl apply -f k8s/neo4j-configmap.yaml
kubectl apply -f k8s/neo4j-secret.yaml
```

4. 部署Neo4j集群：
```bash
kubectl apply -f k8s/neo4j-statefulset.yaml
```

5. 验证部署：
```bash
# 检查Pod状态
kubectl get pods

# 检查服务状态
kubectl get services

# 检查存储状态
kubectl get pvc

# 查看集群状态
kubectl exec neo4j-0 -- cypher-shell -u neo4j -p <your-password> "CALL dbms.cluster.overview();"
```

## 访问Neo4j

部署完成后，可通过以下方式访问：

1. 获取外部IP：
```bash
export NEO4J_HOST=$(kubectl get svc neo4j -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

2. 访问地址：
- Neo4j Browser: http://$NEO4J_HOST:7474
- Neo4j Bolt: bolt://$NEO4J_HOST:7687

## 维护指南

### 1. 集群管理
```bash
# 查看集群状态
kubectl exec neo4j-0 -- cypher-shell -u neo4j -p <password> "CALL dbms.cluster.overview();"

# 扩展集群
kubectl scale statefulset neo4j --replicas=5
```

### 2. 备份管理
```bash
# 创建备份
kubectl exec neo4j-0 -- neo4j-admin backup --backup-dir=/data/backups --database=neo4j

# 查看备份状态
kubectl exec neo4j-0 -- ls -l /data/backups
```

### 3. 监控
```bash
# 查看集群指标
kubectl exec neo4j-0 -- neo4j-admin metrics

# 查看存储使用情况
kubectl get pvc
```

### 4. 升级
```bash
# 升级Neo4j版本
kubectl set image statefulset/neo4j neo4j=neo4j:新版本号

# 验证升级
kubectl rollout status statefulset/neo4j
```

## 故障排除

### 1. 集群问题
```bash
# 检查集群连接
kubectl exec neo4j-0 -- cypher-shell -u neo4j -p <password> "CALL dbms.cluster.overview();"

# 查看集群日志
kubectl logs -l app=neo4j -f
```

### 2. 存储问题
```bash
# 检查PVC状态
kubectl describe pvc neo4j-data-neo4j-0

# 检查存储类
kubectl get storageclass azurefile-premium -o yaml
```

### 3. 网络问题
```bash
# 检查服务状态
kubectl describe svc neo4j
kubectl describe svc neo4j-headless
```

## 注意事项

1. 集群安全性：
   - 配置网络策略限制Pod间通信
   - 使用安全的密码
   - 定期更新证书

2. 性能优化：
   - 使用Premium存储提高性能
   - 根据负载调整资源限制
   - 监控集群性能指标

3. 高可用性：
   - 至少部署3个节点确保法定人数
   - 跨可用区部署
   - 配置自动备份

4. 存储管理：
   - 监控存储使用量
   - 定期清理不需要的数据
   - 配置存储自动扩展

## 支持与帮助

如遇到问题，请：
1. 检查集群状态和日志
2. 验证存储配置
3. 参考Neo4j官方文档
4. 联系Azure或Neo4j支持

## 许可证

本部署需要Neo4j企业版许可证，因为：
1. 使用了集群功能
2. 需要高级监控功能
3. 需要热备份功能

请确保在部署前获取有效的企业版许可证。 