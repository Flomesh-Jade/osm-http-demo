
# 1. 下载并安装 osm-edge 命令行工具

```bash

system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v1.3.0
curl -L https://github.com/flomesh-io/osm-edge/releases/download/${release}/osm-edge-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/osm version
cp ./${system}-${arch}/osm /usr/local/bin/

```

# 2. 安装 osm-edge

```bash

export osm_namespace=osm-system 
export osm_mesh_name=osm 
osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.certificateProvider.kind=tresor \
    --set=osm.image.registry=flomesh \
    --set=osm.image.tag=1.3.0 \
    --set=osm.image.pullPolicy=Always \
    --set=osm.sidecarLogLevel=warn \
    --set=osm.controllerLogLevel=warn \
    --set=osm.remoteLogging.enable=true \
    --set=osm.remoteLogging.address=10.25.0.100 \
    --set=osm.remoteLogging.port=8123 \
    --set=osm.remoteLogging.authorization="Basic ZGVmYXVsdDoxMjM0NTY=" \
    --set=osm.remoteLogging.sampledFraction=1.0 \
    --timeout=900s
```
🔴 注意：
参数含义：
--osm_namespace：osm控制面所在命名空间，命令会自动创建该命名空间 \
--mesh-name：osm-mesh名称 \
--set=osm.certificateProvider.kind=tresor：使用 OSM 的 Tresor 证书发行者 \
--set=osm.image.registry=flomesh：仓库地址，默认flomesh，要求能够访问dockerhub，否则需要导入私有镜像库 \
--set=osm.sidecarLogLevel=warn：sidecar日志级别，支持warn，debug，error，info \
--set=osm.controllerLogLevel=warn：controller日志级别，支持warn，debug，error，info \
--set=osm.remoteLogging.enable=true：开启日志 \
--set=osm.remoteLogging.sampledFraction=1.0：traceing采样率 \
clickhouse 相关的参数，需要根据实际情况修改❗   
    --set=osm.remoteLogging.address=10.25.0.100 ： 服务器地址 \
    --set=osm.remoteLogging.port=8123 ：服务端口 \
    --set=osm.remoteLogging.authorization="Basic ZGVmYXVsdDoxMjM0NTY=" ： 账户密码base64编码 \

部署完成后，查看osm控制面所在命名空间状态：
```bash

kubectl get pod -n osm-system
kubectl get deploy -n osm-system
kubectl get svc -n osm-system
```

# 3. 部署demo服务

## 模拟服务端

```bash

kubectl create namespace pipy
osm namespace add pipy
kubectl apply -n pipy -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/main/demo/traffic-split-v4/pipy-ok.pipy.yaml
```

## 模拟客户端

```bash

kubectl create namespace curl
osm namespace add curl
kubectl apply -n curl -f https://raw.githubusercontent.com/cybwan/osm-edge-start-demo/main/demo/traffic-split-v4/curl.curl.yaml
```

## 等待依赖的 POD 正常启动

```bash

kubectl wait --for=condition=ready pod -n pipy -l app=pipy-ok --timeout=180s
kubectl wait --for=condition=ready pod -n curl -l app=curl --timeout=180s
```

# 4. 配置主版本服务流量策略

## HTTPRouteGroup配置：

```bash

cat <<EOF | kubectl apply -n pipy -f -
apiVersion: specs.smi-spec.io/v1alpha4
kind: HTTPRouteGroup
metadata:
  name: pipy-ok-service
spec:
  matches:
  - name: default
    methods:
    - GET
EOF

```

## 主版本服务v1，TrafficTarget配置：
```bash
cat <<EOF | kubectl apply -n pipy -f -
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: curl-access-pipy-ok-v1
spec:
  destination:
    kind: ServiceAccount
    name: pipy-ok-v1
    namespace: pipy
  rules:
  - kind: HTTPRouteGroup
    name: pipy-ok-service
    matches:
    - default
  sources:
  - kind: ServiceAccount
    name: curl
    namespace: curl
EOF
```

## 主版本服务可用性访问测试：

```bash

curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "$curl_client" -n curl -c curl -- curl -si pipy-ok-v1.pipy:8080
kubectl exec "$curl_client" -n curl -c curl -- curl -si pipy-ok.pipy:8080
```


# 5. 配置灰度版本流量策略

## HTTPRouteGroup配置升级，加入v2灰度条件配置
```bash
cat <<EOF | kubectl apply -n pipy -f -
apiVersion: specs.smi-spec.io/v1alpha4
kind: HTTPRouteGroup
metadata:
  name: pipy-ok-service-v2
spec:
  matches:
  - name: v2
    headers: 
    - "version" : "v2"
    methods:
    - GET
EOF
```

## 灰度版本服务v2，TrafficTarget配置：

```bash

cat <<EOF | kubectl apply -n pipy -f -
kind: TrafficTarget
apiVersion: access.smi-spec.io/v1alpha3
metadata:
  name: curl-access-pipy-ok-v2
spec:
  destination:
    kind: ServiceAccount
    name: pipy-ok-v2
    namespace: pipy
  rules:
  - kind: HTTPRouteGroup
    name: pipy-ok-service-v2
    matches:
    - v2
  sources:
  - kind: ServiceAccount
    name: curl
    namespace: curl
EOF
```

## 设置灰度策略

```bash

cat <<EOF | kubectl apply -n pipy -f -
apiVersion: split.smi-spec.io/v1alpha4
kind: TrafficSplit
metadata:
  name: pipy-ok-split-v1
spec:
  service: pipy-ok
  matches:
  - kind: HTTPRouteGroup
    name: pipy-ok-service
  backends:
  - service: pipy-ok-v1
    weight: 100
EOF

cat <<EOF | kubectl apply -n pipy -f -
apiVersion: split.smi-spec.io/v1alpha4
kind: TrafficSplit
metadata:
  name: pipy-ok-split-v2
spec:
  service: pipy-ok
  matches:
  - kind: HTTPRouteGroup
    name: pipy-ok-service-v2
  backends:
  - service: pipy-ok-v2
    weight: 100
EOF
```
## 灰度访问测试：

```bash

curl_client="$(kubectl get pod -n curl -l app=curl -o jsonpath='{.items[0].metadata.name}')"
kubectl exec "$curl_client" -n curl -c curl -- curl -si pipy-ok.pipy:8080
kubectl exec "$curl_client" -n curl -c curl -- curl -si pipy-ok.pipy:8080 -H "version:v2"
```




