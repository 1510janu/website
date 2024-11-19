---
reviewers:
- eparis
- pmorie
title: How to configure Redis using ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

ConfigMap is a Kubernetes mechanism that lets you inject configuration data into application [pods](/docs/concepts/workloads/pods/). The ConfigMap concept allows the decoupling of configuration artifacts from image content to keep containerized applications portable. For more information, refer to [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/). 

Redis (REmote DIctionary Server) is an open-source, in-memory, NoSQL key/value store that can function as a database, application cache and message broker. Here, Redis is being used primarily as a cache.

This tutorial will walk you through creating a ConfigMap with Redis configuration values, deploying a Redis pod that uses the ConfigMap, and verifying that the configuration is correctly applied.


## {{% heading "prerequisites" %}}


{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

Before you begin, ensure that the following prerequisites are met: 
* You need to have a Kubernetes cluster. If you do not already have a cluster, create one by using [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) or use one of these Kubernetes playgrounds:
    - [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
    - [Play with Kubernetes](https://labs.play-with-k8s.com/)
  
  **Note**: It is recommended that the cluster must have at least two nodes that are not acting as control plane hosts.
* Kubectl version must be v1.14 or above

  **Note**: If you are using a lower version of Kubectl, [upgrade](/docs/tasks/tools/) to version 1.14 or above. To verify the version, run `kubectl version`.
* You're familiar with:
    -  Basic Kubernetes concepts
    -  [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/)


<!-- lessoncontent -->

## Why configure Redis using ConfigMap?
Configuring Redis using a ConfigMap in Kubernetes centralizes and externalizes configuration management and makes it easier to update and maintain settings without modifying application code or container images.  This approach simplifies troubleshooting, enhances portability, and is ideal for managing dynamic configurations in Redis.

### Step 1: Create ConfigMap with Redis configuration

1. In your terminal, create a ConfigMap with an empty configuration block:

  ```shell
  cat <<EOF >./example-redis-config.yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: example-redis-config
  data:
    redis-config: ""
  EOF
  ```

2. Apply the ConfigMap created above, along with a Redis pod manifest:

  ```shell
  kubectl apply -f example-redis-config.yaml
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
  ```


3. Verify the contents of the Redis pod manifest and ensure the following:

   **Note**: This step ensures that the data from  `data.redis-config` in the `example-redis-config` ConfigMap is accessible in the pod as `/redis-master/redis.conf`.
    * A volume named `config` is created in `spec.volumes[1]`
    * The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
    * The `config` volume is mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.


{{% code_sample file="pods/config/redis-pod.yaml" %}}

4. Verify that the Redis pod and the ConfigMap are created:

  ```shell
  kubectl get pod/redis configmap/example-redis-config 
  ```

  You should see an output similar to this:

  ```
  NAME        READY   STATUS    RESTARTS   AGE
  pod/redis   1/1     Running   0          8s
  
  NAME                             DATA   AGE
  configmap/example-redis-config   1      14s
  ```

5. Verify that the `redis-config` key in the `example-redis-config` ConfigMap is empty: 

  ```shell
  kubectl describe configmap/example-redis-config
  ```

  You should see an output similar to this with an empty `redis-config` key:

  ```shell
  Name:         example-redis-config
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  
  Data
  ====
  redis-config:
  ```

6. Run `kubectl exec` to enter the pod and then run the `redis-cli` tool to verify the current configuration:

  ```shell
  kubectl exec -it redis -- redis-cli
  ```

7. Verify `maxmemory`:

  ```shell
  127.0.0.1:6379> CONFIG GET maxmemory
  ```

  You should see an output with the default value of 0:
  
  ```shell
  1) "maxmemory"
  2) "0"
  ```

8. Verify `maxmemory-policy`:

  ```shell
  127.0.0.1:6379> CONFIG GET maxmemory-policy
  ```
  You should see an output with the default value of `noeviction`:
  
  ```shell
  1) "maxmemory-policy"
  2) "noeviction"
  ```

### Step 2: Deploy the Redis pod that uses ConfigMap configuration

1.	Add configuration values to `example-redis-config` ConfigMap:

{{% code_sample file="pods/config/example-redis-config.yaml" %}}

2. Apply the updated ConfigMap:

  ```shell
  kubectl apply -f example-redis-config.yaml
  ```

3.	Verify that ConfigMap is updated:

  ```shell
  kubectl describe configmap/example-redis-config
  ```

  You should see an output with the updated configuration values:

  ```shell
  Name:         example-redis-config
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>
  
  Data
  ====
  redis-config:
  ----
  maxmemory 2mb
  maxmemory-policy allkeys-lru
  ```

4. Verify the Redis pod again using `redis-cli` and running `kubectl exec` to see if the configuration is applied:

  ```shell
  kubectl exec -it redis -- redis-cli
  ```

5. Verify `maxmemory`:

  ```shell
  127.0.0.1:6379> CONFIG GET maxmemory
  ```

  You must see an output that remains at the default value of 0:

  ```shell
  1) "maxmemory"
  2) "0"
  ```

6. Verify `maxmemory-policy`: 

  ```shell
  127.0.0.1:6379> CONFIG GET maxmemory-policy
  ```

  You must see an output that remains at the default value of `noeviction`:

  ```shell
  1) "maxmemory-policy"
  2) "noeviction"
  ```
  **Note**: Pods don’t automatically reflect updates made to ConfigMap. To apply the changes, you must restart the pod by deleting the existing pod and recreating it.

7. Delete the existing pod and recreate it: 

  ```shell
  kubectl delete pod redis
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
  ```

### Step 3: Verify configuration 

1.	Verify the updated configuration: 

  ```shell
  kubectl exec -it redis -- redis-cli
  ```

2. Verify `maxmemory`:

  ```shell
  127.0.0.1:6379> CONFIG GET maxmemory
  ```

  You must see an output with an updated value of 2097152:

  ```shell
  1) "maxmemory"
  2) "2097152"
  ```

3. Verify `maxmemory-policy`:

  ```shell
  127.0.0.1:6379> CONFIG GET maxmemory-policy
  ```
  You must see an output with an updated value of `allkeys-lru`:

  ```shell
  1) "maxmemory-policy"
  2) "allkeys-lru"
  ```

4. Delete the created resources:

  ```shell
  kubectl delete pod/redis configmap/example-redis-config
  ```

## {{% heading "whatsnext" %}}


* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Follow an example of [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
