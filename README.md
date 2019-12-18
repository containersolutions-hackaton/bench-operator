# Bench-operator

A Kubernetes Operator for creating distributed load tests on Kubernetes.

See <https://github.com/operator-framework/operator-sdk/blob/master/doc/helm/user-guide.md#helm-user-guide-for-operator-sdk>.

## Operator deployment

See <https://github.com/operator-framework/operator-sdk/blob/master/doc/helm/user-guide.md#build-and-run-the-operator>.

Quickstart:

```
kubectl create -f deploy/crds/apiextensions.k8s.io_locustloadtests_crd.yaml

kubectl create -f deploy/service_account.yaml

kubectl create -f deploy/role.yaml

kubectl create -f deploy/role_binding.yaml

kubectl create -f deploy/operator.yaml
```
