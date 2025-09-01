**前提条件**：K8s 集群已提前安装 `nfs-csi` 驱动（确保集群支持 NFS 动态存储供应）

## 1. 安装 NFS 服务端与客户端组件

在 **NFS 服务器节点**（需指定一台 K8s 集群内 / 可访问的节点作为 NFS 服务端）执行以下命令，安装 NFS 核心组件：

```bash
# 更新 apt 缓存并安装 NFS 服务端（nfs-kernel-server）和客户端工具（nfs-common）
apt-get update && apt-get install -y nfs-kernel-server nfs-common
```

- `nfs-kernel-server`：NFS 服务端核心程序，提供共享目录的管理与访问服务；
- `nfs-common`：NFS 客户端工具，即使当前节点是服务端，也需安装以支持后续目录挂载验证。

## 2. 创建 NFS 共享目录并配置权限

### 2.1 创建共享目录

创建用于 K8s 存储的 NFS 根目录（示例路径 `/data`，可根据实际需求修改）：

```bash
mkdir -p /data
```

- `-p`：确保目录递归创建，若父目录不存在则自动创建。

### 2.2 配置目录权限

为避免 K8s Pod 访问共享目录时出现权限问题，将目录属主 / 属组设置为 `nobody:nogroup`（NFS 默认匿名用户组，与客户端匿名访问权限匹配）：

```bash
chown -R nobody:nogroup /data/
# 可选：设置目录权限为 755（确保读写执行权限合理）
chmod -R 755 /data/
```

## 3. 配置 NFS 共享规则

通过 `/etc/exports` 文件定义 NFS 共享策略，指定哪些客户端可访问共享目录及访问权限：

### 3.1 编辑共享配置文件

```bash
cat > /etc/exports << EOF
# NFS 共享规则：共享目录  客户端范围(权限参数)
/data *(rw,sync,no_root_squash)
EOF
```

### 3.2 权限参数说明

| 参数             | 作用解释                                                     |
| ---------------- | ------------------------------------------------------------ |
| `/data`          | 需共享的本地目录（即步骤 2 创建的目录）                      |
| `*`              | 允许访问的客户端范围（`*` 表示所有客户端，生产环境建议限制为 K8s 集群网段，如 `172.24.34.0/24`） |
| `rw`             | 客户端对共享目录拥有 **读 + 写** 权限（若仅需只读，可改为 `ro`） |
| `sync`           | 数据同步写入磁盘（确保数据不丢失，安全性高；若追求性能可改为 `async`，但有数据丢失风险） |
| `no_root_squash` | 允许客户端的 `root` 用户保持 `root` 权限（K8s 中部分 Pod 需 root 权限操作存储，建议开启） |

## 4. 重新加载 NFS 配置

修改 `/etc/exports` 后，需重新加载配置使规则生效，无需重启服务：

```bash
exportfs -ra
```

- `-r`：重新读取 `/etc/exports` 配置；
- `-a`：应用所有共享规则；
- 验证配置：执行 `exportfs` 命令，若输出 `/data *` 说明共享规则加载成功。

## 5. 启动 NFS 服务并配置开机自启

### 5.1 启动 NFS 服务

```bash
systemctl start nfs-kernel-server
```

### 5.2 配置开机自启

确保服务器重启后 NFS 服务自动恢复，避免影响 K8s 存储使用：

```bash
systemctl enable nfs-kernel-server
```

### 5.3 验证服务状态

检查 NFS 服务是否正常运行：

```bash
systemctl status nfs-kernel-server
# 若输出 "active (running)" 表示服务正常
```

## 6. 在 K8s 中创建 NFS 存储类（StorageClass）

通过 StorageClass 定义 NFS 存储的动态供应规则，K8s 可基于此自动创建 PersistentVolume（PV），无需手动创建。

### 6.1 编写 StorageClass 配置文件（sc.yaml）

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi  # 存储类名称，后续 PVC 需引用此名称
provisioner: nfs.csi.k8s.io  # NFS-CSI 驱动的 provisioner 名称（需与已安装的驱动一致）
parameters:
  server: 172.24.34.18  # NFS 服务端节点的 IP 地址（替换为你的 NFS 服务器 IP）
  share: /data          # NFS 服务端的共享目录（与步骤 2 创建的目录一致）
reclaimPolicy: Delete  # PV 回收策略：Delete（PVC 删除后自动删除 PV 及后端存储数据）/ Retain（保留 PV 和数据）
volumeBindingMode: Immediate  # 卷绑定模式：Immediate（PVC 创建后立即绑定 PV）/ WaitForFirstConsumer（等待第一个 Pod 使用时再绑定）
mountOptions:  # NFS 挂载选项（根据需求调整）
  - hard        # 硬挂载：客户端访问失败时持续重试，避免数据不一致（推荐生产环境使用）
  - nolock      # 禁用文件锁：减少锁冲突，部分场景（如多 Pod 共享存储）需开启
  - nfsvers=4.1 # 指定 NFS 版本：使用 NFS 4.1 协议（兼容性和性能较好，若服务端不支持可改为 3）
#  - nfsvers=3   # 若 NFS 服务端仅支持 NFS 3，可启用此配置并注释 4.1
```

### 6.2 应用 StorageClass 配置

在 K8s 集群的任意控制节点（或有权限的节点）执行：

```bash
kubectl apply -f sc.yaml
```

### 6.3 验证 StorageClass 创建结果

```bash
kubectl get sc
# 若输出 "nfs-csi   nfs.csi.k8s.io   Delete          Immediate           10s" 表示创建成功
```

## 7. 验证 NFS 存储可用性（可选）

创建一个测试 PVC 和 Pod，验证 NFS 存储是否能正常使用：

### 7.1 创建测试 PVC（test-pvc.yaml）

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany  # 访问模式：多节点读写（NFS 支持此模式，适合多 Pod 共享）
  resources:
    requests:
      storage: 1Gi  # 请求存储容量
  storageClassName: nfs-csi  # 引用上述创建的 StorageClass 名称
```

### 7.2 应用 PVC 并验证

```bash
# 应用 PVC 配置
kubectl apply -f test-pvc.yaml
# 查看 PVC 状态：若 STATUS 变为 Bound，说明 PV 已自动创建并绑定
kubectl get pvc test-nfs-pvc
```

### 7.3 创建测试 Pod（test-pod.yaml）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs-pod
spec:
  containers:
  - name: test-container
    image: busybox:1.35
    command: ["sh", "-c", "echo 'test nfs storage' > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: nfs-volume
      mountPath: /data  # Pod 内挂载路径
  volumes:
  - name: nfs-volume
    persistentVolumeClaim:
      claimName: test-nfs-pvc  # 引用测试 PVC 名称
```

### 7.4 验证存储写入

```bash
# 应用 Pod 配置
kubectl apply -f test-pod.yaml
# 查看 Pod 状态：确保 STATUS 为 Running
kubectl get pod test-nfs-pod
# 进入 Pod 验证文件：若能看到 test.txt 且内容正确，说明 NFS 存储正常
kubectl exec -it test-nfs-pod -- cat /data/test.txt
# （可选）在 NFS 服务端验证：若 /data 目录下出现类似 "pvc-xxx-test-nfs-pvc" 的子目录且包含 test.txt，说明后端存储正常
ls /data
```

## 注意事项

1. **网络互通**：确保 K8s 所有节点（尤其是 Worker 节点）能 ping 通 NFS 服务端 IP，且开放 NFS 相关端口（默认 2049、111 及随机端口，生产环境建议通过 iptables 限制仅 K8s 集群网段访问）；
2. **权限一致性**：若 Pod 内用户非 root，需在 NFS 共享目录下提前创建对应权限的子目录，避免写入失败；
3. **NFS 版本兼容**：若服务端仅支持 NFS 3，需将 StorageClass 的 `nfsvers=4.1` 改为 `nfsvers=3`；
4. **数据安全**：`reclaimPolicy: Delete` 适合测试环境，生产环境建议改为 `Retain`，避免误删 PVC 导致数据丢失。