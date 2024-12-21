# Kubernetes(AWS EKS) Observability with EFK, Prometheus, Grafana, and Jaeger

**This project implements a Kubernetes observability stack to monitor, log, and trace a multi-service Node.js application running on AWS EKS.**

## Architecture
![Project Architecture](images/architecture.gif)

### Step 1: Launch and connect to the EC2 Instance:
- Select the latest Ubuntu AMI.
- Choose a t2.medium or larger instance type (for sufficient resources).
- Assign a security group allowing: 
    - SSH (Port `22`)
    - HTTP (Port `80`)
    - NodeJS app (Port `3001` & `3002`)
    - Grafana UI (Port `3000`)
    - Prometheus (Port `9090`)
    - Alertmanager (Port `9093`)
    - Kibana (Port `5601`)
    - Jaeger (Port `16686`)
    - `3000-30002` (Kibana, Grafana, Jaeger)
    - NodePort range (`30000-32767`)
- Connect to the instance.
    - `ssh -i "your-key.pem" ubuntu@<instance-public-ip>`

### Step 2: Update the System & Install Pre-requisites
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar unzip -y

# Install Docker
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ubuntu

# Install Kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Aws CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
rm awscliv2.zip

# Configure awscli (Obtain ```Access-Key-ID``` and ```Secret-Access-Key``` from the AWS Management Console).
aws configure

# Install eksctl
curl -LO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz"
tar -xzf eksctl_$(uname -s)_amd64.tar.gz
sudo mv eksctl /usr/local/bin
eksctl version
rm eksctl_$(uname -s)_amd64.tar.gz

# Install Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### Step 3: Set Up the EKS Cluster
- Create an EKS Cluster.
```bash
eksctl create cluster --name=observability \ 
                    --region=us-east-1 \
                    --zones=us-east-1a,us-east-1b \
                    --without-nodegroup
```
- Associate an IAM OIDC (OpenID Connect) provider with the cluster, allowing the cluster to integrate with AWS IAM roles.
- This step is essential for enabling Prometheus, external-dns, ALB ingress, and other AWS services. 
```bash
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster observability \
    --approve
```
- Add a private managed node group to the cluster.
```bash
eksctl create nodegroup --cluster=observability \
                        --region=us-east-1 \
                        --name=observability-ng-private \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=3 \
                        --node-volume-size=20 \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking
```
```bash
# Update ./kube/config file to access the cluster using kubectl.
aws eks update-kubeconfig --name observability

# Verify the cluster.
kubectl get nodes
```

### Step 4: Write Custom Metrics

Please take a look at `application/service-a/index.js` file to learn more about custom metrics. below is the brief overview
- **Express Setup**: Initializes an Express application and sets up logging with Morgan.
- **Logging with Pino**: Defines a custom logging function using Pino for structured logging.
- **Prometheus Metrics with prom-client**: Integrates Prometheus for monitoring HTTP requests using the prom-client library:
    - `http_requests_total`: counter
    - `http_request_duration_seconds`: histogram
    - `http_request_duration_summary_seconds`: summary
    - `node_gauge_example`: gauge for tracking async task duration

**Basic Routes:**
- `/` : Returns a "Running" status.
- `/healthy`: Returns the health status of the server.
- `/serverError`: Simulates a 500 Internal Server Error.
- `/notFound`: Simulates a 404 Not Found error.
- `/logs`: Generates logs using the custom logging function.
- `/crash`: Simulates a server crash by exiting the process.
- `/example`: Tracks async task duration with a gauge.
- `/metrics`: Exposes Prometheus metrics endpoint.
- `/call-service-b`: To call service b & receive data from service b

### Step 5: Build and Push Docker Images
```bash
cd application/service-a
docker build -t <dockerhub-username>/service-a:latest .

cd ../service-b
docker build -t <dockerhub-username>/service-b:latest .

# Push images to Docker Hub
docker login
docker push <dockerhub-username>/demoservice-a:latest
docker push <dockerhub-username>/demoservice-b:latest
```

### Step 6: Deploy to Kubernetes
- Review the Kubernetes manifest files located in `kubernetes-manifest/`.
- Apply the Kubernetes manifest files to your cluster by running:
```bash
kubectl create namespace dev
kubectl apply -k kubernetes-manifest/ -n dev

# Verify that pods and services are running:
kubectl get pods -n dev
kubectl get svc -n dev
kubectl get all -n dev
```

### Step 7: Set Up the ALB Ingress Controller in AWS EKS
```bash
# ALB Ingress Controller requires IAM permissions to manage AWS resources like load balancers. Create an IAM policy and role that grants these permissions.

# Download IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

# Create IAM Policy
aws iam create-policy \
    --policy-name ALBIngressControllerPolicy \
    --policy-document file://iam_policy.json

# Create IAM Role
eksctl create iamserviceaccount \
    --cluster=observability \
    --namespace=alb-resource \
    --name=alb-ingress-controller \
    --role-name ALBIngressControllerRole \
    --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/ALBIngressControllerPolicy \
    --approve

# Add helm repo
helm repo add eks https://aws.github.io/eks-charts`
helm repo update

# Install the ALB Ingress Controller
helm install alb-ingress-controller eks/alb-ingress-controller \            
    -n alb-resource \
    --set clusterName=observability \
    --set serviceAccount.create=false \
    --set serviceAccount.name=alb-ingress-controller \
    --set ingressClass=alb \

# Verify Installation
kubectl get pods -n kube-system
kubectl get deployment -n alb-resource alb-ingress-controller

# Apply the Ingress resource to specify how traffic should be routed to your services.
kubectl apply -f nodejs-alb-ingress.yaml

# After the ALB is created, it will be accessible through the DNS name provided by AWS. You can find this DNS name in the Load Balancer details page in the AWS Console.
# To access your service, simply open the URL in a browser: http://<ALB-DNS-Name>/
# You should be routed to your service (e.g., Service A or Service B, based on the paths specified in your Ingress configuration).
curl http://<ALB-DNS>/service-a
curl http://<ALB-DNS>/service-b
```

### Step 8: Expose the Application
```bash
# Identify the service type
kubectl get svc -n dev

# For a LoadBalancer service, note the External IP
# Access the Service Through the ALB

# For NodePort service, access the app using
http://<EC2-Public-IP>:<NodePort>
```

### Step 9: Configure AWS EBS Storage Class using Helm
```bash
# Add the AWS EBS CSI Helm Repository
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

# Install the EBS CSI Driver in your cluster to manage EBS volumes.
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
--namespace ebs-resource --create-namespace

# Verify Installation
kubectl get pods -n ebs-resource | grep ebs

# Apply the Storage Class
kubectl apply -f gp2-storageclass.yaml

# Verify the Storage Class
kubectl get storageclass
```

### Step 10: Configure Cluster Monitoring (Prometheus, Grafana, AlertManager): 
```bash
# Use Helm to install Prometheus and Grafana
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus Stack
# This installs: Prometheus, Grafana, AlertManager, Node Exporter, Kube-State-Metrics 
helm install prometheus prometheus-community/kube-prometheus-stack \
                            -n monitoring --create-namespace \
                            -f values-prometheus.yaml

# Check pods and services
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get all -n monitoring

# Verify the PersistentVolumeClaim (PVC): You should see a PVC created for Prometheus. It will dynamically provision an EBS volume.
kubectl get pvc -n monitoring

# Apply Alertmanager Configuration
# Before configuring Alertmanager, we need credentials to send emails. For this project, I am using Gmail, but any SMTP provider like AWS SES can be used. so please grab the credentials for that.
# Open your Google account settings and search App password & create a new password & put the password in `alerts-alertmanager-servicemonitor-manifest/email-secret.yml`
# Add your email id in the `alerts-alertmanager-servicemonitor-manifest/alertmanagerconfig.yml`
# HighCpuUsage: Triggers a warning alert if the average CPU usage across instances exceeds 50% for more than 5 minutes.
# PodRestart: Triggers a critical alert immediately if any pod restarts more than 2 times.
kubectl apply -k alerts-alertmanager-servicemonitor-manifest/`
kubectl rollout restart deployment prometheus-stack-kube-prometheus-alertmanager -n monitoring

# Expose Prometheus and Grafana
kubectl port-forward svc/prometheus-operated -n monitoring 9090:9090 --address 0.0.0.0
kubectl port-forward svc/prometheus-stack-grafana -n monitoring 3000:80 --address 0.0.0.0
```
- Access Prometheus at `http://<EC2-Public-IP>:9090`.
- Access Grafana at `http://<EC2-Public-IP>:3000`.
- Default username: `admin`, Default password: `prom-operator` (or retrieve it with `kubectl get secret`):
    - `kubectl get secret prometheus-stack-grafana -n dev -o jsonpath="{.data.admin-password}" | base64 --decode`

**Configure Data Source:**
- Grafana should auto-detect Prometheus as a data source (If not, manually add it).
- Navigate to Configuration > Data Sources.
- Click Add Data Source.
- Select Prometheus and set the URL to Prometheus service:
    - `http://prometheus-stack-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090`
- Save and test the connection.

**Import Kubernetes Dashboards**
- Log into Grafana.
- Import a Kubernetes monitoring dashboard:
    - Go to Dashboards → Import.
    - Use the dashboard ID `315` (Kubernetes cluster monitoring) from Grafana's public repository.
- View cluster metrics such as:
    - CPU and memory usage.
    - Pod resource consumption.
    - Node health.

**Verify Alert Configurations:**
- Wait for 4-5 minutes and then check the Prometheus UI to confirm that the custom metrics implemented in the Node.js application are available:
    - `http_requests_total`: counter
    - `http_request_duration_seconds`: histogram
    - `http_request_duration_summary_seconds`: summary
    - `node_gauge_example`: gauge for tracking async task duration.

### Step 11: Configure Logging with EFK Stack (Elasticsearch, Fluent Bit, Kibana):
```bash
#  Install Elasticseacrch.
helm repo add elastic https://helm.elastic.co
helm repo update

helm install elasticsearch elastic/elasticsearch \
                                -n logging \
                                --create-namespace \
                                --set persistence.enabled=true \
                                --set persistence.existingClaim="" \
                                -f values-elasticsearch.yaml

kubectl get pods -n logging
# Wait until all Elasticsearch pods are running.

# Verify Persistent Volume Claims (PVCs): The output will show PVCs created for Elasticsearch pods and bound to EBS volumes.
kubectl get pvc -n monitoring
# Verify Persistent Volumes (PVs):
kubectl get pv

# Install Fluent Bit to collect logs and forward them to Elasticsearch.
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

helm install fluent-bit fluent/fluent-bit -n logging --create-namespace \
--set backend.type=es \
--set backend.es.host=elasticsearch-master.logging.svc.cluster.local \
--set backend.es.port=9200

# Verify Fluent Bit Logs:
kubectl logs -l app.kubernetes.io/name=fluent-bit -n logging

# Install Kibana for visualizing logs.
helm install kibana elastic/kibana -n logging
kubectl port-forward -n logging svc/kibana-kibana 5601:5601 --address 0.0.0.0
```
- Access Kibana at: `http://<EC2-Public-IP>:5601`
- Set up an index pattern for Elasticsearch (e.g., `fluent-bit-*`) to view and analyze logs.

### Step 12: Configure Tracing with Jaeger and OpenTelemetry
```bash
# Install Jaeger
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update
helm install jaeger jaegertracing/jaeger -n tracing --create-namespace
kubectl port-forward -n tracing svc/jaeger-query 16686:16686 --address 0.0.0.0
```
- Access Jaeger UI at: `http://<EC2-Public-IP>:16686`.

### Step 13: Testing Workflow

- **Application Testing**
    - Using the LoadBalancer or NodePort, test the `/` and `/healthy` endpoints to verify the application is running:
    - `curl http://<LoadBalancer-IP>/`
    - `curl http://<LoadBalancer-IP>/healthy`
- **Metrics Testing**
    - View application metrics at `/metrics` endpoint: `curl http://<LoadBalancer-IP>/metrics`
    - Confirm custom metrics (e.g., `http_requests_total`) in Prometheus.
    - Simulate a Pod Failure: Delete Prometheus pods to verify that data persists.
        - `kubectl delete pod <prometheus-pod-name> -n monitoring`
        - After the pod restarts, access Prometheus to ensure metrics and logs are still available.
- **Logs Testing**
    - The Node.js application already has logging integrated using Pino. Fluent Bit will automatically capture and forward these logs to Elasticsearch.
    - Access your application endpoints to generate logs: `curl http://<LoadBalancer-IP>/logs`
    - Simulate Pod Restart: Delete an Elasticsearch pod to simulate a node failure.
        - `kubectl delete pod <elasticsearch-pod-name> -n logging`.
    -  After the pod restarts, check logs in Kibana by filtering with `fluent-bit-*` to ensure the logs are still available.
    - This confirms that the EBS-backed persistent volumes retained the Elasticsearch data.
- **Tracing Testing**
    - Access `/call-service-b` to generate traces: `curl http://<LoadBalancer-IP>/call-service-b`
    - This will make a request from service-a to service-b.
    - Confirm traces in Jaeger. Open Jaeger UI: `http://<EC2-Public-IP>:16686`
    - Search for `service-a` or `service-b` in the Services dropdown.
    - Explore the spans and visualize request latencies and dependencies.
- **Alert Testing**
    - Hit the `/crash` endpoint multiple times to trigger the PodRestart alert: `curl http://<LoadBalancer-IP>/crash`.
    - Check email for notifications.
    - Alternatively, you can run the automated script `test.sh`, which will automatically send random requests to the LoadBalancer and generate metrics: `./test.sh <<LOAD_BALANCER_DNS_NAME>>`

### Step 14: Troubleshooting and Known Issues
```bash
Issue	Cause	Solution
Prometheus pods not running	Resource limitations	Allocate more memory to the cluster.
Logs missing in Kibana	Fluent Bit misconfiguration	Verify Fluent Bit Elasticsearch settings.
Jaeger traces missing	Incorrect endpoint in app configs	Check Jaeger collector endpoint URL.
```

### Step 15: Observability Dashboards
```bash
Component	    Tool	        Access URL                       Default Credentials

Logs	        Kibana	        http://<EC2-Public-IP>:5601      Not required
Metrics	        Prometheus	    http://<EC2-Public-IP>:9090      Not required
Dashboards	    Grafana	        http://<EC2-Public-IP>:3000      admin/prom-operator
Traces	        Jaeger	        http://<EC2-Public-IP>:16686     Not required
Alerts	        Alertmanager    Email Notifications              Configured via Alertmanager
```
