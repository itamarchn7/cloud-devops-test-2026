# Answers for GCP Cloud DevOps Engineer exercise

## 1. Service interruptions
### Part 1
#### 1. 
If the only Pod is terminated, updated, or fails, there is no replacement Pod available.
During a version upgrade:
Kubernetes needs to replace the old Pod with a new one. If the old Pod is terminated before the new Pod becomes ready, the API becomes unavailable. During this period, users may experience errors or service disruptions.

*Solution:*
Increase the number of replicas to at least two (replicas: 2) to ensure service availability during updates or Pod failures.

#### 2. 
A Readiness Probe is a check performed by Kubernetes to determine whether a container is ready to receive traffic.
Assume the Pod has been created, but the FastAPI application is still: Loading configuration. Connecting to PostgreSQL. Loading initial data. Performing internal initialization.
The container may be in the Running state, but the application is not yet ready to handle requests.
Without a Readiness Probe, Kubernetes has no reliable way to verify that the application is ready. As a result, a newly started Pod may receive requests too early.

*Solution:*
We will define an application endpoint, for example:
```text
GET /health/ready
```
The endpoint will return HTTP 200 only when the application is ready to handle requests. Otherwise, it will return an HTTP 5XX error.
Kubernetes will periodically check this endpoint. If HTTP 200 is returned, the Pod is marked as Ready.
If the probe fails, the Pod is considered not ready and will not receive traffic through the Service.
A Readiness Probe failure does not terminate or restart the container. It only temporarily removes the Pod from the pool of Pods considered ready to receive traffic.

#### 3. 
A Startup Probe is designed for applications that may take some time to complete their startup process. 
For example, assume our API requires 60 seconds to complete its initialization.
If we configure overly aggressive Liveness Probes, Kubernetes may incorrectly determine that the application is stuck and restart it, even though it is still performing a normal startup process.

*Solution:*
Kubernetes gives the application enough time to complete its startup process.
As long as the Startup Probe has not succeeded, Readiness and Liveness Probes are not executed.
Once the Startup Probe succeeds, the other probes begin to run.

#### 4. 
Without resource requests and limits, Kubernetes does not know how much CPU and memory the application requires.

*Solution:*
We will define:
**Requests**: Specify the resources required by the container and influence Pod scheduling decisions.
**Limits**: Set the maximum amount of CPU and memory the container is allowed to consume.

#### 5. 
Missing Graceful Shutdown - If we are performing a new Deployment, Kubernetes needs to terminate the old Pod.
If the Pod is terminated immediately, some HTTP requests may still be in progress, causing users to experience errors.
Graceful Shutdown is an orderly application shutdown process that allows the application to finish processing active requests before exiting.

*Solution:*
We will define:
```yaml
terminationGracePeriodSeconds: 45
```
This means the application has up to 45 seconds to complete its shutdown process before Kubernetes forcefully terminates the container.

#### 6. Zero-Downtime Deployment 
A Rolling Update allows Kubernetes to replace Pods gradually instead of stopping all old Pods and starting the new ones at once.
To control this process, we have two important settings:
**maxSurge** - Defines how many additional Pods, beyond the desired replica count, can be created during the update.
**maxUnavailable** - Defines how many Pods, out of the desired replica count, are allowed to be unavailable during the update.

For example, if we have two replicas:
maxSurge: 1 allows Kubernetes to temporarily scale up to 3 Pods.
maxUnavailable: 0 ensures that Kubernetes does not intentionally reduce the number of available Pods below two during the update.

This way, Kubernetes creates a new Pod, waits until it becomes Ready, and only then proceeds with removing the old Pods.

### How to test the solution?

We will divide the tests into two categories:
- Tests that can be performed in our local Kubernetes environment.
- Tests that require a real GKE environment or a suitable simulated infrastructure.

#### Test 1 – YAML File Validation

After saving the file as `api-deployment.yaml`:

```powershell
kubectl apply --dry-run=client -f .\api-deployment.yaml
```

This command checks the resource configuration against the Kubernetes API without actually creating the resource.

We can also preview the changes that would be applied:

```powershell
kubectl diff -f .\api-deployment.yaml
```

#### Test 2 – Deployment Verification

Apply the Deployment:

```powershell
kubectl apply -f .\api-deployment.yaml
```

Check the Deployment and Pod status:

```powershell
kubectl get deployments -n production
kubectl get pods -n production -o wide
```
We expect to see two Pods in the `Ready` state, preferably running on different Nodes when possible.

#### Test 3 – Rolling Update Verification

Run:
```powershell
kubectl rollout status deployment/survey-api -n production
```

To view the Deployment revision history:
```powershell
kubectl rollout history deployment/survey-api -n production
```

During a version update, we can monitor the process using:
```powershell
kubectl get pods -n production -w
```

The goal is to observe a new Pod being created and reaching the `Ready` state before an old Pod is terminated.

#### Test 4 – Self-Healing Verification

Delete one of the Pods:
```powershell
kubectl delete pod <POD_NAME> -n production
```

Then monitor the Pods:
```powershell
kubectl get pods -n production -w
```
We expect the ReplicaSet to create a replacement Pod to restore the desired number of two replicas.

#### Test 5 – Probe Verification

Check that the Probes are properly configured:
```powershell
kubectl describe deployment survey-api -n production
```

We will also inspect the Kubernetes events:
```powershell
kubectl get events -n production --sort-by=.lastTimestamp
```

As part of a dedicated test, we can intentionally cause the Readiness Probe to fail in a controlled manner and verify that the Pod transitions out of the `Ready` state without the container being restarted.

### Part 2 
#### Entire Zone Goes Down
Both GKE Nodes are located in the same Zone. Assume that the Zone they are running in experiences a major outage.
As a result, both nodes may become unavailable and all pods running on those nodes will be affected.
The API, Frontend, and Worker services may stop working.
Even if we configured two API replicas, both may become unavailable at the same time.
Increasing the number of replicas is not sufficient if all replicas are located within the same failure domain.

*Solution: GKE Regional Cluster*
Instead of running the Cluster and Worker Nodes in a single Availability Zone, we will use a GKE Regional Cluster and distribute the Worker Nodes across multiple Zones within the same Region.
Each Pod should be scheduled in a different Zone.
This way, if Zone A goes down, the Node and Pod in Zone B can continue serving requests.

To achieve this, several changes are required:
- Enable a GKE Regional Cluster.
- Deploy Worker Nodes across multiple zones within me-west1.
- Maintain sufficient Nodes and capacity to handle the loss of an entire Zone.
- Run at least two replicas for services that require high availability.
- Configure Pod distribution across zones using topologySpreadConstraints with the topology.kubernetes.io/zone topology key.
- Configure appropriate Autoscaling for the Node Pool capacity and the number of Pods.

For example:
```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: survey-api
```
Here, we instruct Kubernetes to distribute pods across zones, ensuring that the difference in the number of pods between zones does not exceed 1.
I used DoNotSchedule to prevent Kubernetes from scheduling pods in a way that violates the distribution requirement.

If there is insufficient capacity or no suitable Zone available, a Pod may remain in the Pending state. Therefore, we must ensure that sufficient Zones and capacity are available in advance.

We can also add topology.kubernetes.io/hostname to distribute pods across different Nodes within the same Zone.

#### Cloud SQL Database
Even if our GKE cluster is deployed across multiple Zones, the database in this exercise is configured as a Single-Zone instance.
This means that if the Zone hosting Cloud SQL goes down, the application may continue running from a Kubernetes perspective, but it will no longer be able to access the database.

*Solution: Cloud SQL High Availability (Regional HA)*
With Cloud SQL HA, two logical instances are configured:
- **Primary Instance** – The active instance that handles read and write operations.
- **Standby Instance** – A standby instance located in a different Availability Zone, ready to take over in case of a failure.

If Zone A goes down, Cloud SQL performs an automatic failover to the Standby Instance in Zone B.
The application continues connecting through the Cloud SQL instance connection endpoint and should not require a manual IP address change during each failover.
However, existing connections may be interrupted, so the application must implement retry logic and reconnect to the database.


## 2. Backup and recovery
