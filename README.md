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
values.yaml
```yaml
existingSecret: ""

postgresql:
  postgresqlPassword: somepassword
  image:
    tag: 12.3.0-debian-10-r71
unifiedUpgradeAllowed: true
databaseUpgradeReady: true

pipelines:
  # Generate new one usin openssl rand -hex 32
  masterKey: 689ccf61bace3ceed76059b1a13723274f625de7533dbcea487672472ce21c71
  # Generate new one using uuidgen | tr '[:upper:]' '[:lower:]'
  authToken: 11de15c0-42cf-4463-a117-b02ba9adb620
  serviceId: jfpip@1234567898
  joinKey: 
  jfrogUrl: 
  jfrogUrlUI: 
  msg:
    uiUserPassword: somepassword
  www:
    ingress:
      enabled: true
      annotations:
        ingress.kubernetes.io/force-ssl-redirect: "false"
        ingress.kubernetes.io/proxy-body-size: "0"
        ingress.kubernetes.io/proxy-read-timeout: "600"
        ingress.kubernetes.io/proxy-send-timeout: "600"
        kubernetes.io/ingress.class: nginx
        nginx.ingress.kubernetes.io/proxy-body-size: "0"
      path: /
      hosts:
        - pipelines.example.com

  api:
    service:
      type: LoadBalancer
    ingress:
      enabled: true
      annotations:

        ingress.kubernetes.io/force-ssl-redirect: "false"
        ingress.kubernetes.io/proxy-body-size: "0"
        ingress.kubernetes.io/proxy-read-timeout: "600"
        ingress.kubernetes.io/proxy-send-timeout: "600"
        kubernetes.io/ingress.class: nginx
        nginx.ingress.kubernetes.io/proxy-body-size: "0"

      path: /
      hosts:
        - pipelines-api.example.com


rabbitmq:
  enabled: true
  replicaCount: 1
  auth:
    username: rmqadmin
    password: password
  ingress:
    enabled: true
    hostname: pipelines-msg.example.com
    path: /
    tls: false

    annotations:
      ingress.kubernetes.io/force-ssl-redirect: "false"
      ingress.kubernetes.io/proxy-body-size: "0"
      ingress.kubernetes.io/proxy-read-timeout: "600"
      ingress.kubernetes.io/proxy-send-timeout: "600"
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/proxy-body-size: "0"
      nginx.ingress.kubernetes.io/force-ssl-redirect: "false"
```
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
