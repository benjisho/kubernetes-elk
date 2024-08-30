# **Deploys the ELK Stack on Kubernetes**

This guide provides detailed instructions for deploying the ELK stack on Kubernetes, using either raw Kubernetes manifests with `kubectl` or Helm with customizable templates. Each method has its own advantages, depending on your deployment needs.

## **Table of Contents**
1. [Project Structure](#project-structure)
2. [Minikube Setup](#minikube-setup)
   - [Running Minikube as Root (Not Recommended)](#running-minikube-as-root-not-recommended)
3. [Creation of Resources](#3-creation-of-resources)
4. [Enabling HTTPS with NGINX](#enabling-https-with-nginx)
   - [4.1 Generate a Self-Signed SSL Certificate](#41-generate-a-self-signed-ssl-certificate)
   - [4.2 Deploy NGINX with `kubectl`](#42-deploy-nginx-with-kubectl)
   - [4.3 Deploy NGINX with Helm](#43-deploy-nginx-with-helm)
   - [4.4 Access Kibana via HTTPS](#44-access-kibana-via-https)
5. [Installation and Deployment](#4-installation-and-deployment)
   - [Deploy with `kubectl`](#deploy-with-kubectl)
   - [Deploy with Helm](#deploy-with-helm)
6. [Maintenance and Monitoring](#5-maintenance-and-monitoring)
7. [Troubleshooting and Updates](#6-troubleshooting-and-updates)
8. [Deletion and Clean-Up](#7-deletion-and-clean-up)

---

## **1. Project Structure**

```bash
kubernetes/
├── option-1-helm/
│   ├── templates/
│   │   ├── elasticsearch.yaml
│   │   ├── kibana.yaml
│   │   ├── logstash.yaml
│   │   ├── nginx.yaml         # New NGINX deployment
│   │   ├── setup.yaml
│   └── values.yaml
└── option-2-kubectl/
    ├── elasticsearch.yaml
    ├── kibana.yaml
    ├── logstash.yaml
    ├── nginx.yaml              # New NGINX deployment
    ├── setup.yaml
config/
└── ssl/
    ├── nginx.conf               # NGINX configuration file
    ├── self-signed.crt          # Self-signed certificate
    ├── self-signed.key          # Private key for the certificate

```

The project is structured into two main deployment options:

- **`kubernetes/option-2-kubectl/`**: Contains raw Kubernetes manifests for deploying the ELK stack using `kubectl`.
- **`kubernetes/option-1-helm/`**: Contains a Helm chart for deploying the ELK stack with customizable templates.

### **Directory Contents**

#### **kubectl Deployment (`kubernetes/option-2-kubectl/`)**
- **`elasticsearch.yaml`**: StatefulSet for Elasticsearch.
- **`logstash.yaml`**: Deployment for Logstash.
- **`kibana.yaml`**: Deployment for Kibana.
- **`nginx.yaml`**: Deployment and service for NGINX with HTTPS support.
- **`setup.yaml`**: Job for initializing Elasticsearch users and roles.

#### **Helm Deployment (`kubernetes/option-1-helm/`)**
- **`Chart.yaml`**: Metadata about the Helm chart.
- **`values.yaml`**: Default configuration values.
- **`templates/`**: Helm templates for generating Kubernetes manifests dynamically.
  - **`nginx.yaml`**: NGINX deployment template for HTTPS support.

---

## **2. Minikube Setup**

If you are using Minikube for your Kubernetes environment, follow these steps to set up and start your Minikube cluster:

### **Install Minikube**
If you don't already have Minikube installed, you can install it by following the [official Minikube installation guide](https://minikube.sigs.k8s.io/docs/start/).

### **Start Minikube**
Start the Minikube cluster with the Docker driver:

```bash
minikube start --driver=docker
```

### **Verify the Cluster Status**
Check the status of the Minikube cluster to ensure it's running:

```bash
minikube status
```

### **Set Up Docker Environment (Optional)**
If you want to use Minikube's Docker daemon, configure your shell:

```bash
eval $(minikube -p minikube docker-env)
```

This allows you to build Docker images directly within the Minikube environment.


---

### **Running Minikube as Root (Not Recommended)**

#### **Warning**

Running Minikube as root is generally not recommended due to the following risks:

1. **Security Risks**: Running as root can expose your system to security vulnerabilities, especially if a container is compromised, as it could potentially gain elevated privileges.
2. **File Permissions Issues**: Files created by Minikube or Docker while running as root may have restrictive permissions, causing issues for non-root users trying to access or manage these files later.
3. **Driver Restrictions**: Some Minikube drivers, like the Docker driver, have specific restrictions when running as root. For instance, Minikube may refuse to start, or you may need to use the `--force` flag, which can lead to unexpected behavior.

#### **Steps to Run Minikube as Root**

If you still need to run Minikube as root, here are the steps:

1. **Start Minikube with the `none` Driver**:
   The `none` driver runs Kubernetes directly on the host without a VM, which is compatible with root privileges.
   ```bash
   minikube start --driver=none
   ```

2. **Forcing Docker Driver (If Absolutely Necessary)**:
   If you must use the Docker driver as root, you can bypass the restriction with the `--force` flag:
   ```bash
   minikube start --driver=docker --force
   ```

3. **Verify Minikube Status**:
   Ensure that Minikube is running:
   ```bash
   minikube status
   ```

4. **Running Kubernetes Commands**:
   Since Minikube is running as root, you’ll need to run all subsequent `kubectl` and Docker commands as root or with `sudo`.

#### **Risks and Considerations**

- **Security**: Consider using a less privileged user or setting up a separate environment for running Minikube to minimize security risks.
- **Maintenance**: Be aware of potential maintenance issues, such as cleaning up resources and managing permissions for files and directories created by root.

---

## **3. Creation of Resources**

Before deploying the ELK stack, you need to create the necessary Kubernetes resources:

### **Secrets and ConfigMaps**

#### **Create Secrets**

Create the `elastic-credentials` secret to store passwords for Elasticsearch, Kibana, and Logstash:

```bash
kubectl create secret generic elastic-credentials \
  --from-literal=elastic_password=<your-elastic-password> \
  --from-literal=kibana_password=<your-kibana-password> \
  --from-literal=logstash_password=<your-logstash-password>
```

#### **Create ConfigMaps**

Create the `logstash-config` ConfigMap with the Logstash configuration:

```bash
kubectl create configmap logstash-config --from-file=logstash/config/logstash.yml
```

Create the `setup-scripts` ConfigMap for the setup job:

```bash
kubectl create configmap setup-scripts --from-file=setup/entrypoint.sh --from-file=setup/lib.sh
```
---

## **4. Enabling HTTPS with NGINX**

This section explains how to configure and deploy NGINX as a reverse proxy to provide HTTPS access to Kibana. NGINX will handle SSL termination and redirect traffic from port 443 to Kibana's default port 5601.

### **4.1 Generate a Self-Signed SSL Certificate**

Generate a self-signed SSL certificate using the following command:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./config/ssl/self-signed.key -out ./config/ssl/self-signed.crt
```

### **4.2 Deploy NGINX with `kubectl`**

1. **Add NGINX Configuration and Secrets:**
   - The `nginx.yaml` in `kubernetes/option-2-kubectl/` includes the NGINX deployment and service configuration. It also references the SSL certificate and key stored in a Kubernetes secret.

2. **Deploy NGINX:**
   - Apply the NGINX deployment using `kubectl`:
   ```bash
   kubectl apply -f kubernetes/option-2-kubectl/nginx.yaml
   ```

### **4.3 Deploy NGINX with Helm**

1. **NGINX Configuration in Helm:**
   - The Helm chart includes a `nginx.yaml` template that defines the NGINX deployment and service with HTTPS support. The SSL certificate and key are referenced in a Kubernetes secret within the Helm chart.

2. **Deploy NGINX with Helm:**
   - Install or upgrade the Helm chart to deploy NGINX:
   ```bash
   helm upgrade --install elk-stack kubernetes/option-1-helm/
   ```

### **4.4 Access Kibana via HTTPS**

1. **Forward the NGINX Service Port:**
   - To access Kibana securely via HTTPS, forward the NGINX service port to your local machine:
   ```bash
   kubectl port-forward service/nginx 443:443
   ```

2. **Open Kibana in Your Browser:**
   - Navigate to `https://localhost` in your browser.

---

## **5. Installation and Deployment**

### **Deploy with `kubectl`**

#### **Structure and Files**

The `kubectl` deployment is defined by raw Kubernetes manifests located in the `kubernetes/option-2-kubectl/` directory.

#### **How to Deploy**

Deploy the ELK stack using `kubectl`:

```bash
kubectl apply -f kubernetes/option-2-kubectl/
```

### **Accessing Kibana**

To access the Kibana UI:

1. Forward the Kibana service port to your local machine:
   ```bash
   kubectl port-forward service/kibana 5601:5601
   ```
2. Open your browser and go to `http://localhost:5601`.

#### **When to Choose This Option**
- **Simplicity**: Best for straightforward deployments without much customization.
- **Fine-Grained Control**: Offers full control over every detail in the manifests.
- **Smaller Deployments**: Ideal for smaller environments or proof-of-concept setups.

### **Deploy with Helm**

#### **Structure and Files**

The Helm deployment is defined by a Helm chart located in the `kubernetes/option-1-helm/` directory.

#### **How to Deploy**

##### **Basic Deployment with Default Values**

Deploy using the default values in `values.yaml`:

```bash
helm install elk-stack kubernetes/option-1-helm
```

##### **Custom Deployment with Overrides**

Deploy with custom values by overriding specific settings:

```bash
helm install elk-stack kubernetes/option-1-helm --set elasticVersion=7.12.0 --set elasticsearch.storage=2Gi
```

### **Accessing Kibana**

To access the Kibana UI:

1. Forward the Kibana service port to your local machine:
   ```bash
   kubectl port-forward service/kibana 5601:5601
   ```
2. Open your browser and go to `http://localhost:5601`.

#### **When to Choose This Option**
- **Scalability**: Suitable for large and complex environments.
- **Reusability**: Ideal for deploying multiple instances with varying configurations.
- **Customization**: Offers extensive customization through Helm's templating system.

---

## **6. Maintenance and Monitoring**

### **Monitoring the Deployment**

After deployment, monitor the status of the pods:

```bash
kubectl get pods
```

Ensure that all services (`elasticsearch`, `kibana`, `logstash`, `setup`) are running smoothly.

### **Checking Logs**

Verify the logs for each component to ensure they are functioning correctly:

```bash
kubectl logs elasticsearch-0
kubectl logs kibana-7f58c79487-5fvvv
kubectl logs logstash-6f98c4c97f-w9kpw
kubectl logs setup-gdp4v
```

---

## **7. Troubleshooting and Updates**

### **Troubleshoot Deployment Issues**

If you encounter deployment issues:

1. **Immutable Fields**: If changes to the pod template cause errors:
   ```bash
   kubectl delete job setup
   ```

2. **Reapply Helm Chart**: After making necessary changes:
   ```bash
   helm upgrade --install elk-stack ./kubernetes/option-1-helm/
   ```

### **Updating Resources**

To update configurations or secrets, modify the relevant YAML files or Helm values, then reapply using `kubectl` or Helm.

---

## **8. Deletion and Clean-Up**

### **Delete the ELK Stack**

To remove the ELK stack deployment:

#### **Using `kubectl`**

```bash
kubectl delete -f kubernetes/option-2-kubectl/
```

#### **Using Helm**

```bash
helm uninstall elk-stack
```

### **Clean Up Resources**

After deletion, ensure all related resources (pods, services, ConfigMaps, secrets) are also removed:

```bash
kubectl get all
```

Delete any remaining resources manually if needed.