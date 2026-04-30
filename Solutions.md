</details>

******

<details>
<summary>Exercise 1: Deploy your Application and Prepare the Setup </summary>
 <br />

**steps**
```sh
# Create K8s cluster on LKE for example and set kubeconfig file
chmod 400 ~/Downloads/monitoring-kubeconfig.yaml
export KUBECONFIG=~/Downloads/monitoring-kubeconfig.yaml

# Create docker-registry secret
DOCKER_REGISTRY_SERVER=docker.io
DOCKER_USER=your dockerID, same as for `docker login`
DOCKER_EMAIL=your dockerhub email, same as for `docker login`
DOCKER_PASSWORD=your dockerhub pwd, same as for `docker login`

kubectl create secret docker-registry my-registry-key \
--docker-server=$DOCKER_REGISTRY_SERVER \
--docker-username=$DOCKER_USER \
--docker-password=$DOCKER_PASSWORD \
--docker-email=$DOCKER_EMAIL

# Execute Ansible playbook to deploy java and mysql apps in k8s cluster
ansible-playbook 1-configure-k8s.yaml

# NOTES:
# If you get an error on creating ingress component related to "nginx-controller-admission" webhook, than manually delete the ValidationWebhook and try again. To delete the ValidationWebhook:
kubectl get ValidatingWebhookConfiguration # gives you the name
kubectl delete ValidatingWebhookConfiguration {name}
```

</details>

******

<details>
<summary>Exercise 2: Start Monitoring your Applications </summary>
 <br />

**steps**
```sh
# Deploy promentheus operator
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring
helm install monitoring-stack prometheus-community/kube-prometheus-stack -n monitoring

# Access Prometheus UI and view its targets
kubectl -n monitoring port-forward svc/monitoring-stack-kube-prom-prometheus 9090:9090
http://127.0.0.1:9090/targets

# Clean up and prepare for new installation
helm uninstall mysql-release
helm uninstall ingress-controller -n ingress

# NOTE:
# We are using "release: monitoring-stack" label to expose scrape endpoint. This label may change with newer prometheus stack version, so to check which label you need to apply, do the following
# - Get name of the prometheus CRD
kubectl get prometheuses.monitoring.coreos.com -n monitoring
# - Print out the ServiceMonitor selector
kubectl get prometheuses.monitoring.coreos.com {crd-name} -n monitoring -o yaml | grep serviceMonitorSelector -A 2

# Build and use the correct java-app image with metrics exposed:
# check out the java app code with prometheus client inside
git checkout main
# build a new jar
gradle clean build
# build a docker image
docker build -t {docker-hub-id}:{repo-name}:{tag} .
docker push {docker-hub-id}:{repo-name}:{tag}
# set the correct image name "{docker-hub-id}:{repo-name}:{tag}" in "kubernetes-manifests/java-app.yaml" file


# To add metrics scraping to nginx, mysql and java apps, execute ansible playbook
# Ensure you have added the correct IP addressing details to your ingress YAML file
ansible-playbook 2-configure-k8s.yaml

# Access Prometheus UI and see that new targets for mysql, nginx and your java application have been added
http://127.0.0.1:9090/targets
```

</details>

******

<details>
<summary>Exercise 3: Configure Alert Rules </summary>
 <br />

**steps**
```sh
# NOTE:
# We are using "release: monitoring-stack" label to add alert rules. This label may change with newer prometheus stack version, so to check which label you need to apply, do the following
# - Get name of the prometheus CRD
kubectl get prometheuses.monitoring.coreos.com
# - Print out the alert rule selector
kubectl get prometheuses.monitoring.coreos.com {crd-name} -o yaml | grep ruleSelector -A 2


# Execute following to add prometheus alert rules
kubectl apply -f kubernetes-manifests/3-nginx-alert-rules.yaml
kubectl apply -f kubernetes-manifests/3-mysql-alert-rules.yaml
kubectl apply -f kubernetes-manifests/3-java-alert-rules.yaml
kubectl apply -f kubernetes-manifests/3-k8s-alert-rules.yaml 
```

</details>

******

<details>
<summary>Exercise 4: Send Alert Notifications </summary>
 <br />

**steps**
```sh
# Use the following guide to set up your Slack channel:
https://www.freecodecamp.org/news/what-are-github-actions-and-how-can-you-automate-tests-and-slack-notifications/

# Configure your email account as I show in the monitoring module video:
10 - Configure Alertmanager with Email Receiver

# Execute following to configure alert manager to send notifications
kubectl apply -f kubernetes-manifests/4-email-secret.yaml
kubectl apply -f kubernetes-manifests/4-slack-secret.yaml
kubectl apply -f kubernetes-manifests/4-alert-manager-configuration.yaml
```

</details>

******

### Official Docs of prometheus monitoring API for K8s
https://docs.openshift.com/container-platform/latest/rest_api/monitoring_apis/monitoring-apis-index.html

