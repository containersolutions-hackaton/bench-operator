# Cloud Natives Hackathon

## Goal

* Kubernetes custom Operator
  * Use the distributed [Locust load testing tool](https://locust.io/)
  * Automatically attaches locust load testing pod to each Kubernetes slave deployment and sends load testing results back to Kubernetes master
* Access a Kubernetes (K8s) cluster from different namespaces (i.e. userXXX.training.csol.cloud)
* Each namespace's deployments (i.e. Wordpress, MySQL, NGINX caching, K8S operator, benchmarking) needed to communicate with each other

## Deployment

### Secret Key Pod (within Deployment)

Secret setup instead of the secret step in the example https://kubernetes.io/docs/concepts/configuration/secret/

## Notes

* Use container for MySQL

```bash
curl -o mysql.yaml https://kubernetes.io/examples/application/wordpress/mysql-deployment.yaml
k apply -f mysql.yaml
k get pods
kubectl delete pod wordpress-mysql-66594fb556-2h8bg --now
```

We discussed using Github pages for pushing containers.

The key (i.e. `db-user-pass`) must match that used in the .yaml file. In our case it was changed to `mysql-pass`.
Check that the .yaml file key for username and password match (i.e. username instead of username.txt).

Generate secrets:

```bash
echo -n 'admin' > ./username.txt
echo -n 'password' > ./password.txt
kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
kubectl get secrets
```

Check secrets:

```bash
NAME                  TYPE                                  DATA   AGE
db-user-pass          Opaque                                2      25s
default-token-xmf9d   kubernetes.io/service-account-token   3      22h
```

```bash
kubectl describe secrets/db-user-pass
kubectl delete secret db-user-pass --now
```

Generate secrets manually (alternative):

```bash
echo -n 'admin' | base64
YWRtaW4=
echo -n 'password' | base64
cGFzc3dvcmQ=
```

Create secret file with contents:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
```

Add contents to file (or right click and paste usig Vim in cloud):

```bash
cat <<EOF >>./secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
EOF
```

Deploy the secret container (pod) using its configuration into the Deployment:

```bash
kubectl apply -f ./secret.yaml
```

Check and delete listed secret:

```bash
kubectl get secrets
kubectl delete secret mysecret --now
```

Add contents of a raw Github repo file into a new file in the cloud session:

```bash
curl -o mysql.yaml https://raw.githubusercontent.com/containersolutions-hackaton/bench-operator/luke-mysql/mysql-deployment.yaml
```

Deploy the MySQL container (pod) to the Deployment:

```bash
k apply -f mysql.yaml
```

List the PersistantVolume container pod (pvc), all pods, the Service pod (i.e. where network ports exposed for communication between each deployment) and a specific namespace of user122:

```bash
kubectl get pvc
kubectl get pods
kubectl get services -n user122
k get namespaces
```

Additionally had to a WORDPRESS_DB_USER to the Wordpress .yaml file (otherwise `k describe...` and `k logs...` showed error). See Dockerhub documentation.

It was necessary to delete pods when passwords were changed and redeploy if they weren't configured to persist and restart by themselves.

Check the contents of the deployed secret pod .yaml file. Convert its encrypted password into base64. Edit it live.

```bash
k get secret mysql-pass -o yaml
echo "YXBpVXJsOiAiaHR0cHM6Ly9teS5hcGkuY29tL2FwaS92MSIKdXNlcm5hbWU6IHt7dXNlcm5hbWV9fQpwYXNzd29yZDoge3twYXNzd29yZH19" | base64 -d
k edit secret mysql-pass
```

We had to tell the deployment that each participant was working in a different deployment namespace (i.e. user123 instead of user122) by changing the workpress-mysql in the wordpress.yaml file to the external namespace name i.e.

```yaml
...
value: wordpress-mysql.user123.namespace-b.svc.cluster.local
...
```

Note: It was not necessary to re-deploy the mysql.yaml to its container pod, since it already points to the password container pod (that doesn't exist until we've deployed it), but when we've deployed the password container, we should then delete the mysql.yaml deployment, then kubernetes will detect that the pod's environment has changed, and since it has a persistent volume (i.e. it restores the state), it will re-create the mysql.yaml deployment automatically that'll then point to the password container. Note that since we were using different Deployments for each user namespace, it is necessary to write a chown function that listens to changes to secrets, then it knows that the configuration changed and when to redeploy each different pod that are on different deployments.

```
k delete pod wordpress-mysql-66594fb556-2h8bg --now
// k apply -f mysql.yaml
k describe pod wordpress-mysql-66594fb556-2h8bg
```

Go into wordpress container to check the variables:

```
k exec wordpress0..... -ti /bin/bash
env
```

Go into MySQL container and add an admin with privileges:

```bash
mysql -u root -p
CREATE USER 'admin'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

Setup a proxy container pod so it redirects ingresses to the Wordpress container. We used cube-ctl to connect from localhost via tunnel (not `k proxy` inside) so when go to root address it redirects from ingress deployment to the wordpress container

Download file from cloud host to directory on localhost:

```bash
scp -P 32297 user123@IP_ADDRESS:wordpress.yml .
```

Benchmarking with apache benchmarking a given website:
```
ab -c 5 -n 50 http://wordpress-ingress.training.csol.cloud
```

Write csv file with % of time it took to serve that percentage of requests
```
ab -c 10 -n 100 -e csv-file http://wordpress-ingress.training.csol.cloud
```

Write measured values as gnu plotfile
```
ab -g gnuplot-file http://wordpress-ingress.training.csol.cloud
```

We needed to write a .yaml file that allowed a Kubernetes Operator to use a benchmarking container pod that ran the `ab` command to benchmark the difference in loading time between a Wordpress site endpoint that had caching (i.e. http://wordpress-cache.training.csol.cloud), and one that didn't have caching (i.e. http://wordpress-ingress.training.csol.cloud).
It would modify the environment variables that would be used as the parameters to the `ab` command.

Reference: http://www.mikemead.me/blog/website-load-testing-with-apache-bench/

## Resources Used

* Kubernetes - https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/
* Locust - https://docs.locust.io/en/stable/index.html

## Books & Certification

* Certification in Kubernetes - https://training.linuxfoundation.org/cybermonday/
* Cloud Native Transformation - https://www.oreilly.com/library/view/cloud-native-transformation/9781492048893/?utm_content=103192326&utm_medium=social&utm_source=twitter&hss_channel=tw-2810968542
