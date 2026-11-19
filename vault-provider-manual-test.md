# OpenShell Vault 凭证驱动 — 生产手工测试手册

> 路径：Provider 凭证注入（Vault credential driver）
> 适用：发行版 OpenShell（v0.1.7+，无需定制镜像）+ 现有生产 Vault
> 验证目标：provider 密钥存进 Vault → sandbox 内只见占位符 → supervisor 在 L7 出口替换真值

## 架构

```
openshell provider create --credential KEY=<值>
        │ gateway 进程内 Vault 驱动（HTTP API）
        ▼
Vault KV  secret/openshell/provider-credentials/<hash>   ← 真值只存这里
        │ provider 记录只存不透明 handle
        ▼
sandbox attach provider → env 是占位符 openshell:resolve:env:v..._KEY
        │ workload 发请求 → supervisor L7 出口匹配 profile endpoint
        ▼
supervisor 从 Vault 取真值 → 替换 Authorization 头 → 转发上游
```

**关键：真值永不进 sandbox 的 env/命令行；gateway 用 K8s SA 向 Vault 认证。**

---

## 0. 前提与变量

```bash
# 按你的生产环境填写
VAULT_ADDR=https://vault.prod.example.com:8200   # 生产 Vault，必须 HTTPS（驱动强制非回环 HTTPS）
GW_NS=openshell                                  # gateway 命名空间
RELEASE=openshell                                # helm release 名（决定 SA 名，见下）
KV_MOUNT=secret                                  # Vault KV 挂载点
KV_VERSION=2                                     # 1 或 2，按生产实际
ROLE=openshell-gateway                           # Vault K8s auth role 名
VAULT_AUTH_MOUNT=kubernetes                      # Vault K8s auth 挂载点
```

前提清单：
- [ ] 生产 Vault 可达，**HTTPS**（自签 CA 需准备 CA 证书；公共 CA 可跳过 CA ConfigMap）
- [ ] Vault 已启用 Kubernetes auth，且 `auth/kubernetes/config` 配好 `kubernetes_host` + CA + token reviewer
- [ ] K8s 集群 + kubectl/helm/openshell CLI
- [ ] 确定 gateway ServiceAccount 名：chart 默认 SA 名 = fullname（`fullnameOverride` 或 release 名）。下文以 SA=`openshell` 为例（设 `fullnameOverride=openshell`）

---

## 1. Vault 侧准备

```bash
# 1.1 KV 引擎（若已有 secret 挂载可跳过；注意 v1/v2）
vault secrets enable -path=$KV_MOUNT kv-v2          # v2；v1 用 kv

# 1.2 存储 policy（v2 路径带 data/ 和 metadata/；v1 去掉 data/）
vault policy write openshell-provider-storage - <<'POLICY'
path "secret/data/openshell/provider-credentials/*" {
  capabilities = ["create", "read", "update", "delete"]
}
path "secret/metadata/openshell/provider-credentials/*" {
  capabilities = ["read", "delete", "list"]
}
POLICY

# 1.3 K8s auth role：绑定 gateway 的 SA（名字必须与 chart 实际 SA 一致）
vault write auth/$VAULT_AUTH_MOUNT/role/$ROLE \
  bound_service_account_names=openshell \
  bound_service_account_namespaces=$GW_NS \
  policies=openshell-provider-storage \
  ttl=1h
```

> v1 注意：policy 路径用 `secret/openshell/provider-credentials/*`（无 data/），且下面 values 的 `kvVersion: "1"`。

---

## 2. CA ConfigMap（私有 CA 时）

```bash
# 把生产 Vault 的 CA 证书放进 gateway 命名空间的 ConfigMap，键名必须 ca.crt
kubectl -n $GW_NS create configmap vault-ca --from-file=ca.crt=./vault-ca.crt
```
公共 CA / 平台信任根：跳过此步，values 里 `caConfigMapName` 留空。

---

## 3. helm install gateway（Vault values）

```bash
cat > vault-values.yaml <<EOF
server:
  credentialDrivers:
    vault:
      enabled: true
      address: $VAULT_ADDR
      authMethod: kubernetes
      role: $ROLE
      mount: $KV_MOUNT
      kvVersion: "$KV_VERSION"
      kubernetesAuthMount: $VAULT_AUTH_MOUNT
credentialDrivers:
  vault:
    caConfigMapName: vault-ca        # 公共 CA 留空 ""
EOF

helm upgrade --install $RELEASE deploy/helm/openshell -n $GW_NS --create-namespace \
  -f vault-values.yaml \
  --set fullnameOverride=openshell \
  --set server.telemetryEnabled=false \
  --wait --timeout 5m
```

> 生产用发行版镜像：chart 默认 `ghcr.io/nvidia/openshell/gateway:<appVersion>`，无需指定本地镜像。
> 生产 TLS 保持开启（默认），不要设 disableTls。访问走 mTLS（见步骤 4）。

验证渲染：
```bash
kubectl -n $GW_NS get cm openshell-config -o jsonpath='{.data.gateway\.toml}' | grep -A9 credential_drivers
# 应看到 credential_drivers = ["vault"] + address/role/ca_bundle 自动渲染
```

---

## 4. 访问 gateway

**开发/测试（明文，仅可信环境）：**
```bash
helm upgrade $RELEASE deploy/helm/openshell -n $GW_NS --reuse-values \
  --set server.disableTls=true --set server.auth.allowUnauthenticatedUsers=true
kubectl -n $GW_NS port-forward svc/openshell 18080:8080 &
export OPENSHELL_GATEWAY_ENDPOINT=http://127.0.0.1:18080
```

**生产（mTLS）：** 用 chart 生成的客户端证书，`openshell gateway add` 注册 mTLS gateway（参考 chart PKI）。下文命令对两种访问方式通用。

---

## 5. 导入 profile + 创建 provider

```bash
# 5.1 导入 provider profile（每个新 gateway 都要导一次）
openshell provider profile import --from ./providers --global

# 5.2 创建 provider（密钥自动存进 Vault）
openshell provider create --name vault-test --type openai \
  --credential OPENAI_API_KEY=<测试密钥>
```

> 自定义推理端点（非 api.openai.com）：需单独 profile 指明 host，见附录 A。

---

## 6. 验证（核心）

```bash
# 6.1 密钥确实存进 Vault（应看到托管 hash 路径）
vault kv list $KV_MOUNT/openshell/provider-credentials
# 取值核对（v2）：
vault kv get -field=value $KV_MOUNT/openshell/provider-credentials/<hash>

# 6.2 provider get 不泄露明文
openshell provider get vault-test          # 只显示 key 名，无值

# 6.3 sandbox 内 env 是占位符（真值不进 sandbox）
openshell sandbox create --name vault-sb --provider vault-test \
  --no-auto-providers --no-keep --no-tty -- bash -lc 'echo $OPENAI_API_KEY'
# 期望输出：openshell:resolve:env:v..._OPENAI_API_KEY  （占位符，非真值）

# 6.4 L7 注入（可选，需匹配 endpoint）：sandbox 内向上游发请求，
#     supervisor 在出口把占位符换成真值。用附录 A 的自定义 profile + echo 服务验证。

# 6.5 删除 provider → Vault 对象同步清除
openshell provider delete vault-test
vault kv list $KV_MOUNT/openshell/provider-credentials   # 对应 hash 应消失
```

通过标准：
- [ ] 6.1 Vault 里有托管路径，值正确
- [ ] 6.2 provider get 无明文
- [ ] 6.3 sandbox env 是占位符
- [ ] 6.5 删除后 Vault 对象消失

---

## 7. 清理

```bash
openshell sandbox delete vault-sb 2>/dev/null
openshell provider delete vault-test 2>/dev/null
helm uninstall $RELEASE -n $GW_NS
kubectl delete namespace $GW_NS
# Vault 侧 role/policy 按需保留或清理
```

---

## 故障排查

| 现象 | 原因 / 处理 |
|---|---|
| 启动报 `address must use https` | 生产 Vault 必须 HTTPS；驱动拒绝非回环 HTTP |
| `Vault Kubernetes auth rejected the service account token` | role 的 `bound_service_account_names/namespaces` 与 gateway 实际 SA/ns 不符；核对 SA 名（fullname） |
| `UnknownIssuer` / TLS 失败 | CA ConfigMap 缺失或 ca.crt 不对；核对 vault-ca 与 Vault 证书链 |
| hostname mismatch | Vault 证书 SAN 不含 address 主机名；证书需覆盖该 DNS |
| `permission denied` 读 KV | policy 路径不对（v1/v2 的 data/ 差异）；核对 kvVersion 与 policy |
| sandbox env 是真值而非占位符 | provider 未正确 attach，或用了 --env 直接传值（应走 --provider） |
| `provider profile not found` | 新 gateway 未导入 profile（步骤 5.1） |

---

## 附录 A：自定义推理端点 profile（L7 注入到非 openai 主机）

若你的上游是本地/私有 OpenAI 兼容服务（如 `http://192.168.5.190:8080`），openai profile 的 endpoint 不匹配，需单独 profile：

```yaml
# strata-local.yaml
id: strata-local
display_name: Local Inference
category: inference
inference_capable: true
credentials:
  - name: api_key
    env_vars: [OPENAI_API_KEY]
    required: true
    auth_style: bearer
    header_name: authorization
discovery:
  credentials: [api_key]
endpoints:
  - host: 192.168.5.190
    port: 8080
    protocol: rest
    access: read-write
    enforcement: enforce          # 凭据注入必须 L7 inspect，不能 L4-only
binaries: [/usr/local/bin/opencode, /usr/bin/curl]
```

```bash
openshell provider profile import -f strata-local.yaml --global
openshell provider create --name local-llm --type strata-local --credential OPENAI_API_KEY=<key>
# sandbox policy 需放行该 endpoint（protocol: rest, enforcement: enforce）
```

> 注意：凭据注入 endpoint 必须声明 L7（`protocol: rest, enforcement: enforce`），OpenShell 拒绝把凭据发到无法审计的 L4 通道。
