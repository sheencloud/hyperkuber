## install helm chart on openshift
for example ,helm install name "hkcmp",you should 
```
oc adm policy remove-scc-from-user privileged -z hkcmp-mysql
oc adm policy remove-scc-from-user privileged -z hkcmp-redis
oc adm policy add-scc-to-user anyuid -z hkcmp-mysql
oc adm policy add-scc-to-user anyuid -z hkcmp-redis
```