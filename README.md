1. Dockerfile & Building Images

**`Dockerfile` for Spring Boot App:**

```dockerfile
FROM openjdk:17-alpine
ARG JAR_FILE=target/app.jar
COPY ${JAR_FILE} /app.jar
ENV JAVA_OPTS=""
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -Djava.security.egd=file:/dev/./urandom -jar /app.jar"]
```

Build your image:

```sh
mvn clean package
docker build -t yourrepo/task-app:latest .
```

---

 2. Kubernetes Manifests

**a) MongoDB Deployment with PVC**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo
spec:
  replicas: 1
  selector: { matchLabels: { app: mongo } }
  template:
    metadata:
      labels: { app: mongo }
    spec:
      containers:
      - name: mongo
        image: mongo:6.0
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-storage
          mountPath: /data/db
      volumes:
      - name: mongo-storage
        persistentVolumeClaim:
          claimName: mongodb-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  selector:
    app: mongo
```

**b) Task App Deployment + Service**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-app
spec:
  replicas: 1
  selector: { matchLabels: { app: task-app } }
  template:
    metadata:
      labels: { app: task-app }
    spec:
      containers:
      - name: task-app
        image: yourrepo/task-app:latest
        env:
        - name: SPRING_DATA_MONGODB_URI
          value: mongodb://mongo:27017/tasksdb
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: task-app
spec:
  type: NodePort
  selector:
    app: task-app
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
```

---

## 🛠️ 3. Deploying to Local Kubernetes

Using Minikube / kind / Docker Desktop:

```bash
kubectl apply -f mongo-deployment.yaml
kubectl apply -f task-app-deployment.yaml
```

Verify:

```bash
kubectl get pods
kubectl get svc
curl http://localhost:30080/tasks
```

Make sure endpoints respond correctly, demonstrating deployment success.

 4. Java: Triggering Pod-Based TaskExecution

Switch from `Runtime.exec(...)` to Fabric8-based creation of BusyBox pods. Here’s a snippet:

```java
import io.fabric8.kubernetes.client.*;
import io.fabric8.kubernetes.api.model.*;

public TaskExecution runTaskOnPod(Task task) {
  try (KubernetesClient client = new KubernetesClientBuilder().build()) {
    String podName = "task-" + UUID.randomUUID();
    Pod pod = new PodBuilder()
      .withNewMetadata().withName(podName).endMetadata()
      .withNewSpec()
        .withRestartPolicy("Never")
        .addNewContainer()
          .withName("runner")
          .withImage("busybox")
          .withCommand("sh","-c", task.getCommand())
        .endContainer()
      .endSpec()
      .build();

    client.pods().inNamespace("default").create(pod);
    client.pods().inNamespace("default").withName(podName)
      .waitUntilCondition(p -> "Succeeded".equals(p.getStatus().getPhase()) ||
                              "Failed".equals(p.getStatus().getPhase()), 5, TimeUnit.MINUTES);

    String output = client.pods().inNamespace("default")
      .withName(podName)
      .getLog();

    client.pods().inNamespace("default").withName(podName).delete();

    return new TaskExecution(Date.from(...), Date.from(...), output);
  }
}
```

This:

* Creates a new pod per execution with BusyBox and the task command.
* Waits for its completion.
* Collects logs as `output`.
* Cleans it up.

📘 Fabric8 patterns inspired by Red Hat’s guide ([developers.redhat.com][1], [medium.com][2], [thinkmicroservices.com][3], [dzone.com][4], [ganapathi-naik.medium.com][5]) and job execution tutorial .

---

## 🎯 5. Testing & Proof

Take screenshots after applying manifests:

1. `kubectl get pods` showing `mongo-*` and `task-app-*`.
2. `kubectl get pvc` confirming PVC status.
3. `curl http://localhost:30080/tasks` returning JSON list (proof of running app).
4. Sample screenshot showing POST of task and execution through pod creation.

---
