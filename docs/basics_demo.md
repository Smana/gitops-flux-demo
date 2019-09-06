# Play with a deployment

### Create a deployment nginx

Dry-run to display the main field of a deployment object
```
kubectl create deployment nginx --image nginx --dry-run -o yaml
```

Create the deployment and show the replicaset managed by the deployment
```
kubectl create deployment nginx --image nginx
```

```
kubectl describe deploy nginx | grep -i replicaset
```

Scale up the deployment
```
kubectl scale deploy nginx --replicas 6

### Trigger a rolling update

Edit the object and add a random label
```
kubectl edit deploy nginx
```

Check the rollout status
```
kubectl rollout status deploy nginx
deployment "nginx" successfully rolled out
```

Show that there are 2 replicasets
```
kubectl get replicaset
NAME               DESIRED   CURRENT   READY   AGE
nginx-55bd7c9fd    0         0         0       6m53s
nginx-6cd969c5c7   1         1         1       3m19s
```

Show that the newly created pod has the label
```
kubectl get po --show-labels
```

# Rollback to the previous state (old replicaset)

```
kubectl rollout undo deployment nginx
deployment.extensions/nginx rolled back
```

```
kubectl get replicaset
NAME               DESIRED   CURRENT   READY   AGE
nginx-55bd7c9fd    1         1         1       7m54s
nginx-6cd969c5c7   0         0         0       4m20s
```

```
kubectl get po --show-labels
NAME                    READY   STATUS    RESTARTS   AGE   LABELS
nginx-55bd7c9fd-vwxvk   1/1     Running   0          15s   app=nginx,pod-template-hash=55bd7c9fd
```