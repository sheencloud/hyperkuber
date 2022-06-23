# Helm Chart 

## 通过Helm chart安装 hyperkuber 
1，添加hyperkuber chart repo
```
helm repo add harbor https://sheencloud.com/charts
```
2, 下载chart包
```
helm pull sheencloud/hyperkuber

```
3，修改Values.yaml的默认是值
```
# 目前hyperkuber的chart依赖mysql，redis组件，chart来自bitnami，安装时请修改values的默认配置
# mysql：https://raw.githubusercontent.com/bitnami/charts/master/bitnami/mysql/values.yaml
# redis：https://raw.githubusercontent.com/bitnami/charts/master/bitnami/redis/values.yaml
```

4，安装Chart 
```
helm install hkcmp -f hyperkuber-0.1.0.tgz -n hyperkuber --create-namespace
```