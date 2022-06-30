# ocp-logging-test-based-loki-vector

[![Contribute](https://www.eclipse.org/che/contribute.svg)](https://codeready-codeready-workspaces-operator.apps.sandbox-m2.ll9k.p1.openshiftapps.com/#https://github.com/k-srkw/ocp-logging-test-based-loki-vector.git)

OpenShift Logging Test based on Loki and Vector

```
$ oc new-project logging-test
```

## [Deploy MinIO](https://docs.min.io/minio/k8s/openshift/deploy-minio-on-openshift.html)

```
$ oc create secret generic minio-creds-secret \
    --from-literal=accesskey=admin \
    --from-literal=secretkey=adminadmin

$ oc create secret generic console-secret \
    --from-literal=CONSOLE_ACCESS_KEY=consoleadmin \
    --from-literal=CONSOLE_SECRET_KEY=adminadmin

$ oc create serviceaccount minio
$ oc adm policy add-scc-to-user nonroot -z minio

$ oc apply -f minio-tenant.yaml
```

## [Deploy Loki](https://github.com/grafana/loki/tree/main/operator)

```
oc project openshift-logging

$ oc create secret generic loki-bucket \
    --from-literal=region=ap-northeast-1 \
    --from-literal=bucketnames=loki-bucket \
    --from-literal=access_key_id=admin \
    --from-literal=access_key_secret=adminadmin \
    --from-literal=endpoint=minio.logging-test.svc.cluster.local \
    -n openshift-logging

$ oc apply -f lokistack-gateway.

$ oc -n openshift-logging create secret generic lokistack-gateway-bearer-token \
    --from-literal=token="/var/run/secrets/kubernetes.io/serviceaccount/token" \
    --from-literal=ca-bundle.crt="$(oc get cm lokistack-dev-gateway-ca-bundle -o json | jq -r '.data."service-ca.crt"')"

$ oc apply -f rbac.yaml

$ oc apply -f clusterlogforwarder.yaml
```