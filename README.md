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

## [Deploy Loki with Gateway](https://github.com/grafana/loki/tree/main/operator)

```
oc project openshift-logging

$ oc apply -f clusterlogging.yaml

or

$ oc apply -f clusterlogging-vector.yaml

$ oc create secret generic loki-bucket \
    --from-literal=region=<region_name> \
    --from-literal=bucketnames=<bucket_name> \
    --from-literal=access_key_id=<access_key_name> \
    --from-literal=access_key_secret=<secret_name> \
    --from-literal=endpoint=<endpoint_url> \
    -n openshift-logging

$ oc apply -f lokistack-gateway.yaml

$ oc apply -f rbac.yaml

$ oc apply -f clusterlogforwarder.yaml
```

Gateway が TLS に対応していない。 TLS 接続の Secret で Token を入れていれても多分 HTTP だと認識されない。

## Cleanup

```
$ oc project openshift-logging
$ oc delete -f clusterlogforwarder.yaml
$ oc delete -f rbac.yaml
$ oc delete secret lokistack-gateway-bearer-token
$ oc delete -f lokistack-gateway.yaml
$ oc delete secret loki-bucket
$ oc delete -f clusterlogging.yaml

$ oc project logging-test
$ oc delete -f minio-tenant.yaml
$ oc adm policy remove-scc-from-user nonroot -z minio
$ oc delete sa minio
$ oc delete secret console-secret
$ oc delete secret minio-creds-secret 
$ oc delete project logging-test
```

# debug

```
$ curl -H "Authorization: Bearer sha256~<token>" -G -s  "http://lokistack-dev-gateway-http.openshift-logging.svc:8080/api/logs/v1/application/loki/api/v1/labels"

$ curl -H "Authorization: Bearer sha256~<token>" -G -s  "http://lokistack-dev-gateway-http.openshift-logging.svc:8080/api/logs/v1/application/loki/api/v1/query" --data-urlencode 'query={log_type="application"}' | jq
$ curl -H "Authorization: Bearer sha256~<token>" -G -s  "http://lokistack-dev-gateway-http.openshift-logging.svc:8080/api/logs/v1/application/loki/api/v1/query_range" --data-urlencode 'query={log_type="application"}' --data-urlencode 'step=5s' | jq
$ curl -H "Authorization: Bearer sha256~<token>" -G -s  "http://lokistack-dev-gateway-http.openshift-logging.svc:8080/api/logs/v1/application/loki/api/v1/query_range" --data-urlencode 'query={log_type="infra"}' --data-urlencode 'step=5s' | jq
$ curl -H "Authorization: Bearer sha256~<token>" -G -s  "http://lokistack-dev-gateway-http.openshift-logging.svc:8080/api/logs/v1/application/loki/api/v1/query" --data-urlencode 'query={log_type="audit"}' | jq
```
