## Deploying the ELK Stack on Kubernetes

You can deploy the ELK stack on Kubernetes using two different approaches: `kubectl` with raw Kubernetes manifests or Helm with customizable templates. Each approach has its advantages, depending on your needs and the complexity of your deployment.

### 1. Deploy with `kubectl`

#### Structure and Files

The `kubectl` option is located under the `kubernetes/option-2-kubectl/` directory. This directory contains raw Kubernetes manifests that describe the deployment of Elasticsearch, Logstash, Kibana, and a one-time setup job.

Here are the key files:

- **`elasticsearch.yaml`**: Defines a `StatefulSet` for Elasticsearch, ensuring persistent storage for your data and enabling easier scaling.
- **`logstash.yaml`**: Defines a `Deployment` for Logstash, responsible for collecting, processing, and forwarding logs to Elasticsearch.
- **`kibana.yaml`**: Defines a `Deployment` for Kibana, which provides a web interface for visualizing data stored in Elasticsearch.
- **`setup.yaml`**: Defines a `Job` that runs a one-time setup script to initialize users and roles in Elasticsearch.

#### What It Does

This approach uses predefined Kubernetes resources without any templating. The files directly specify how each component should be deployed and connected, including configuration, secrets, and storage.

#### How to Deploy

```sh
kubectl apply -f kubernetes/option-2-kubectl/
```

This command will create all the necessary Kubernetes resources for the ELK stack.

#### When to Choose This Option

- **Simplicity**: If you need a straightforward deployment without much customization.
- **Smaller Deployments**: Ideal for smaller environments or proof-of-concept deployments where advanced templating isn't required.
- **Fine-Grained Control**: You have full control over every detail in the manifests, which is useful if you need to make very specific adjustments.

### 2. Deploy with Helm

#### Structure and Files

The Helm option is located under the `kubernetes/option-1-helm/` directory. This directory contains a Helm chart that automates the deployment of the ELK stack. 

Here are the key files:

- **`Chart.yaml`**: Contains metadata about the Helm chart, including the version and description.
- **`values.yaml`**: Provides default configuration values that can be customized when deploying the chart.
- **`templates/`**: Contains Helm templates for Kubernetes resources (like `StatefulSet`, `Deployment`, `Service`, etc.). These templates are populated with values from `values.yaml` or from user-provided overrides.

#### What It Does

Helm templates allow for dynamic generation of Kubernetes manifests. The `values.yaml` file provides default values, but you can override these during deployment to customize the ELK stack. The templates inside the `templates/` directory are used to generate Kubernetes manifests on the fly, based on the input values.

#### How to Deploy

##### Basic Deployment with Default Values

```sh
helm install elk-stack kubernetes/option-1-helm
```

This command will deploy the ELK stack using the default values defined in `values.yaml`.

##### Custom Deployment with Overrides

```sh
helm install elk-stack kubernetes/option-1-helm --set elasticVersion=7.12.0 --set elasticsearch.storage=2Gi
```

This command will override specific values in the `values.yaml` file, allowing for a customized deployment.

#### When to Choose This Option

- **Scalability**: Helm is better suited for larger, more complex environments where you may need to deploy multiple instances with varying configurations.
- **Reusability**: If you plan to deploy the ELK stack in multiple environments (e.g., dev, staging, production) with slight variations, Helm makes this process easier.
- **Customization**: Helm's templating system allows for extensive customization without needing to manually edit the raw Kubernetes manifests.

### 3. Deploy with Helm and Custom Templates

#### What the Templates Are For

The templates in the `templates/` directory are used to generate the actual Kubernetes manifests based on the `values.yaml` file or any overrides you provide during deployment. These templates make it easy to reuse the same configuration across different environments while allowing for flexibility.

#### When to Use Templates

- **Complex Environments**: Use templates when deploying to environments that require different configurations, such as varying resource limits, storage sizes, or number of replicas.
- **Automation**: Helm's templating and values system is ideal if you are integrating the deployment into a CI/CD pipeline where the configuration might need to change dynamically.
- **Environment-Specific Configurations**: If your deployment needs to adapt to different environments (e.g., a dev environment with minimal resources vs. a production environment with higher availability), Helm templates allow you to manage these differences cleanly.

#### When Not to Use Templates

- **Simple Deployments**: If you do not need the flexibility of Helm or if your deployment is small and straightforward, the plain `kubectl` approach might be more appropriate.
- **Learning Curve**: If your team is unfamiliar with Helm and templating, it might introduce unnecessary complexity.

Certainly! Here's a step-by-step procedure for deploying the ELK stack on Kubernetes using Helm, including the troubleshooting steps we followed.

### **Deep Dive Procedure to Deploy ELK Stack on Kubernetes Using Helm**

#### **1. Set Up the Kubernetes Cluster**
   - Ensure you have a running Kubernetes cluster. If you're using Minikube, start the cluster with:
     ```bash
     minikube start --driver=docker
     ```

#### **2. Prepare the Helm Chart**
   - Create a directory structure for Kubernetes deployment using Helm:
     ```bash
     mkdir -p kubernetes/option-1-helm/templates
     ```
   - Place the necessary YAML files for Elasticsearch, Kibana, Logstash, and Setup in the `templates` directory.

#### **3. Create Necessary Kubernetes Resources**

   - **Create Secrets and ConfigMaps:**
     - Create the `elastic-credentials` secret, which stores passwords for Elasticsearch, Kibana, and Logstash:
       ```bash
       kubectl create secret generic elastic-credentials \
         --from-literal=elastic_password=<your-elastic-password> \
         --from-literal=kibana_password=<your-kibana-password> \
         --from-literal=logstash_password=<your-logstash-password>
       ```
     - Create the `logstash-config` ConfigMap with the Logstash configuration:
       ```bash
       kubectl create configmap logstash-config --from-file=logstash/config/logstash.yml
       ```
     - Create the `setup-scripts` ConfigMap for the setup job:
       ```bash
       kubectl create configmap setup-scripts --from-file=setup/entrypoint.sh --from-file=setup/lib.sh
       ```

#### **4. Deploy the ELK Stack Using Helm**

   - Install or upgrade the Helm chart:
     ```bash
     helm upgrade --install elk-stack ./kubernetes/option-1-helm/
     ```

#### **5. Troubleshoot Deployment Issues (if any)**

   - If you encounter errors with immutable fields or configuration issues, delete the affected resources:
     ```bash
     kubectl delete job setup
     ```
   - Update the necessary YAML files and reapply the Helm chart:
     ```bash
     helm upgrade --install elk-stack ./kubernetes/option-1-helm/
     ```

#### **6. Monitor the Deployment**

   - Check the status of the pods:
     ```bash
     kubectl get pods
     ```
   - Ensure that all pods (`elasticsearch`, `kibana`, `logstash`, `setup`) are in the `Running` state.

#### **7. Access and Verify the ELK Stack**

   - **Access Kibana:**
     - Forward the Kibana service port to your local machine:
       ```bash
       kubectl port-forward service/kibana 5601:5601
       ```
     - Open a browser and navigate to `http://localhost:5601` to access the Kibana UI.
   - **Check Logs:**
     - Verify the logs for each component to ensure they are running correctly:
       ```bash
       kubectl logs elasticsearch-0
       kubectl logs kibana-7f58c79487-5fvvv
       kubectl logs logstash-6f98c4c97f-w9kpw
       kubectl logs setup-gdp4v
       ```

#### **8. Clean Up Resources**

   - After the setup job completes, check its status:
     ```bash
     kubectl get jobs
     ```
   - If the job is no longer needed, delete it:
     ```bash
     kubectl delete job setup
     ```

### **Conclusion**

By following this procedure, you should have successfully deployed the ELK stack on Kubernetes using Helm, with all components running and accessible via Kibana. The setup and configuration should be verified by checking the logs and ensuring all services are functioning correctly.

If further adjustments or troubleshooting are required, repeat the necessary steps to resolve any issues.

### Summary

- **Use `kubectl`** if you need simplicity, fine-grained control, or are dealing with a smaller deployment.
- **Use Helm with Default Values** for scalable, reusable, and consistent deployments across multiple environments.
- **Use Helm with Custom Templates** if you need extensive customization, are dealing with complex environments, or are automating deployments as part of a CI/CD pipeline.
