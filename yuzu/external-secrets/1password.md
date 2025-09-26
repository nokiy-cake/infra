```shell
kubectl create secret generic op-credentials \
  --from-file=1password-credentials.json=./1password-credentials.json \
  -n external-secrets
```
```shell
kubectl create secret generic onepassword-token \
  --from-literal=token=<your-token-value> \
  -n external-secrets
```
