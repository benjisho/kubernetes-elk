# **Deploys the ELK Stack on Kubernetes**

This guide provides detailed instructions for deploying the ELK stack on Kubernetes, using either raw Kubernetes manifests with `kubectl` or Helm with customizable templates. Each method has its own advantages, depending on your deployment needs.

## **Table of Contents**
1. [Project Structure](#project-structure)
2. [Minikube Setup](#minikube-setup)
   - [Running Minikube as Root (Not Recommended)](#running-minikube-as-root-not-recommended)
3. [Creation of Resources](#creation-of-resources)
4. [Installation and Deployment](#installation-and-deployment)
   - [Deploy with `kubectl`](#deploy-with-kubectl)
   - [Deploy with Helm](#deploy-with-helm)
5. [Maintenance and Monitoring](#maintenance-and-monitoring)
6. [Troubleshooting and Updates](#troubleshooting-and-updates)
7. [Deletion and Clean-Up](#deletion-and-clean-up)

---

## **1. Project Structure**

The project is structured into two main deployment options:

- **`kubernetes/option-2-kubectl/`**: Contains raw Kubernetes manifests for deploying the ELK stack using `kubectl`.
- **`kubernetes/option-1-helm/`**: Contains a Helm chart for deploying the ELK stack with customizable templates.

### **Directory Contents**

#### **kubectl Deployment (`kubernetes/option-2-kubectl/`)**
- **`elasticsearch.yaml`**: StatefulSet for Elasticsearch.
- **`logstash.yaml`**: Deployment for Logstash.
- **`kibana.yaml`**: Deployment for Kibana.
- **`setup.yaml`**: Job for initializing Elasticsearch users and roles.

#### **Helm Deployment (`kubernetes/option-1-helm/`)**
- **`Chart.yaml`**: Metadata about the Helm chart.
- **`values.yaml`**: Default configuration values.
- **`templates/`**: Helm templates for generating Kubernetes manifests dynamically.

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

## **4. Installation and Deployment**

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

## **5. Maintenance and Monitoring**

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

## **6. Troubleshooting and Updates**

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

## **7. Deletion and Clean-Up**

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