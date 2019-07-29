# Github deploy key

```
$ TMP_DIR=$(mktemp -d) && cd ${TMP_DIR}
$ ssh-keygen -q -N "" -f ./identity

$ kubectl -n flux create secret generic demo-flux-ssh --from-file=./identity -n flux
secret/hub-gateway-flux-ssh created
```