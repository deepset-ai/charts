# Hayhooks Helm Chart

[Hayhooks](https://github.com/deepset-ai/hayhooks) allows you to run Haystack pipelines as REST APIs. This Helm chart simplifies deploying Hayhooks to a Kubernetes cluster.

## Requirements

* Kubernetes 1.19+
* Helm 3.2.0+

## Installing the Chart

1. **Add the deepset Helm repository:**

    ```console
    helm repo add deepset https://deepset-ai.github.io/charts/
    helm repo update
    ```

2. **Install the chart:**

    ```console
    helm install my-hayhooks deepset/hayhooks \
      --namespace my-namespace \
      --create-namespace
    ```

    Replace `my-hayhooks` with your desired release name and `my-namespace` with the target namespace.

## Configuration

The following table lists the configurable parameters of the Hayhooks chart and their default values.

| Parameter                       | Description                                                                                      | Default                         |
| ------------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------- |
| `replicaCount`                  | Number of Hayhooks pods to deploy.                                                              | `1`                             |
| `image.repository`              | Hayhooks container image repository.                                                              | `deepset/hayhooks`              |
| `image.pullPolicy`              | Image pull policy.                                                                               | `IfNotPresent`                  |
| `image.tag`                     | Image tag (defaults to the chart's appVersion).                                                 | Chart's `appVersion`            |
| `hayhooks.rootPath`             | Base path under which the Hayhooks API is served. Set this according to your Ingress path.         | `""`                            |
| `pipelinesDirMount`             | Path inside the container where pipeline *wrapper directories* will be mounted.                 | `/opt/pipelines`                |
| `nameOverride`                  | Override the chart name.                                                                         | `""`                            |
| `fullnameOverride`              | Override the full release name.                                                                  | `""`                            |
| `service.port`                  | Kubernetes service port for Hayhooks.                                                              | `1416`                          |
| `ingress.enabled`               | Enable Ingress resource.                                                                          | `true`                          |
| `ingress.className`             | Ingress controller class name.                                                                   | `""`                            |
| `ingress.annotations`           | Additional annotations for the Ingress resource.                                                  | `{nginx.ingress.kubernetes.io/rewrite-target: /$1}` |
| `ingress.hosts[0].host`         | Hostname for the Ingress rule.                                                                    | `localhost`                     |
| `ingress.hosts[0].paths[0].path`| Path for the Ingress rule. Should generally align with `hayhooks.rootPath`.                       | `/(.*)`                         |
| `ingress.hosts[0].paths[0].pathType` | Path type for the Ingress rule.                                                               | `ImplementationSpecific`        |
| `ingress.tls`                   | TLS configuration for Ingress.                                                                    | `[]`                            |
| `resources`                     | Pod resource requests and limits.                                                                 | `{}`                            |
| `autoscaling.enabled`           | Enable Horizontal Pod Autoscaler.                                                                 | `false`                         |
| `autoscaling.minReplicas`       | Minimum number of replicas for HPA.                                                              | `1`                             |
| `autoscaling.maxReplicas`       | Maximum number of replicas for HPA.                                                              | `100`                           |
| `autoscaling.targetCPUUtilizationPercentage` | Target CPU utilization for HPA.                                                     | `80`                            |
| `volumes`                       | Additional volumes to add to the Deployment. Use this to mount pipeline wrapper directories.     | `[]`                            |
| `volumeMounts`                  | Additional volume mounts for the Hayhooks container. Mount pipeline volumes to `pipelinesDirMount`. | `[]`                            |
| `nodeSelector`                  | Node selector constraints.                                                                       | `{}`                            |
| `tolerations`                   | Pod tolerations.                                                                                 | `[]`                            |
| `affinity`                      | Pod affinity and anti-affinity rules.                                                            | `{}`                            |
| `podAnnotations`                | Additional annotations for the Pods.                                                             | `{}`                            |
| `podLabels`                     | Additional labels for the Pods.                                                                  | `{}`                            |
| `podSecurityContext`            | Security context for the Pods.                                                                   | `{}`                            |
| `securityContext`               | Security context for the Hayhooks container.                                                      | `{}`                            |
| `envVariables`                  | Additional environment variables as a map.                                                        | `{}`                            |
| `extraEnvVars`                  | Additional environment variables as a list (supports `valueFrom`).                                 | `[]`                            |
| `runtimeClassName`              | Runtime class name for the Pods.                                                                 | `""`                            |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example:

```console
helm install my-hayhooks deepset/hayhooks \
  --set image.tag=v0.5.0 \
  --set ingress.hosts[0].host=hayhooks.example.com \
  --set resources.limits.cpu=500m \
  --set resources.limits.memory=1Gi
```

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example:

```console
helm install my-hayhooks deepset/hayhooks -f values.yaml
```

## Loading Pipelines

Hayhooks deploys pipelines defined using `PipelineWrapper` classes located in Python files (typically `pipeline_wrapper.py`). These wrappers might load pipeline configurations from YAML files or define them programmatically.

To make your pipelines available to Hayhooks in Kubernetes, you need to mount the *directories* containing your `pipeline_wrapper.py` files (and any associated YAML files) into the container using a Kubernetes volume.

1. **Structure your pipeline:** Each pipeline should reside in its own directory, containing at least a `pipeline_wrapper.py` file.

    ```bash
    my_pipelines/
    ├── pipeline_one/
    │   ├── pipeline_wrapper.py
    │   └── pipeline_one.yaml  # Optional
    └── pipeline_two/
        └── pipeline_wrapper.py
    ```

2. **Create a volume:** Use a suitable Kubernetes volume type (`hostPath`, `persistentVolumeClaim`, `configMap` for scripts, etc.) to hold your pipeline directories (e.g., the `my_pipelines` directory from the example above).
3. **Configure `values.yaml`:**
    * Define the volume in the `volumes` section.
    * Mount this volume to the path specified by `pipelinesDirMount` (default: `/opt/pipelines`) using the `volumeMounts` section. Hayhooks will automatically scan this directory for subdirectories containing `pipeline_wrapper.py` files upon startup.

**Example using a hostPath volume:**

Assume your pipeline directories (like `pipeline_one`, `pipeline_two`) are located on the node under `/data/hayhooks_pipelines`.

```yaml
# values.yaml
pipelinesDirMount: "/opt/pipelines" # Default mount point inside container

volumes:
  - name: pipeline-wrappers
    hostPath:
      path: /data/hayhooks_pipelines # Path on the Kubernetes node containing directories like 'pipeline_one'
      type: DirectoryOrCreate

volumeMounts:
  - name: pipeline-wrappers
    mountPath: "/opt/pipelines" # Must match pipelinesDirMount
```

Then install/upgrade with `helm install/upgrade my-hayhooks deepset/hayhooks -f values.yaml`. Hayhooks will load the pipelines defined in the `pipeline_wrapper.py` files found within the subdirectories of `/opt/pipelines` inside the container.

**Important:** Ensure your Hayhooks container image includes any additional Python dependencies required by your `PipelineWrapper`'s `setup()` method (e.g., `trafilatura`, specific Haystack integrations). You might need to build a custom image.

## Ingress and Root Path

* The `hayhooks.rootPath` value tells the Hayhooks application the base URL path it's served under.
* The `ingress.hosts[0].paths[0].path` value tells the Ingress controller which URL paths to route to Hayhooks.
* The `ingress.annotations."nginx.ingress.kubernetes.io/rewrite-target"` controls how the Ingress controller rewrites the path before forwarding the request.

These values should be consistent:

* If `hayhooks.rootPath` is `""` (default), Hayhooks expects to be served at the root (`http://<host>/`). Your Ingress `path` should be `/(.*)` and `rewrite-target` should be `/$1`.
* If `hayhooks.rootPath` is `/haystack`, Hayhooks expects (`http://<host>/haystack/`). Your Ingress `path` should be `/haystack(/|$)(.*)` and `rewrite-target` should be `/$2`.

Adjust these values in your `values.yaml` or via `--set` according to your desired deployment structure.

## Deploying Locally (e.g., using Minikube or Kind)

When deploying this chart to a local Kubernetes cluster like Minikube or Kind for testing or development, you might encounter networking differences compared to a cloud environment.

### Prerequisites

* **Local Kubernetes Cluster:** Minikube, Kind, Docker Desktop Kubernetes, etc.
* **Ingress Controller:** You still need an Ingress controller running in your cluster.
  * **Minikube:** Enable via `minikube addons enable ingress`.
  * **Kind:** Install manually, e.g., using the [official Nginx Ingress Helm chart](https://kubernetes.github.io/ingress-nginx/deploy/#installation-guide) or the [Kind documentation instructions](https://kind.sigs.k8s.io/docs/user/ingress/).
  * **Docker Desktop:** Usually includes an Ingress controller.

### Accessing the Service

By default, the chart creates an Ingress resource configured for `localhost`. However, Services of type `LoadBalancer` (which the Nginx Ingress controller often uses) might not get an external IP automatically assigned in local clusters, preventing direct access via `http://localhost`.

Here are common ways to access Hayhooks locally:

1. **Port-Forwarding to the Hayhooks Service:**
    * Bypasses the Ingress controller. Useful for direct access or if Ingress isn't needed for testing.
    * Find the service name (default: `<release-name>-hayhooks`, check with `kubectl get svc -l app.kubernetes.io/instance=<release-name>`).
    * Run: `kubectl port-forward svc/<service-name> 1416:1416` (Replace `<service-name>` with the actual name).
    * Access via `http://localhost:1416`.

2. **Port-Forwarding to the Ingress Controller Pod:**
    * Tests the full request path, including the Ingress resource rules.
    * Find the Ingress controller pod name (e.g., `kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o name`).
    * Run: `kubectl port-forward -n <ingress-namespace> <pod-name> 8080:80` (Replace namespace and pod name).
    * Access via `http://localhost:8080` (assuming the Ingress host is `localhost` and path is `/`).

3. **Using NodePort (Less Reliable Locally):**
    * Find the NodePort for the Ingress controller's service (e.g., `kubectl get svc -n ingress-nginx`). Look for the mapping like `80:XXXXX/TCP`.
    * Access via `http://localhost:XXXXX`. This might not work reliably depending on the local cluster setup (e.g., Kind often requires extra configuration).

4. **Minikube Tunnel:**
    * If using Minikube and the Ingress service is `LoadBalancer`, run `minikube tunnel` in a separate terminal.
    * This should allow access via `http://localhost`.

5. **Kind `extraPortMappings` (Recommended for `localhost` access):**
    * To enable direct `http://localhost` access with Kind, you need to map host ports to the nodes *before* creating the cluster.
    * Create a Kind configuration file (e.g., `kind-config.yaml`):

        ```yaml
        kind: Cluster
        apiVersion: kind.x-k8s.io/v1alpha4
        nodes:
        - role: control-plane
          kubeadmConfigPatches:
          - |
            kind: InitConfiguration
            nodeRegistration:
              kubeletExtraArgs:
                node-labels: "ingress-ready=true"
          extraPortMappings:
          - containerPort: 80
            hostPort: 80 # Map host port 80 -> node port 80
            protocol: TCP
          - containerPort: 443
            hostPort: 443 # Map host port 443 -> node port 443
            protocol: TCP
        ```

    * Create the cluster: `kind create cluster --config kind-config.yaml`
    * Install the Ingress controller.
    * Install the Hayhooks chart.
    * Access should now work via `http://localhost`.

Choose the method that best suits your local environment and testing needs. Port-forwarding (methods 1 or 2) is often the quickest way to get started.
