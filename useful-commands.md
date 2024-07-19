# Useful commands

## Get all infos about k8s apis

```bash
kubectl api-resources
```

> Meaning of API version field
>
> v1 is used to denote a stable object

## Runing commands in a container

```bash
kubectl exec -it <pod_name> -- /bin/bash
```
