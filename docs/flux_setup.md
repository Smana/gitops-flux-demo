# Init a tiller in the namespace flux

Create a certificate for the tiller instance in the namespace flux
```
$ kubectl apply -f cert-manager/certificates/tiller-certificate.yaml -n flux
certificate.certmanager.k8s.io/flux-tiller created
```

This will create a `certificate` named `flux-tiller`
```
$ kubectl get certificates -n flux
NAME               READY   SECRET                 AGE
flux-tiller        True    flux-tiller-tls        1m
```
Check that the secret containing the certificate has been created properly
```
$ kubectl describe certificate -n flux flux-tiller
Name:         flux-tiller
Namespace:    flux
...
  Secret Name:   flux-tiller-tls
Status:
  Conditions:
    Last Transition Time:  2019-06-16T17:31:56Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2019-06-17T17:31:56Z
Events:
  Type    Reason      Age   From          Message
  ----    ------      ----  ----          -------
  Normal  CertIssued  13m   cert-manager  Certificate issued successfully

$ kubectl get secrets -n flux flux-tiller-tls
NAME              TYPE                DATA   AGE
flux-tiller-tls   kubernetes.io/tls   3      15d
```

We want to use this certificate `flux-tiller-tls` to initialize the tiller in the namespace `flux`.
In order to do so, we need to extract the key,cert and ca from the secret as follows
```
$ kubectl get secrets -n flux -o json flux-tiller-tls | jq -r '.data["tls.crt"]'| base64 -d > /tmp/tls.crt
$ kubectl get secrets -n flux -o json flux-tiller-tls | jq -r '.data["tls.key"]'| base64 -d > /tmp/tls.key
$ kubectl get secrets -n flux -o json flux-tiller-tls | jq -r '.data["ca.crt"]'| base64 -d > /tmp/ca.crt
```

Then run the following commands
```
$ kubectl create serviceaccount tiller -n flux
$ kubectl create clusterrolebinding --serviceaccount=flux:tiller --clusterrole=cluster-admin
$ helm init --service-account=tiller --tiller-namespace=flux --tiller-tls \
--tiller-tls-cert=/tmp/tls.crt --tiller-tls-key=/tmp/tls.key --tls-ca-cert=/tmp/ca.crt --tiller-tls-verify \
--override 'spec.template.spec.containers[0].command'='{/tiller,--storage=secret}'
```

Create a client certificate in order to be able to use helm commands
```
$ kubectl apply -f cert-manager/certificates/flux-helm-client-certificate.yaml -n flux

$ kubectl get certificates -n flux
NAME               READY   SECRET                 AGE
flux-helm-client   True    flux-helm-client-tls   36m
flux-tiller        True    flux-tiller-tls        33m
```

Configure your helm client to use the certificate
**warning**: don't forget to backup your helm certs if you have ones.
```
$ kubectl get secrets -o json -n flux flux-helm-client-tls | jq -r '.data["ca.crt"]' | base64 -d > ~/.helm/ca.pem                                   $ kubectl get secrets -o json -n flux flux-helm-client-tls | jq -r '.data["tls.crt"]' | base64 -d > ~/.helm/cert.pem
$ kubectl get secrets -o json -n flux  flux-helm-client-tls | jq -r '.data["tls.key"]' | base64 -d > ~/.helm/key.pem
```

Check that you can request the tiller located in the namespace `flux`
```
$ helm version --tiller-namespace=flux --tls
Client: &version.Version{SemVer:"v2.12.3", GitCommit:"eecf22f77df5f65c823aacd2dbd30ae6c65f186e", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.12.3", GitCommit:"eecf22f77df5f65c823aacd2dbd30ae6c65f186e", GitTreeState:"clean"}
```