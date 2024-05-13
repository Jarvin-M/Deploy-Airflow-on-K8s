# Deploying Airflow on local Kubernetes(KinD) cluster with Helm

This is part of a blog post - see link for complete blog

## Steps
1. Clone github repository with all the files you need: *link*
2. Install tools listed in the above section. I used a MacBook to set this up, so i used [Homebrew](https://brew.sh/) to install the tools
3. Setup local K8s KinD cluster with cluster definition in `kind-cluster.yaml`. Within the cluster definition, we can pass storage from your host (local file system in this case) to the KinD node via `extraMounts` to persist data or files - [documentation](https://kind.sigs.k8s.io/docs/user/configuration/#extra-mounts).
    1. In this example we want to persist the Airflow DAG and log files. Update `hostPath` to a location on your local machine

        ```yaml
        extraMounts:
            - hostPath: /<path-to-dir>/airflow_on_k8s/dags
                containerPath: /dags
            - hostPath: /<path-to-dir>/airflow_on_k8s/logs
                containerPath: /logs
        ```
    3. The paths set in this step will later be mounted to our Airflow deployment and act as persisted storage
    4. Create cluster- ` kind create cluster --config kind-cluster.yaml `
        1. At the end of this tutorial you can clean up the cluster and associated elements by - `kind delete cluster --name airflow-tutorial-cluster`
4. Deploy Airflow with Helm
    1. Pre-requisites for deployment
        1. Create  a namespace - `airflow`. [Namespaces](https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/) is a way to organize a cluster by isolating groups of resources/applications. `--dry-run=client` flag allows you to preview the object that would be sent to your cluster, without really applying it.
            1. `kubectl create namespace airflow --dry-run=client -o yaml > airflow.yaml`
            2. `kubectl apply -f airflow.yaml`
        2. Create persistent volumes. A [persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) is a piece of storage on the K8s cluster that has an independent lifespan.
            1.  `kubectl apply -f kubernetes-persistent-volumes.yaml -n airflow`
        3. Create claims for persistent volumes. A claim is a request for storage by a user that consumes the Persistent Volume resources. In this example, we [bind each claim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#binding) to a specific persistent volumes- 
            1. `kubectl apply -f persistent-volume-claims.yaml`
        4. Setup an [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) that acts as a load balancer that routes traffic from outside our cluster and routes it to the appropriate pods.  This will be helpful for us to access the Airflow UI on a browser via the web [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) in the deployment . For this, we use NGINX Ingress controller in our KinD cluster - [docs](https://kind.sigs.k8s.io/docs/user/ingress/#using-ingress)
            1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml`
            2. `kubectl wait --namespace ingress-nginx \
                       --for=condition=ready pod \
                       --selector=app.kubernetes.io/component=controller \
                       --timeout=90s`
    2. Setup Airflow deployment via Helm
        1. Helm is a tool to manage Kubernetes applications with [charts](https://helm.sh/docs/topics/charts/) with which you define,  deploy, run and scale all the required K8S components for said application. Helm Charts also help you manage and maintain dependencies and configurations(see different [approaches of configuration management](https://www.giantswarm.io/blog/application-configuration-management-in-kubernetes/))
        2. An official maintained helm chart is available for [Airflow](https://airflow.apache.org/docs/helm-chart/stable/index.htm)- contains all the crucial resource definitions needed for an Airflow application
            1. Add the chart repository-
                1. `helm repo add apache-airflow https://airflow.apache.org`
                2. `helm repo update`
            2. Extract values from airflow helm chart. `Values` is a built-in object of Helm and it contains configuration settings that are passed to the chart. With this file/object, one can define desired values for different aspects of the deployment.
                1. `helm show values apache-airflow/airflow > values.yaml`
                2. Adjust values to desired ones. Below are the choices I made and why:
                    1. Change executor config setting to `KubernetesExecutor` - there are [multiple Airflow executors](https://docs.astronomer.io/learn/airflow-executors-explained) with varying pros and cons and suitable usecases. I chose Kubernetes executor because of its resource optimization and scaling capabilities where it can dynamically launch pods for every task and terminate the pods when task is completed. 
                    2. Enable web ingress to ensure we can route external traffic via the Ingress controller to our deployment. This will make the Airflow UI accessible. Read more about how you can configure [Path Types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types) and [DNS hosts](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types)
                        
                        ```yaml
                        web:
                            # Enable web ingress resource
                            enabled: true
                        
                            # The path for the web Ingress
                            path: "/"
                            # The pathType for the above path (used only with Kubernetes v1.19 and above)
                            pathType: "ImplementationSpecific"
                            # The hostnames or hosts configuration for the web Ingress
                            hosts: []
                        ```
                        
                    3. Enable persistence for logs and DAGs by specifying our previously defined  Persistent Volume claims under `existingClaim` and matching access modes of the corresponding persistent volume. This configuration ensures that DAG files that we add to our local filesystem(see step 3) can be loaded by Airflow and corresponding logs from a DAG run are persisted as well.
            3. Deploy helm chart with Airflow to our KinD cluster on
                1. `helm upgrade --install airflow apache-airflow/airflow -n airflow -f helm/values.yaml --debug`
            4. Inspect created Airflow deployment
                1. `kubectl get deployment -n airflow -o wide`
            5. Access Airflow UI on http://localhost/. Default logins are U: `airflow` P: `airflow`
5. Add example DAG files in defined mount location - `~/dags` and it should automatically load in the Airflow UI.