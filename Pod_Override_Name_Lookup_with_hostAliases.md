### This is how you can override a DNS namelookup to resolve to a specific IP address


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  hostAliases:
  - ip: "10.0.0.10"
    hostnames:
    - my-fixed-name
    - my-alias-name
```
