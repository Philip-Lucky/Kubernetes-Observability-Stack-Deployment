#  Kubernetes Observability Stack (K3s + Prometheus + Grafana)

A production-grade Kubernetes monitoring solution. 

##  Architecture
* **Kubernetes Distribution:** K3s (Single-node)
* **Package Manager:** Helm 3
* **Monitoring Stack:** Prometheus, Grafana, kube-state-metrics, node-exporter
* **Alerting:** Alertmanager (Slack Integration)
* **Target Environment:** AWS EC2 (`t3.medium` / 4GB RAM)

##  Prerequisites
* An AWS EC2 instance with at least 4GB of RAM running Ubuntu or Amazon Linux.
* SSH access to the instance.
* **Port 8080** opened in your AWS Security Group to access the Grafana UI.
* A free Slack Incoming Webhook URL (for Alertmanager).

---

##  Deployment Guide

### Phase 1: Cluster Setup (K3s & Helm)
Install a lightweight Kubernetes cluster and the Helm package manager.

1. **Install K3s:**
   ```bash
   curl -sfL [https://get.k3s.io](https://get.k3s.io) | sh -


 2. **Configure Kubeconfig:**


```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
Install Helm:

```bash
curl [https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3) | bash
Phase 2: Deploy the Observability Stack
We apply strict memory limits to prevent the stack from crashing the 4GB instance.


 
3. **Add the Helm Repository:**

```bash
helm repo add prometheus-community [https://prometheus-community.github.io/helm-charts](https://prometheus-community.github.io/helm-charts)
helm repo update

4. **Create the custom-values.yaml file:**
Note: Replace YOUR_SLACK_WEBHOOK_URL with your actual Slack URL before deploying.

```YAML
alertmanager:
  enabled: true
  config:
    route:
      receiver: 'slack-notifications'
      group_by: ['alertname', 'namespace']
    receivers:
    - name: 'slack-notifications'
      slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK_URL'
        send_resolved: true
        title: 'K8s Alert: {{ .Status | toUpper }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'
prometheus:
  prometheusSpec:
    retention: 2d
    resources:
      requests:
        memory: 512Mi
      limits:
        memory: 1Gi
grafana:
  resources:
    requests:
      memory: 128Mi
    limits:
      memory: 256Mi

5. **Deploy via Helm:**

```bash
helm install observability prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f custom-values.yaml

<img width="1359" height="588" alt="deployment ruuning" src="https://github.com/user-attachments/assets/b457c1a7-8c55-499a-a5d5-87803a581b61" />


### Phase 3: Access & Visualize (Grafana)

Port-Forward the UI:

```bash
kubectl port-forward svc/observability-grafana 8080:80 -n monitoring --address 0.0.0.0
Log In: Navigate to http://<YOUR_EC2_PUBLIC_IP>:8080 (Default credentials: admin / prom-operator).
<img width="1584" height="629" alt="inbound rules 01" src="https://github.com/user-attachments/assets/9b722a54-efc9-4533-833d-8ecdb8d221a7" />


Import Dashboard: Import Dashboard ID 15661 for a comprehensive Kubernetes cluster overview. Select observability-prometheus as the data source.
<img width="1583" height="730" alt="grafana 01" src="https://github.com/user-attachments/assets/1799e2b2-bda9-493b-ab78-38e36924df40" />
<img width="1585" height="735" alt="grafana 02" src="https://github.com/user-attachments/assets/d83c8a65-83cc-4b19-8f89-7368be5f4311" />
<img width="1585" height="739" alt="grafana 03" src="https://github.com/user-attachments/assets/45515487-3430-43a1-bfa6-82a28cf49f74" />

Testing the Setup
To verify metrics are flowing, deploy a dummy application and scale it up to trigger resource usage:

```bash
kubectl create namespace demo-app
kubectl create deployment nginx-dummy --image=nginx -n demo-app
kubectl scale deployment nginx-dummy --replicas=5 -n demo-app
<img width="1179" height="199" alt="namespce inginx app deploy" src="https://github.com/user-attachments/assets/c1caa3e6-006c-424a-8cbb-93692c9b26d6" />
<img width="1209" height="165" alt="namespace inginx app deploy" src="https://github.com/user-attachments/assets/e1ed919b-7110-4f42-9d39-b82ff9f54544" />
<img width="1578" height="735" alt="new grafana showing the nginx app" src="https://github.com/user-attachments/assets/5c06d092-e92f-48d2-8613-7fdbc4d77165" />
<img width="1581" height="637" alt="new grafana" src="https://github.com/user-attachments/assets/5f262fb5-d39c-49f3-ad94-798e4a39bbe7" />
<img width="1589" height="729" alt="new grafana 03" src="https://github.com/user-attachments/assets/528edd83-f536-4c08-8b1d-df61003c3600" />


