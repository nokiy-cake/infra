```shell
# Since connect chart 2.3.0, double base64 encoding is no longer needed.
# Just use --from-file with the credentials file downloaded from 1Password.
kubectl create secret generic op-credentials \
  --from-file=1password-credentials.json=./1password-credentials.json \
  -n external-secrets
```
```shell
kubectl create secret generic onepassword-token \
  --from-literal=token=<your-token-value> \
  -n external-secrets
```
