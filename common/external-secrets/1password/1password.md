```shell
# 1password need to mannually encode the credentials file to base64
kubectl create secret generic op-credentials \
  --from-literal=1password-credentials.json=$(cat ./1password-credentials.json | base64 -w 0) \
  -n external-secrets
```
```shell
kubectl create secret generic onepassword-token \
  --from-literal=token=<your-token-value> \
  -n external-secrets
```
