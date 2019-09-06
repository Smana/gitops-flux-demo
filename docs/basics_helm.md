kubectl get po -n kube-system -l app=helm

helm version

helm repo list

helm repo update

helm create mychart

tree mychart

helm search wordpress

helm install --name myblog --set wordpressUsername=smana,wordpressPassword=mypass,mariadb.mariadbRootPassword=Orleans stable/wordpress --dry-run --debug

helm install --name myblog --set wordpressUsername=smana,wordpressPassword=password,mariadb.mariadbRootPassword=secretpassword stable/wordpress

watch 'helm status myblog'