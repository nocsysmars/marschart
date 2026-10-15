# MARS 在 Kubernetes 环境的部署指南（sia 版）

本文件说明如何在 Kubernetes (K8s) 集群上部署 MARS SDN 控制器（**sia** 版本）。MARS 由多个容器组成，全部运行在同一个 Pod 内（all-in-one 架构）：

| 容器 | 镜像 (sia tag) | 用途 |
| --- | --- | --- |
| `container-mars` | `nocsysmars/mars:sia` | MARS/ONOS 控制器核心 |
| `container-nginx` | `nocsysmars/nginx:1.27.0-sia` | Web 前端 / 反向代理 (HTTPS, TFTP, HTTP) |
| `container-logstash` | `nocsysmars/logstash:6.1.0-sia` | 日志收集 (syslog、sFlow 等) |
| `container-es` | `nocsysmars/elasticsearch:6.1.0-sia` | Elasticsearch 数据库 |

### 代码来源

本文件涉及的部署文件均来自 GitHub，请使用 **`sia`** 分支：

| 目录 | GitHub 仓库 | 分支 |
| --- | --- | --- |
| `marschart/` （Helm Chart） | https://github.com/nocsysmars/marschart | `sia` |
| `mars_k8s_allinone/` （原生 YAML） | https://github.com/nocsysmars/mars_k8s_allinone | `sia` |

取得方式（择一）：

```bash
# 方式 A 使用的 Helm Chart
git clone -b sia https://github.com/nocsysmars/marschart.git

# 方式 B 使用的原生 YAML
git clone -b sia https://github.com/nocsysmars/mars_k8s_allinone.git
```

数据持久化使用两个 PersistentVolumeClaim (PVC)：
- `es-data`（10 Gi）：Elasticsearch 数据
- `mars-download`（1 Gi）：MARS 下载目录

本文件提供两种部署方式，请择一采用：
- **方式 A：Helm Chart**（建议，便于参数化与升级）
- **方式 B：原生 YAML (all-in-one)**（最直接，一条命令完成）

---

## 1. 前置需求

部署前请确认环境符合下列条件：

1. 一套可用的 Kubernetes 集群（建议 v1.19 以上），且 `kubectl` 已配置好可连接的 kubeconfig。
   ```bash
   kubectl get nodes
   ```
2. 集群具备可用的 **StorageClass**，能动态供应 PVC（例如 OpenEBS local、local-path、或云厂商的默认 StorageClass）。
   ```bash
   kubectl get storageclass
   ```
3. 节点可从镜像仓库拉取 `nocsysmars/*` 镜像（可连外网，或已将镜像推送到内部私有仓库）。
4. （采用方式 A 时）本机已安装 **Helm v3**。
   ```bash
   helm version
   ```
5. 资源建议：每个容器默认 `requests` 约 500m CPU / 1000Mi 内存，请确认节点有足够余量（整体至少约 2 vCPU / 4Gi 内存以上较稳定）。
6. Elasticsearch 需要节点内核参数 `vm.max_map_count=262144`。部署内含 `init-sysctl` initContainer 会以 privileged 权限自动设置，请确认集群允许 privileged 容器；若不允许，请改由管理员在节点上预先设置：
   ```bash
   sudo sysctl -w vm.max_map_count=262144
   ```

---

## 2. 方式 A：使用 Helm Chart 部署

Helm Chart 位于 `marschart/` 目录（来源：https://github.com/nocsysmars/marschart ，`sia` 分支）。若尚未取得：

```bash
git clone -b sia https://github.com/nocsysmars/marschart.git
```

### 2.1 查看与调整参数

主要可调整的参数在 `marschart/values.yaml`，常用项目：

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `namespace` | `default` | 部署的命名空间 |
| `replicaCount` | `1` | Pod 副本数（all-in-one 架构建议维持 1） |
| `marsimage.tag` | `sia` | MARS 镜像版本 |
| `marsimage.defaultTopology` | `clos` | 默认网络拓扑 |
| `marsimage.defaultAPP` | （一长串 app 清单） | ONOS 启动时加载的 App 清单 |
| `service.type` | `NodePort` | 对外服务类型 (`NodePort` / `LoadBalancer` / `ClusterIP`) |
| `service.port` | `443` | 服务对外端口（对应容器 443 的 HTTPS） |
| `resources.*` | 500m CPU / 1000Mi | 各容器资源需求 |

> 注意：Chart 中的 PVC 模板 (`es-download-pvc.yaml`、`mars-download-pvc.yaml`) 默认使用 `storageClassName: local` 并指定了固定的 `volumeName`（`pvc-es` / `pvc-mars`）。若您的集群不是使用 `local` StorageClass 或没有对应的 PV，请先调整这两个文件中的 `storageClassName`，并移除 `volumeName` 让系统动态绑定，否则 PVC 会卡在 Pending。

### 2.2 执行安装

```bash
# 创建命名空间（若 values.yaml 指定非 default）
kubectl create namespace sdn-mars

# 在 marschart 的上层目录执行安装
helm install mars ./marschart \
  --namespace sdn-mars \
  --set namespace=sdn-mars
```

如需覆盖其他参数，可用 `--set` 或自备 values 文件：

```bash
helm install mars ./marschart \
  --namespace sdn-mars \
  --set namespace=sdn-mars \
  --set service.type=NodePort \
  --set marsimage.tag=sia
```

### 2.3 升级与卸载

```bash
# 调整参数后升级
helm upgrade mars ./marschart --namespace sdn-mars --set marsimage.tag=<新版本>

# 查看状态
helm status mars --namespace sdn-mars

# 卸载（PVC 不会自动删除，需要时手动清理）
helm uninstall mars --namespace sdn-mars
```

---

## 3. 方式 B：使用原生 YAML (all-in-one) 部署

此方式使用 `mars_k8s_allinone/mars_deployment.yaml`（来源：https://github.com/nocsysmars/mars_k8s_allinone ，`sia` 分支），一个文件完成 Namespace、PVC、Deployment、Service 创建，命名空间固定为 `sdn-mars`。若尚未取得：

```bash
git clone -b sia https://github.com/nocsysmars/mars_k8s_allinone.git
```

### 3.1 执行部署

```bash
kubectl apply -f mars_k8s_allinone/mars_deployment.yaml
```

此命令会创建：
- Namespace `sdn-mars`
- PVC `es-data`（10 Gi）与 `mars-download`（1 Gi）
- Deployment `mars`（含 4 个容器 + init 容器）
- Service `mars`（NodePort，见下表）

### 3.2 Service 对外端口（NodePort）

| 服务 | 容器端口 | NodePort | 说明 |
| --- | --- | --- | --- |
| web | 443 | `30913` | MARS Web 界面 (HTTPS) |
| download | 80 | `31499` | 下载服务 (HTTP) |

> 若默认 NodePort 与既有服务冲突，请编辑 YAML 中 Service 的 `nodePort` 字段后重新 apply。

### 3.3 调整 StorageClass

YAML 中的 PVC 默认注释掉 `storageClassName`（使用集群默认 StorageClass）。若需指定特定 StorageClass，请取消注释并填入名称后再 apply。

### 3.4 卸载

```bash
kubectl delete -f mars_k8s_allinone/mars_deployment.yaml
```

---

## 4. 验证部署

无论采用哪种方式，部署后请确认资源状态。

```bash
# 设置命名空间变量（视实际命名空间调整）
NS=sdn-mars

# 1. 确认 PVC 已 Bound
kubectl get pvc -n $NS

# 2. 确认 Pod 进入 Running，且所有容器就绪（READY 应为 4/4）
kubectl get pods -n $NS -o wide

# 3. 若 Pod 未就绪，查看事件与日志
kubectl describe pod -l app=mars -n $NS
kubectl logs -l app=mars -n $NS -c container-mars
kubectl logs -l app=mars -n $NS -c container-es

# 4. 确认 Service
kubectl get svc -n $NS
```

### 获取访问地址（NodePort）

```bash
NS=sdn-mars
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
NODE_PORT=$(kubectl get svc mars -n $NS -o jsonpath='{.spec.ports[?(@.name=="web")].nodePort}')
echo "MARS Web: https://$NODE_IP:$NODE_PORT"
```

用浏览器打开上述地址即可访问 MARS Web 界面（HTTPS，首次连接会出现自签证书警告，属正常现象）。

---

## 5. 常见问题排查

| 现象 | 可能原因 | 处理方式 |
| --- | --- | --- |
| PVC 一直 `Pending` | 无可用 StorageClass，或 Chart 指定的 `storageClassName`/`volumeName` 与集群不符 | 检查 `kubectl get sc`，调整 PVC 的 `storageClassName`，必要时移除固定的 `volumeName` |
| Elasticsearch 容器反复重启 | 节点 `vm.max_map_count` 未设置，或 initContainer 无 privileged 权限 | 确认集群允许 privileged；或由管理员在节点执行 `sysctl -w vm.max_map_count=262144` |
| Pod `ImagePullBackOff` | 节点无法拉取 `nocsysmars/*` 镜像 | 确认可连外网或已配置私有仓库与 `imagePullSecrets` |
| 无法由浏览器连入 | NodePort 被防火墙阻挡或端口冲突 | 开放对应 NodePort，或改用 `LoadBalancer` / Ingress |
| 内存不足导致 Pod 被 OOMKilled | 节点资源不足 | 增加节点资源，或下调 `resources` 设置 |

---

## 6. 架构备注

- MARS 采用 **all-in-one** 架构：MARS、Nginx、Logstash、Elasticsearch 四个容器共用同一个 Pod，容器间通过 `127.0.0.1`（localhost）互相通信（例如 MARS 以 `127.0.0.1:9200` 连接 Elasticsearch）。
- 因属同一 Pod，`replicaCount` 建议维持 `1`；水平扩展（autoscaling）在此架构下不适用。
- 若需要将各组件拆分为独立 Pod（或以 Docker Compose）部署，另有 `mars_allin_one`（来源：https://github.com/nocsysmars/mars_allin_one ，`sia` 分支）提供对应配置，可按需求评估采用。
- 本 sia 版的镜像标签与 App 清单均对齐 `mars_allin_one` 的 `docker-compose.yml`（`sia` 分支）；Elasticsearch 为 6.1.0，使用 6.x 的 `discovery.zen.minimum_master_nodes` 单节点 bootstrap 方式，已移除仅适用于 ES 7.x 的 `cluster.initial_master_nodes`。
