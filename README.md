# **Deploys the ELK Stack on Kubernetes**

This guide provides detailed instructions for deploying the ELK stack on Kubernetes, using either raw Kubernetes manifests with `kubectl` or Helm with customizable templates. Each method has its own advantages, depending on your deployment needs.

## **Table of Contents**
1. [Project Structure](#project-structure)
2. [Creation of Resources](#creation-of-resources)
3. [Installation and Deployment](#installation-and-deployment)
   - [Deploy with `kubectl`](#deploy-with-kubectl)
   - [Deploy with Helm](#deploy-with-helm)
4. [Maintenance and Monitoring](#maintenance-and-monitoring)
5. [Troubleshooting and Updates](#troubleshooting-and-updates)
6. [Deletion and Clean-Up](#deletion-and-clean-up)

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

## **2. Creation of Resources**

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

## **3. Installation and Deployment**

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

## **4. Maintenance and Monitoring**

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

## **5. Troubleshooting and Updates**

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

## **6. Deletion and Clean-Up**

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