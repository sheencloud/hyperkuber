## before install

### update configmap mysql settings
```
```

### update configmap redis settings


### update ingress  host address
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: portal
  namespace: hyperkuber
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: "portal.hyperkuber.hk.ocp4.com" ## change me
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: portal
            port:
              number: 80
```