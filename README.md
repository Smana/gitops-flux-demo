# gitops-flux-demo
Supports for a GitOps demo

# Initialize Helm (Kube-system)

On GKE we need to give cluster-admin perms
```
$ kubectl create clusterrolebinding smana --clusterrole=cluster-admin --user=smainklh@gmail.com
```

**Warning**: Installing Helm with cluster-admin perms is not a good security practice.
This tiller will be used in order to bootstrap the core components and will be deleted later on.

```
$ kubectl create sa tiller -n kube-system
$ kubectl create clusterrolebinding tiller --serviceaccount=kube-system:tiller --clusterrole=cluster-admin
$ helm init --upgrade --service-account=tiller
```

# Deploy cert-manager

```
$ kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml
$ kubectl create namespace cert-manager
$ kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
$ helm install --name cert-manager --namespace cert-manager --version v0.7.2 jetstack/cert-manager
```

Check that the pods are running properly

```kubectl get po -n cert-manager
NAME                                                              READY   STATUS    RESTARTS   AGE
cert-manager-86c45c86c8-9m4kd                                     1/1     Running   2          14d
cert-manager-cainjector-6885996d5-zgfcm                           1/1     Running   12         14d
cert-manager-webhook-59dfddccfd-rdrdn                             1/1     Running   0          14d
```

# Deploy vault-operator

We'll use a vault issuer, so we'll make use of the operator that makes the vault installation easier.

## Install the vault-operator
```
$ helm upgrade --install vault-operator stable/vault-operator --set etcd-operator.enabled=true --namespace cert-manager
```

Check that the pods are running properly
```
$ kubectl get po -n cert-manager
NAME                                                              READY   STATUS    RESTARTS   AGE
cert-manager-86c45c86c8-9m4kd                                     1/1     Running   2          14d
cert-manager-cainjector-6885996d5-zgfcm                           1/1     Running   12         14d
cert-manager-webhook-59dfddccfd-rdrdn                             1/1     Running   0          14d
vault-operator-5b5c6b64dc-s2wl5                                   1/1     Running   2          14d
vault-operator-etcd-operator-etcd-backup-operator-69b95848dx8mw   1/1     Running   2          14d
vault-operator-etcd-operator-etcd-operator-765667ff65-kdshj       1/1     Running   2          14d
vault-operator-etcd-operator-etcd-restore-operator-8c6bdc9t2ssw   1/1     Running   11         14d
```

# Create a vault cluster

```
$ vault-operator/vault-pki-vault.yaml
```

This will create a vault cluster
```
$ kubectl get vault -n cert-manager
NAME        AGE
vault-pki   14d
```

```
$ kubectl describe vault vault-pki -n cert-manager
Name:         vault-pki
Namespace:    cert-manager
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"vault.security.coreos.com/v1alpha1","kind":"VaultService","metadata":{"annotations":{},"name":"vault-pki","namespace":"cert...
API Version:  vault.security.coreos.com/v1alpha1
Kind:         VaultService
Metadata:
  Creation Timestamp:  2019-06-01T10:39:38Z
  Generation:          1
  Resource Version:    7181134
  Self Link:           /apis/vault.security.coreos.com/v1alpha1/namespaces/cert-manager/vaultservices/vault-pki
  UID:                 8e2127b7-8459-11e9-918f-42010a840107
Spec:
  TLS:
    Static:
      Client Secret:  vault-pki-default-vault-client-tls
      Server Secret:  vault-pki-default-vault-server-tls
  Base Image:         quay.io/coreos/vault
  Config Map Name:
  Nodes:              2
  Version:            0.9.1-0
Status:
  Client Port:   8200
  Initialized:   true
  Phase:         Running
  Service Name:  vault-pki
  Updated Nodes:
    vault-pki-5669bfcb94-6t99d
    vault-pki-5669bfcb94-7vzn8
  Vault Status:
    Active:  vault-pki-5669bfcb94-6t99d
    Sealed:  <nil>
    Standby:
      vault-pki-5669bfcb94-7vzn8
Events:  <none>
```

# Unseal the vault server
In another terminal, use port-forward in order to use the vault cli
```
$ kubectl get vault vault-pki -n cert-manager -o jsonpath='{.status.vaultStatus.sealed[0]}' | xargs -0 -I {} kubectl port-forward {} -n cert-manager 8200
```

Get back to your the initial terminal and run
```
$ export VAULT_ADDR='https://localhost:8200'
$ export VAULT_SKIP_VERIFY="true"
```

Now your able to interact with vault, the first thing to do is to initiate it as follows
```
$ vault operator init
Unseal Key 1: xxx
Unseal Key 2: yyy
Unseal Key 3: zzz
Unseal Key 4: bbb
Unseal Key 5: aaa

Initial Root Token: a1b5847a-2ae3-nnbb-ppbb-iiyy
```

As you can see this commands prints 5 `unseal keys` that will be used to unseal the cluster.
We will use the root token for our demo but this is not a good security practice for production.

To unseal the vault server you need to run the following command 3 times and use 3 different unseal tokens.
```
$ vault operator unseal
Unseal Key (will be hidden):
```

Then you will notice that the sealed field will change to `false`:
```
$ vault status
Seal Type       shamir
Sealed          false
Total Shares    5
Threshold       3
Version         0.9.1
Cluster Name    vault-cluster-e48540f1
Cluster ID      ba6328b5-e9b3-8058-6b25-1845d3a74682
HA Enabled      true
HA Cluster      https://vault-pki.cert-manager.svc:8201
HA Mode         active
```

You should be able to run vault commands by using the root token created above as follows

```
$ export VAULT_TOKEN=a1b5847a-2ae3-nnbb-ppbb-iiyy
$ vault secrets list
Path          Type         Description
----          ----         -----------
cubbyhole/    cubbyhole    per-token private secret storage
identity/     identity     identity store
secret/       kv           key/value secret storage
sys/          system       system endpoints used for control, policy and debugging
```

## Configure the vault pki

Enable the PKI secret engine
```
$ vault secrets enable pki
$ vault secrets tune -max-lease-ttl=8760h pki
```

Create the root CA with a TTL of one year
```
$ vault write pki/root/generate/internal common_name=smana.dev ttl=8760h
Key              Value
---              -----
certificate      -----BEGIN CERTIFICATE-----
MIIDLzCA...TAFBoYz
-----END CERTIFICATE-----
expiration       1559645823
issuing_ca       -----BEGIN CERTIFICATE-----
MIIDLzCCA....TAFBoYz
-----END CERTIFICATE-----
serial_number    3d:b0:12:e5:a3:a5:b9:d0:6a:e2:69:16:99:58:5d:67:e9:10:9c:6f
```

Create a role that will be allowed to issue certificates for the domain `smana.dev`
```
$ vault write pki/roles/cert-manager allowed_domains=smana.dev allow_subdomains=true max_ttl=168h
```


## Create an approle that will allow cert-manager to use the Vault API

Create a policy that gives the permissions to the mount point /pki
```
$ vault policy write cert-manager vault-operator/vault-pki-policy.hcl
Success! Uploaded policy: cert-manager
```

Create the approle
```
$ vault auth enable approle
$ vault write auth/approle/role/cert-manager policies=cert-manager token_ttl=10m token_max_ttl=20m

$ vault read auth/approle/role/cert-manager/role-id
Key        Value
---        -----
role_id    9b3d31d1-5912-b575-31f6-xxx
vault write -f auth/approle/role/cert-manager/secret-id
Key                   Value
---                   -----
secret_id             d9ba242a-cf75-1193-b37b-xxx
secret_id_accessor    056968b3-19eb-cd54-4618-xxx
```

# Configure the Vault issuer

The certificate will be generated in the namespace `flux`.
**Note**: an issuer is declared by namespace

Create a secret that contains the vault secret id created earlier
```
$ kubectl create secret generic -n flux --from-literal=secretId=d9ba242a-cf75-1193-b37b-xxx cert-manager-vault-approle
```

Create the issuer
```
$ kubectl apply -f cert-manager/vault-issuer.yaml -n flux
```

Check the vault issuer status
```
$ kubectl describe issuer -n flux vault-issuer
Name:         vault-issuer
Namespace:    flux
API Version:  certmanager.k8s.io/v1alpha1
Kind:         Issuer
....
Status:
  Conditions:
    Last Transition Time:  2019-06-16T17:28:24Z
    Message:               Vault verified
    Reason:                VaultVerified
    Status:                True
    Type:                  Ready
Events:                    <none>
```

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