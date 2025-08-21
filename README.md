# Epay
🔑 Spark on Kubernetes — Architecture
1. Main Components

1).Spark Operator
 - Runs as a Deployment inside your Kubernetes cluster.
 - Watches for SparkApplication CRDs (Custom Resources).
 - Automates creation & cleanup of Spark jobs (Driver + Executors).

2).Spark Driver Pod
 - First pod launched for a job.
 - Handles Spark context, scheduling, and coordinates executors.

3).Spark Executor Pods
 - Spawned by the driver.
 - Do the actual work (tasks, shuffle, caching, etc.).

4).Kubernetes API Server
 - Central brain of the cluster.
 - Operator communicates with it to create/update/delete Pods.

5).etcd (inside Kubernetes)
 - Stores the state of all resources (pods, services, CRDs).


kubectl apply -f spark-pi.yaml


 +-------------------+
 | kubectl apply -f  |
 | spark-pi.yaml     |
 +-------------------+
          |
          v
 +-------------------+     (CRDs in etcd)
 | Kubernetes API    | <----------------+
 | Server            |                  |
 +-------------------+                  |
          |                             |
          v                             |
 +-------------------+   watches        |
 | Spark Operator    |------------------+
 | (controller pod)  |
 +-------------------+
          |
   creates Driver Pod
          v
 +-------------------+   requests executors   +-------------------+
 | Spark Driver Pod  |----------------------->| K8s API → Execs   |
 +-------------------+                        +-------------------+
          |                                              |
  schedules tasks                               multiple Executor Pods
          |                                              |
          v                                              v
   Aggregates Results <---------------------------> Executors Compute


### 2. **Flow of a SparkApplication**

Let’s say you run:

```bash
kubectl apply -f spark-pi.yaml
```

#### Behind the scenes:

1. **CRD Creation**

   * Your YAML defines a `SparkApplication` (Custom Resource).
   * Kubernetes stores this in etcd.
   * Example (simplified):

     ```yaml
     apiVersion: sparkoperator.k8s.io/v1beta2
     kind: SparkApplication
     metadata:
       name: spark-pi
       namespace: default
     spec:
       type: Scala
       mode: cluster
       image: gcr.io/spark-operator/spark:v4.0.0
       mainClass: org.apache.spark.examples.SparkPi
       mainApplicationFile: local:///opt/spark/examples/jars/spark-examples_2.12-4.0.0.jar
       ...
     ```

2. **Spark Operator Detects**

   * The **Spark Operator** is watching for new `SparkApplication` objects (via Kubernetes API).
   * It sees `spark-pi` → translates this into actual Kubernetes resources.

3. **Driver Pod Creation**

   * Operator submits a **Driver Pod** with Spark container.
   * Example driver pod name: `spark-pi-driver`.
   * This pod starts and runs `org.apache.spark.deploy.SparkSubmit`.

4. **Driver Contacts Kubernetes API**

   * Driver connects to the Kubernetes API server to request Executor Pods.
   * Spark driver acts like a scheduler inside Kubernetes.

5. **Executor Pods Creation**

   * Kubernetes API creates multiple Executor pods (e.g., `spark-pi-exec-1`, `spark-pi-exec-2`).
   * Each executor pod runs Spark executor JVM.

6. **Execution Phase**

   * Driver assigns tasks to executors.
   * Executors perform computation and send results back to driver.
   * Logs are stored in pod logs (accessible via `kubectl logs`).

7. **Completion & Cleanup**

   * Once Spark job finishes, driver exits.
   * Operator can automatically delete driver + executor pods (if configured) or keep them for debugging.
   * SparkApplication status updated → `Succeeded` or `Failed`.

### 4. **Namespaces in Play**

* If you submit job in `default` → driver/executors created in `default`.
* If in `spark-operator` → driver/executors created there.
* That’s why `kubectl get sparkapplications -n default` is critical.

---

### 5. **How Spark on K8s is Different from YARN**

* YARN: driver talks to **YARN ResourceManager** for executors.
* K8s: driver talks to **Kubernetes API Server** for executor pods.
* Both achieve the same → dynamic resource allocation.
