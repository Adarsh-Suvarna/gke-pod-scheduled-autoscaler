# Google Kubernetes Engine Pod Autoscale

This section explains how we can reduce costs by deploying a scheduled autoscaler on Google Kubernetes Engine (GKE). This kind of autoscaler scales clusters up or down according to a schedule based on time of day or day of the week.
This approach is useful in the scenarios where sharp changes in traffic patterns are well understood and we want to give hints to the autoscaler that your infrastructure is about to experience spikes.
In this proof of concept (POC) the scheduled autoscaler consists of a set of components that work together to manage scaling based on a schedule. We have used a set of Kubernetes CronJobs to export known information about traffic patterns to a Cloud Monitoring custom metric. This data is then read by a Kubernetes Horizontal Pod Autoscaler (HPA) as input into when the HPA should scale the workload. Along with other load metrics, such as target CPU utilization, the HPA decides how to scale the replicas for a given deployment.

Following sections describe the step by step process to achieve the Google Kubernetes Engine Pod autoscaling.

## Preparing the environment

In the Cloud Shell, configure the Cloud project ID and the computing zone and region

```diff
PROJECT_ID=<PROJECT_ID>
gcloud config set project $PROJECT_ID
gcloud config set compute/region asia-south1
gcloud config set compute/zone asia-south1-a
```
## Create the GKE cluster

Create the GKE cluster either using gcloud command or using GCP Console.

  <p>
  <img src="https://github.com/Adarsh-Suvarna/gke-scheduled-autoscaler/blob/main/img/gke-1.png">
  </p>

## Setting up scheduled autoscaler

In this example we are using Custom Metrics - Stackdriver Adapter is an implementation of Custom Metrics API and External Metrics API using Stackdriver as a backend. Its purpose is to enable pod autoscaling based on Stackdriver custom metrics. This adapter enables Pod autoscaling based on Cloud Monitoring custom metrics. So install the Custom Metrics Cloud Monitoring adapter in the GKE cluster.

```diff
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/k8s-stackdriver/master/custom-metrics-stackdriver-adapter/deploy/production/adapter_new_resource_model.yaml

kubectl wait --for=condition=available --timeout=600s deployment/custom-metrics-stackdriver-adapter -n custom-metrics
```

## Creation of Artifact Registry

1. Create a repository in Artifact Registry and give the read permissions

```diff
gcloud artifacts repositories create gke-scheduled-autoscaler \
  --repository-format=docker --location=asia-south1

gcloud auth configure-docker asia-south1-docker.pkg.dev

gcloud artifacts repositories add-iam-policy-binding gke-scheduled-autoscaler \
   --location=asia-south1 --member=allUsers --role=roles/artifactregistry.reader
```
2. Build and push the custom metric exporter code

```diff
docker build -t asia-south1-docker.pkg.dev/$PROJECT_ID/gke-scheduled-autoscaler/custom-metric-exporter .

docker push asia-south1-docker.pkg.dev/$PROJECT_ID/gke-scheduled-autoscaler/custom-metric-exporter
```

  <p>
  <img src="https://github.com/Adarsh-Suvarna/gke-scheduled-autoscaler/blob/main/img/ar-2.png">
  </p>

## Deploy the application

1.  Deploy the CronJobs that export custom metrics and deploy the HPA that reads from these custom metrics.
```diff
sed -i.bak s/PROJECT_ID/$PROJECT_ID/g ./gke-scheduled-autoscaler/scheduled-autoscale.yaml

kubectl apply -f ./gke-scheduled-autoscaler
```

  <p>
  <img src="https://github.com/Adarsh-Suvarna/gke-scheduled-autoscaler/blob/main/img/vs-3.png">
  </p>

2.  The following listing shows the content of the file ```gke-scheduled-autoscaler/deployment.yaml```

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 1
  strategy:
      rollingUpdate:
         maxSurge: 25%
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
          limits:
            cpu: "60m"
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    run: php-apache
```
3. The following listing shows the content of the file ```gke-scheduled-autoscaler/hpa.yaml```

```diff
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  maxReplicas: 20
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: External
    external:
      metric:
        name: custom.googleapis.com|scheduled_autoscaler
      target:
          type: AverageValue
          averageValue: 1
```

This configuration specifies minReplicas to 1. This means that the workload can be scaled down to its minimum. The configuration also adds an external metric (type: External). This addition means that autoscaling is now triggered by two factors. In this multiple-metrics scenario, the HPA calculates a proposed replica count for each metric and then chooses the metric that returns the highest value. It's important to understand that a scheduled autoscaler can propose that at a given moment the Pod count should be 1. But if the actual CPU utilization is higher than expected for one Pod, the HPA creates more replicas.

4. The following listing shows the content of the file ```gke-scheduled-autoscaler/scheduled-autoscale.yaml```

```diff 
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekdays-pods-scale-up
spec:
  schedule: "0 17 * * 1-5"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: custom-metric-extporter
            image: asia-south1-docker.pkg.dev/PROJECT_ID/gke-scheduled-autoscaler/custom-metric-exporter
            command:
              - /export
              - --name=scheduled_autoscaler
              - --value=70
          restartPolicy: OnFailure
      backoffLimit: 1
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekdays-pods-scale-down
spec:
  schedule: "0 0 * * 1-5"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: custom-metric-extporter
            image: asia-south1-docker.pkg.dev/PROJECT_ID/gke-scheduled-autoscaler/custom-metric-exporter
            command:
              - /export
              - --name=scheduled_autoscaler
              - --value=35
          restartPolicy: OnFailure
      backoffLimit: 1
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekends-pods-scale-up
spec:
  schedule: "0 14 * * 6,0"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: custom-metric-extporter
            image: asia-south1-docker.pkg.dev/PROJECT_ID/gke-scheduled-autoscaler/custom-metric-exporter
            command:
              - /export
              - --name=scheduled_autoscaler
              - --value=70
          restartPolicy: OnFailure
      backoffLimit: 1
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekends-pods-scale-down
spec:
  schedule: "0 0 * * 6,0"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: custom-metric-extporter
            image: asia-south1-docker.pkg.dev/PROJECT_ID/gke-scheduled-autoscaler/custom-metric-exporter
            command:
              - /export
              - --name=scheduled_autoscaler
              - --value=35
          restartPolicy: OnFailure
      backoffLimit: 1
---
```

This configuration specifies that the CronJobs should export the suggested Pod replicas count to a custom metric called custom.googleapis.com|scheduled_autoscaler based on the time of day.

5. Check the number of nodes and HPA replicas running below command.

```diff
kubectl get hpa php-apache
```

## GKE Pod Autoscaling Observations

We have tested the GKE Pod Autoscaling for the desired time and it was working as expected. We have tested for Pod Scale up and Pod scale down by setting up the desired time using Cron Jobs. Below screenshot shows the deployment list, where initially the Pod count set to 20 and then scale down to count of Pod set to 2 and again pod count set to 20 for the desired time.

  <p>
  <img src="https://github.com/Adarsh-Suvarna/gke-scheduled-autoscaler/blob/main/img/cmd-4.png">
  </p>

  <p>
  <img src="https://github.com/Adarsh-Suvarna/gke-scheduled-autoscaler/blob/main/img/cmd-5.png">
  </p>


Below screenshot shows the Pod counts and Cron jobs in the GCP Console.
  <p>
  <img src="https://github.com/Adarsh-Suvarna/gke-scheduled-autoscaler/blob/main/img/gcp-6.png">
  </p>

  <p>
  <img src="https://github.com/Adarsh-Suvarna/gke-scheduled-autoscaler/blob/main/img/gcp-7.png">
  </p>

  <p>
  <img src="https://github.com/Adarsh-Suvarna/gke-scheduled-autoscaler/blob/main/img/gcp-8.png">
  </p>
  
Note: All CronJob schedule: times are based on the timezone of the kube-controller-manager. GKE’s master follows UTC time zone and hence our cron jobs need to be readjusted to run at IST timings.

### Contact Me
 If you have any doubts on this please reach out to me
 - Adarsh Suvarna : adarshasuvarna@outlook.com