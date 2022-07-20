# ocp-logging-test-based-loki-vector

OpenShift Logging Test based on Loki and Vector

## Prerequisites

- Red Hat OpenShift 4.10
- Red Hat OpenShift Logging Operator 5.4.3
- Loki Operator 5.4.3-36
- Minio Operator 4.4.25

## [Deploy MinIO](https://docs.min.io/minio/k8s/openshift/deploy-minio-on-openshift.html)

```
$ oc new-project logging-storage

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

### Debug MinIO

```
$ oc run mc -it --rm --image=docker.io/minio/mc --command -- /bin/sh

sh-4.4# mc alias set minio http://minio.logging-storage.svc.cluster.local admin adminadmin --api S3v4
mc: Configuration written to `/root/.mc/config.json`. Please update your access credentials.
mc: Successfully created `/root/.mc/share`.
mc: Initialized share uploads `/root/.mc/share/uploads.json` file.
mc: Initialized share downloads `/root/.mc/share/downloads.json` file.
Added `minio` successfully.

sh-4.4# mc ls minio
[2022-07-20 04:33:33 UTC]     0B loki-bucket/
```

## [Deploying Loki with Gateway](https://github.com/grafana/loki/tree/main/operator)

### [Create Object Storage Credential](https://github.com/grafana/loki/blob/main/operator/docs/lokistack/object_storage.md#minio)

```
$ oc project openshift-logging
$ oc create secret generic lokistack-dev-minio \
    --from-literal=bucketnames=loki-bucket \
    --from-literal=access_key_id=admin \
    --from-literal=access_key_secret=adminadmin \
    --from-literal=endpoint=http://minio.logging-storage.svc.cluster.local
```

### Deploy Loki

```
$ oc apply -f lokistack-gateway.yaml
```

## [Fowarding Logs to Gateway](https://github.com/grafana/loki/blob/main/operator/docs/user-guides/forwarding_logs_to_gateway.md#openshift-logging)

```
$ oc apply -f clusterlogging-vector.yaml

$ oc apply -f logcollector-rbac.yaml

$ oc -n openshift-logging create secret generic lokistack-gateway-bearer-token \
  --from-literal=token="$(oc extract secret/$(oc get sa logcollector -ojsonpath='{.secrets[*]}' | jq -r 'select(.name | test("^logcollector-token-.+")) | .name') --keys=token --to=-)" \
  --from-literal=ca-bundle.crt="$(oc get cm lokistack-dev-ca-bundle -o json | jq -r '.data."service-ca.crt"')"

$ oc apply -f clusterlogforwarder.yaml
```

### Debug Loki

```
$ TOKEN=<ocp-user-token>

$ curl -H "Authorization: Bearer ${TOKEN}" -G -k "https://lokistack-dev-gateway-http.openshift-logging.svc:8080/api/logs/v1/application/loki/api/v1/labels" | jq
$ curl -H "Authorization: Bearer ${TOKEN}" -G -s -k "https://lokistack-dev-gateway-http.openshift-logging.svc:8080/api/logs/v1/application/loki/api/v1/query_range" --data-urlencode 'query={log_type="application"}' | jq
$ curl -H "Authorization: Bearer ${TOKEN}" -G -s -k "https://lokistack-dev-gateway-http.openshift-logging.svc:8080/api/logs/v1/infrastructure/loki/api/v1/query_range" --data-urlencode 'query={log_type="infrastructure"}' | jq
```

## [Deploy Grafana Dashboard](https://github.com/grafana/loki/blob/main/operator/docs/user-guides/howto_connect_grafana.md#using-the-gateway-with-openshift-based-authentication)

```
$ oc apply -f addon_grafana_gateway_ocp_oauth.yaml
```

## Cleanup

```
$ oc project openshift-logging

$ oc delete -f addon_grafana_gateway_ocp_oauth.yaml

$ oc delete -f clusterlogforwarder.yaml
$ oc delete -f clusterlogging-vector.yaml
$ oc delete -f logcollector-rbac.yaml
$ oc delete secret lokistack-gateway-bearer-token

$ oc delete -f lokistack-gateway.yaml
$ oc delete secret lokistack-dev-minio

$ oc project logging-storage
$ oc delete -f minio-tenant.yaml
$ oc delete pvc --all
$ oc adm policy remove-scc-from-user nonroot -z minio
$ oc delete sa minio
$ oc delete secret console-secret
$ oc delete secret minio-creds-secret 
$ oc delete project logging-storage
```
