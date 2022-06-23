# Quick Start

## 通过kubernetes yaml文件安装
### empty volume
* 安装demo演示环境，集群中无持久化存储，所有存储使用emptyDir
```
kubectl apply -f https://raw.githubusercontent.com/sheencloud/hyperkuber/main/manifests/hkcmp/manifest.yaml
```
通过查看pod状态，确定安装是否。
```
kubectl get po -n hyperkuber
```
查看Ingress地址，通过浏览器访问
```
kubectl get ing -n hyperkuber
```

### persistent volume
* 安装持久化存储环境，集群中存在持久化存储PV或者StorageClass
```
kubectl apply -f https://raw.githubusercontent.com/sheencloud/hyperkuber/main/manifests/hkcmp/manifest-persistent.yaml
```
通过查看pod状态，确定安装是否。
```
kubectl get po -n hyperkuber
```
查看Ingress地址，通过浏览器访问
```
kubectl get ing -n hyperkuber
```
### 修改配置

```
kubectl get cm -n hyperkuber
```
修改hyperkuber-configmap的ConfigMap中的redis以及mysql的配置信息
