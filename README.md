# Install JFrog Pipelines (K8S) with Artifactory (VM) - Workaround

## 1. Deploy Pipelines Router Service

pipelines-router-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pipelines-router
spec:
  ports:
  - name: router-port
    port: 8082
    protocol: TCP
    targetPort: 8082
  publishNotReadyAddresses: true
  selector:
    app.kubernetes.io/name: pipelines
    app.kubernetes.io/instance: pipelines
    component: pipelines-pipelines-services
  sessionAffinity: None
  type: LoadBalancer
```

```bash
kubectl -n pipelines apply -f pipelines-router-service.yaml
```
## 2. Extract ExternalIP
```bash
export ROUTER_SERVICE_EXTERNAL_IP=$(kubectl get svc --namespace pipelines pipelines-router -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $ROUTER_SERVICE_EXTERNAL_IP
```
## 3. Fetch Pipelines Chart and edit statefulset

```bash
helm fetch center/jfrog/pipelines --untar 
```

Edit pipelines/templates/pipelines-statefulset.yaml

REPLACE
```yaml
            - name: JF_SHARED_NODE_IP
              valueFrom:
                fieldRef:
                  fieldPath: "status.podIP"
```
WITH
```yaml
            - name: JF_SHARED_NODE_IP
              value: <ROUTER_SERVICE_EXTERNAL_IP>
```

## 4. Deploy Pipelines chart
```bash
helm upgrade --install pipelines --namespace pipelines ./pipelines/. -f values.yaml
```
## 5. Verify Artifactory system.yaml:
```yaml
shared:
  node:
    ip: <ARTIFACTORY_PUBLIC_IP>
    port: 8082 or 80
```
