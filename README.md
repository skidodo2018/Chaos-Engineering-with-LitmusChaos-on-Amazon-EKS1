# Project 5 Chaos-Engineering-with-LitmusChaos-on-Amazon-EKS1
Project 5

Organizations are embracing microservices-based architectures by refactoring large monolith applications into smaller, independent, and loosely coupled services. These independent services are faster to deploy and scale, enabling organizations to innovate and deliver faster.

However, as the application grows, these microservices present their own challenges. For example, as you deploy tens or hundreds or thousands of microservices, operational tasks such as distributed tracing, debugging, testing, dependency mapping, and so on, become challenging. A failure as a result of network latency, disk failure, server failure, downstream service error, and so on in a microservices-based architecture could render an application unusable.

Despite this, testing for system-level failure scenarios is often unaccounted for and hard for some organizations to make it part of their software development life cycle. To address these challenges, organizations are increasingly practicing Chaos Engineering to test the reliability and performance of distributed systems.

According to [Principles of Chaos Engineering](https://principlesofchaos.org/), “Chaos Engineering is the discipline of experimenting on a system in order to build confidence in the system’s capability to withstand turbulent conditions in production.” Chaos Engineering takes a deliberate approach of injecting failure in a controlled environment using well-planned experiments and helps engineers find weaknesses in systems before they become an outage.

### Practicing Chaos Engineering

The idea of Chaos Engineering is not to break things but to identify and understand systemic weaknesses. It can be achieved by doing controlled chaos experiments. According to [Principles of Chaos Engineering](https://principlesofchaos.org/), a chaos experiment should:

1. Define a measurable steady state of the system that indicates normal behavior as the baseline.
2. Develop a hypothesis that this state will continue in both the control group and the experimental group.
3. Introduce scenarios that reflect real-world events, such as server crashes, network latencies, hardware failures, and so on.
4. Attempt to invalidate the hypothesis by noting differences in behavior between control and experimental groups after chaos is introduced.

### LitmusChaos Architecture

[LitmusChaos](https://litmuschaos.io/) is a cloud-native Chaos Engineering framework for Kubernetes. It is built using the [Kubernetes Operator framework](https://sdk.operatorframework.io/). A [Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) is a software extension to Kubernetes that makes use of [custom resource definitions (CRDs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) to manage applications and their components.

The Litmus Chaos Operator helps reconcile the state of ChaosEngine, a custom resource that holds the chaos intent specified by a developer or DevOps engineer against a particular stateless/stateful Kubernetes deployment. The operator performs specific actions upon creation of the ChaosEngine, its primary resource. The operator also defines a secondary resource (the engine runner pod), which is created and managed by it in order to implement the reconcile functions.

Litmus takes a cloud-native approach to create, manage, and monitor chaos. Chaos is orchestrated using the following Kubernetes CRDs:

- **ChaosEngine**: A resource to link a Kubernetes application or Kubernetes node to a ChaosExperiment. ChaosEngine is watched by the Litmus ChaosOperator, which then invokes ChaosExperiments
- **ChaosExperiment**: A resource to group the configuration parameters of a chaos experiment. ChaosExperiment CRs are created by the operator when experiments are invoked by ChaosEngine.
- **ChaosResult**: A resource to hold the results of a ChaosExperiment.

### Getting started

We will create an Amazon EKS cluster with managed nodes. We’ll then install LitmusChaos and a demo application. Then, we will install chaos experiments to be run on the demo application and observe the behavior.

### Create EKS cluster

You will need the following to complete the project:

- [AWS CLI version 2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)
- [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
- [Helm](https://www.eksworkshop.com/beginner/060_helm/helm_intro/install/index.html)

Create a new EKS cluster using eksctl:

1. Open your terminal and run these commands on it
    
    ```bash
    export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
    export AWS_REGION=us-east-2 #change as per your region of choice
    ```


N:B: I had errors when using us-east-1 region due to no available resources in us-east-1e.

2. Create a file on your PC using this command

```bash
touch cluster.yaml
```

1. Edit the **cluster.yaml** file and place these codes in it , after that save and exit

```bash
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: eks-litmus-demo
  region: ${AWS_REGION}
  version: "1.21"
managedNodeGroups:
  - instanceType: m5.large
    amiFamily: AmazonLinux2
    name: eks-litmus-demo-ng
    desiredCapacity: 2
    minSize: 2
    maxSize: 4
```

![](/images5/yaml.png)

1. Run the below command on the terminal to create the cluster

eksctl create cluster -f cluster.yaml

![](/images5/create-cluster1.png)
![](/images5/create-cluster2.png)


### Install Helm

```bash
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

Verify Helm installation using the command below and confirm that you are using Helm version v3.X:

```bash
helm version --short
```

### Install LitmusChaos

Let’s install LitmusChaos on an Amazon EKS cluster using a Helm chart. The Helm chart will install the needed CRDs, service account configuration, and ChaosCenter.

Add the Litmus Helm repository using the command below:

```bash
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
```

Confirm that you have the Litmus-related Helm charts:

```bash
helm search repo litmuschaos
```

The output should look like below:

![](/images5/helm.png)

Create a namespace to install LitmusChaos.

```bash
kubectl create ns litmus
```

By default, Litmus Helm chart creates NodePort services. Let’s change the backend service type to ClusterIP and front-end service type to LoadBalancer, so we can access the Litmus ChaosCenter using a load balancer.

1. Create a file on your PC and name it **override-litmus.yaml**
2. Edit the **override-litmus.yaml file** and place the below code in it. 

```bash
portal:
  server:
    service:
      type: ClusterIP
  frontend:
    service:
      type: LoadBalancer
```

c.  Install the chaos via by running the below command

```
helm install chaos litmuschaos/litmus --namespace=litmus -f override-litmus.yaml
```

![](/images5/helm-install.png)

Verify that LitmusChaos is running:

```bash
kubectl get pods -n litmus
```

You should see a response similar to the one below:

![](/images5/pods.png)

`kubectl get svc -n litmus`

![](/images5/litmus.png)

export LITMUS_FRONTEND_SERVICE=`kubectl get svc chaos-litmus-frontend-service -n litmus --output jsonpath='{.status.loadBalancer.ingress[0].hostname}:{.spec.ports[0].port}'`echo "Litmus ChaosCenter is available at http://$LITMUS_FRONTEND_SERVICE"

The output should look like below:

```
bash: export: `Litmus ChaosCenter is available at http://xxxxxxxxxxxxxx-xxxxx4882.us-east-2.elb.amazonaws.com:9091'
```

Access Litmus ChaosCenter UI using the URL given above and sign in using the default username “admin” and password “litmus.”

![](/images5/litmus-web-pg.png)

After successful sign-in, you should see the welcome dashboard. Click on the ChaosAgents link from the left-hand navigation.

![](/images5/litmus-admin.png)

A ChaosAgent represents the target cluster where Chaos would be injected via Litmus. Confirm that Self-Agent is in Active status. Note: It may take a couple of minutes for the Self-Agent to become active.

![](/images5/chaos-agent.png)

Confirm the agent installation by running the command below.

```bash
kubectl get pods -n litmus
```

The output should look like below:

![](/images5/pods2.png)

Verify that LitmusChaos CRDs are created:

```bash
kubectl get crds | grep chaos
```

You should see a response similar to the one below showing chaosengines, chaosexperiments, and chaosresults.

![](/images5/grep-chaos.png)

Verify that LitmusChaos API resources are created:

```bash
kubectl api-resources | grep chaos
```

You should see a response similar to the one below:

![](/images5/api-resources.png)

Now that we installed LitmusChaos on the EKS cluster, let’s install a demo application to perform some chaos experiments on.

### Install demo application

Let’s deploy nginx on our cluster using the manifest below to run our chaos experiments on it. Save the manifest as nginx.yaml and apply it.

1. Create a file on your PC and name it **nginx.yaml** 
2. Edit **nginx.yaml** with the below code 

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 512Mi

```

c. Install the demo app by running the below command on your terminal 

```bash
kubectl apply -f nginx.yaml
```

Verify if the nginx pod is running by executing the command below.

![](/images5/nginx.png)

### Chaos Experiments

[Litmus ChaosHub](https://hub.litmuschaos.io/) is a public repository where LitmusChaos community members publish their chaos experiments such as **pod-delete,** **node-drain**, **node-cpu-hog**, etc. In this demo walkthrough, we will perform the **pod-autoscaler** experiment from LitmusChaos hub to test cluster auto scaling on Amazon EKS cluster.

### Experiment: Pod Autoscaler

The intent of the pod auto scaler experiment is to check the ability of nodes to accommodate the number of replicas for a deployment. Additionally, the experiment can also be used to check the cluster auto-scaling feature.

**Hypothesis**: Amazon EKS cluster should auto scale when cluster capacity is insufficient to run the pods.

Chaos experiment can be launched using the Litmus ChaosCenter UI by creating a workflow. Navigate to Litmus Chaos Center and select **Litmus Workflows** in the left-hand navigation and then select the **Schedule a workflow** button to create a workflow.

![](/images5/workflow.png)

Select the Self-Agent radio button on the Schedule a new Litmus workflow page and select Next.

![](/images5/workflow2.png)

Choose Create a new workflow using the experiments from ChaosHubs and leave the Litmus ChaosHub selected from the dropdown.

![](/images5/workflow3.png)

Enter a name for your workflow on the next screen.

![](/images5/workflow4.png)

Let’s add the experiments in the next step. Select Add a new experiment; then search for autoscaler and select the generic/pod-autoscaler radio button.

![](/images5/workflow5.png)
![](/images5/workflow6.png)

Let’s the edit the experiment and change some parameters. Choose the Edit icon:

![](/images5/workflow7.png)

Accept the default values in the General, Target Application, and Define the steady state for this application sections. In the Tune Experiment section, set the TOTAL_CHAOS_DURATION to 180 and REPLICA_COUNT to 10. TOTAL_CHAOS_DURATION sets the desired chaos duration in seconds and REPLICA_COUNT is the number of replicas to scale during the experiment. Select Finish.

![](/images5/workflow8.png)

Then, choose Next and accept the defaults for reliability score and schedule the experiment to run now. Finally, select Finish to run the chaos experiment.

![](/images5/workflow9.png)

The chaos experiment is now scheduled to run and you can look at the status by clicking on the workflow.

![](/images5/workflow10.png)

From the ChaosResults, you will see that the experiment failed because there was no capacity in the cluster to run 10 replicas.

![](/images5/chaos-fail.png)

### Install Cluster Autoscaler

Cluster Autoscaler for AWS provides integration with Auto Scaling groups. Cluster Autoscaler will attempt to determine the CPU, memory, and GPU resources provided by an Auto Scaling group based on the instance type specified in its launch configuration or launch template.

Create an IAM OIDC identity provider for your cluster with the following command.

```bash
eksctl utils associate-iam-oidc-provider --cluster eks-litmus-demo --approve
```

### Create an IAM policy and role

Create an IAM policy that grants the permissions that the Cluster Autoscaler requires to use an IAM role.

```bash

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}

aws iam create-policy \
    --policy-name AmazonEKSClusterAutoscalerPolicy \
    --policy-document file://cluster-autoscaler-policy.json
```

![](/images5/autoscaler-policy.png)

Create an IAM role and attach an IAM policy to it using eksctl.

```bash
eksctl create iamserviceaccount \
    --cluster=eks-litmus-demo \
    --namespace=kube-system \
    --name=cluster-autoscaler \
    --attach-policy-arn="arn:aws:iam::576516118645:policy/AmazonEKSClusterAutoscalerPolicy" \
    --override-existing-serviceaccounts \
    --approve
```

Make sure your service account with the ARN of the IAM role is annotated.

kubectl describe sa cluster-autoscaler -n kube-system

### Deploy the Cluster Autoscaler

Download the Cluster Autoscaler manifest.

```bash
curl -o cluster-autoscaler-autodiscover.yaml https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

Edit the downloaded file to replace <YOUR CLUSTER NAME> with the cluster name (eks-litmus-demo) and add the following two lines.

```bash
- --balance-similar-node-groups
- --skip-nodes-with-system-pods=false
```

The edited section should look like the following:

![](/images5/cluster-name.png)

Apply the manifest file to the cluster.

```bash
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

Patch the deployment to add the `cluster-autoscaler.kubernetes.io/safe-to-evict` annotation to the Cluster Autoscaler pods with the following command.

```bash
kubectl patch deployment cluster-autoscaler \
-n kube-system \
-p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'
```
![](/images5/patched.png)

Find the latest Cluster Autoscaler version that matches the Kubernetes major and minor versions of your cluster. For example, if the Kubernetes version of your cluster is 1.21, find the latest Cluster Autoscaler release that begins with 1.21. Record the semantic version number (1.21.n) for that release to use in the next step.

```bash
export K8S_VERSION=$(kubectl version --short | grep 'Server Version:' | sed 's/[^0-9.]*\([0-9.]*\).*/\1/' | cut -d. -f1,2)export AUTOSCALER_VERSION=$(curl -s "https://api.github.com/repos/kubernetes/autoscaler/releases" | grep '"tag_name":' | grep -m1 ${K8S_VERSION} | sed 's/[^0-9.]*\([0-9.]*\).*/\1/')
```

Set the Cluster Autoscaler image tag to the version that was exported in the previous step with the following command.

```bash
kubectl set image deployment cluster-autoscaler \
-n kube-system \
cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:${AUTOSCALER_VERSION}
```

After you have deployed the Cluster Autoscaler, you can view the logs and verify that it’s monitoring your cluster load.

View your Cluster Autoscaler logs with the following command.

```bash
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
```

Now that we have deployed the Cluster Autoscaler, let’s rerun the same experiment by navigating to Litmus Workflows, then the Schedules tab. Select the three dots menu icon for the workflow and select **Rerun Schedule**.

![](/images5/rerun.png)

This time, the Cluster Autoscaler will add additional nodes to the cluster, and the experiment will pass, which proves our hypothesis.

### Experiment Conclusion

Autoscaling the pod triggered the ClusterAautoscaler as a result of insufficient capacity, and a new node was added to the cluster, and the pods were successfully provisioned.

### Next steps

From the above walkthrough, we saw how to get started with Chaos Engineering using LitmusChaos on Amazon EKS cluster. There are additional experiments such as **pod-delete**, **node-drain**, **node-cpu-hog**, and so on that you can integrate with a CI/CD pipeline to perform Chaos Engineering. LitmusChaos also supports **gitops** and advanced chaos workflows using **Chaos Workflows**.

## pod-delete

Pod delete contains chaos to disrupt state of kubernetes resources. Experiments can inject random pod delete failures against specified application.

- Causes (forced/graceful) pod failure of random replicas of an application deployment.
- Tests deployment sanity (replica availability & uninterrupted service) and recovery workflows of the application pod.

**PRE-REQUISITE:**

**Install Litmus Operator**: a tool for injecting Chaos Experiments

## Installation[](https://docs.litmuschaos.io/docs/getting-started/installation/#installation)

Users looking to use Litmus for the first time have two options available to them today. One way is to use a hosted Litmus service like [ChaosNative Litmus Cloud](https://cloud.chaosnative.com/). Alternatively, users looking for some more flexibility can install Litmus into their own Kubernetes cluster.

Users choosing the self-hosted option can refer to our Install and Configure docs for installing alternate versions and more detailed instructions.

- Self-Hosted
- Hosted (Beta)

Installation of Self-Hosted Litmus can be done using either of the below 

**methods** :

[Helm3](https://docs.litmuschaos.io/docs/getting-started/installation/#install-litmus-using-helm) chart 

[Kubectl](https://docs.litmuschaos.io/docs/getting-started/installation/#install-litmus-using-kubectl) yaml spec file. 

Refer to the below details for Self-Hosted Litmus installation.

### Install Litmus using Helm[](https://docs.litmuschaos.io/docs/getting-started/installation/#install-litmus-using-helm)

The helm chart will install all the required service account configuration and ChaosCenter.

The following steps will help you install Litmus ChaosCenter via helm.

### Add the litmus helm repository[](https://docs.litmuschaos.io/docs/getting-started/installation/#step-1-add-the-litmus-helm-repository)

```bash
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/helm repo list
```

### Create the namespace on which you want to install Litmus ChaosCenter[](https://docs.litmuschaos.io/docs/getting-started/installation/#step-2-create-the-namespace-on-which-you-want-to-install-litmus-chaoscenter)

- The ChaosCenter can be placed in any namespace, but for this scenario we are choose `litmus` as the namespace.

```bash
kubectl create ns litmus
```

### Install Litmus ChaosCenter[](https://docs.litmuschaos.io/docs/getting-started/installation/#step-3-install-litmus-chaoscenter)

```bash
helm install chaos litmuschaos/litmus --namespace=litmus
```

**Expected Output**

```bash
NAME: chaosLAST DEPLOYED: Tue Jun 15 19:20:09 2021NAMESPACE: litmusSTATUS: deployedREVISION: 1TEST SUITE: NoneNOTES:Thank you for installing litmus 😀Your release is named chaos and its installed to namespace: litmus.Visit https://docs.litmuschaos.io to find more info.
```

> **Note**: Litmus uses Kubernetes CRDs to define chaos intent. Helm3 handles CRDs better than Helm2. Before you start running a chaos experiment, verify if Litmus is installed correctly.
> 

### **Install Litmus using kubectl**[](https://docs.litmuschaos.io/docs/getting-started/installation/#install-litmus-using-kubectl)

### **Install Litmus ChaosCenter**[](https://docs.litmuschaos.io/docs/getting-started/installation/#install-litmus-chaoscenter)

Applying the manifest file will install all the required service account configuration and ChaosCenter.

`kubectl apply -f https://litmuschaos.github.io/litmus/2.6.0/litmus-2.6.0.yaml`

---

## **Verify your installation**[](https://docs.litmuschaos.io/docs/getting-started/installation/#verify-your-installation)

### **Verify if the frontend, server, and database pods are running**[](https://docs.litmuschaos.io/docs/getting-started/installation/#verify-if-the-frontend-server-and-database-pods-are-running)

- Check the pods in the namespace where you installed Litmus:
    
    **Expected Output**
    
    `kubectl get pods -n litmus`
    
    `NAME                                    READY   STATUS  RESTARTS  AGElitmusportal-frontend-97c8bf86b-mx89w   1/1     Running 2         6m24slitmusportal-server-5cfbfc88cc-m6c5j    2/2     Running 2         6m19smongo-0                                 1/1     Running 0         6m16s`
    
- Check the services running in the namespace where you installed Litmus:
    
    **Expected Output**
    
    `kubectl get svc -n litmus`
    
    `NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP PORT(S)                       AGElitmusportal-frontend-service   NodePort    10.100.105.154  <none>      9091:30229/TCP                7m14slitmusportal-server-service     NodePort    10.100.150.175  <none>      9002:30479/TCP,9003:31949/TCP 7m8smongo-service                   ClusterIP   10.100.226.179  <none>      27017/TCP                     7m6s`
    

---

## **Accessing the ChaosCenter**[](https://docs.litmuschaos.io/docs/getting-started/installation/#accessing-the-chaoscenter)

To setup and login to ChaosCenter expand the available services just created and copy the `PORT` of the `litmusportal-frontend-service` service

`kubectl get svc -n litmus`

**Expected Output**

`NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGEchaos-litmus-portal-mongo       ClusterIP   10.104.107.117   <none>        27017/TCP                       2mlitmusportal-frontend-service   NodePort    10.101.81.70     <none>        9091:30385/TCP                  2mlitmusportal-server-service     NodePort    10.108.151.79    <none>        9002:32456/TCP,9003:31160/TCP   2m`Copy

> **Note**: In this case, the PORT for litmusportal-frontend-service is 30385. Yours will be different.
> 

Once you have the PORT copied in your clipboard, simply use your IP and PORT in this manner `<NODEIP>:<PORT>` to access the Litmus ChaosCenter.

For example:

`http://172.17.0.3:30385/`

> Where 172.17.0.3 is my NodeIP and 30385 is the frontend service PORT. If using a LoadBalancer, the only change would be to provide a <LoadBalancerIP>:<PORT>. Learn more about how to access ChaosCenter with LoadBalancer
> 

You should be able to see the Login Page of Litmus ChaosCenter. The **default credentials** are

`Username: admin` 

`Password: litmus`

![](/images5/litmus-admin.png)

By default you are assigned with a default project with Owner permissions.

## **Verify Successful Registration of the Self Agent**[](https://docs.litmuschaos.io/docs/getting-started/installation/#verify-successful-registration-of-the-self-agent)

Once the project is created, the cluster is automatically registered as a chaos target via installation of [ChaosAgents](https://docs.litmuschaos.io/docs/getting-started/resources#chaosagents). This is represented as [Self-Agent](https://docs.litmuschaos.io/docs/getting-started/resources#types-of-chaosagents) in [ChaosCenter](https://docs.litmuschaos.io/docs/getting-started/resources#chaoscenter).

`kubectl get pods -n litmus`

`NAME                                     READY   STATUS    RESTARTS   AGEchaos-exporter-547b59d887-4dm58          1/1     Running   0          5m27schaos-operator-ce-84ddc8f5d7-l8c6d       1/1     Running   0          5m27sevent-tracker-5bc478cbd7-xlflb           1/1     Running   0          5m28slitmusportal-frontend-97c8bf86b-mx89w    1/1     Running   0          15mlitmusportal-server-5cfbfc88cc-m6c5j     2/2     Running   1          15mmongo-0                                  1/1     Running   0          15msubscriber-958948965-qbx29               1/1     Running   0          5m30sworkflow-controller-78fc7b6c6-w82m7      1/1     Running   0          5m32s`

**Install this Chaos Experiment**

You can install the Chaos Experiment using the following command

```bash
kubectl apply -f https://hub.litmuschaos.io/api/chaos/2.6.0?file=charts/generic/pod-delete/experiment.yaml
```

In case you want to customize or download the yaml, you can use  the code below

```bash
apiVersion: litmuschaos.io/v1alpha1
description:
  message: |
    Deletes a pod belonging to a deployment/statefulset/daemonset
kind: ChaosExperiment
metadata:
  name: pod-delete
  labels:
    name: pod-delete
    app.kubernetes.io/part-of: litmus
    app.kubernetes.io/component: chaosexperiment
    app.kubernetes.io/version: 2.6.0
spec:
  definition:
    scope: Namespaced
    permissions:
      # Create and monitor the experiment & helper pods
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["create","delete","get","list","patch","update", "deletecollection"]
      # Performs CRUD operations on the events inside chaosengine and chaosresult
      - apiGroups: [""]
        resources: ["events"]
        verbs: ["create","get","list","patch","update"]
      # Fetch configmaps details and mount it to the experiment pod (if specified)
      - apiGroups: [""]
        resources: ["configmaps"]
        verbs: ["get","list",]
      # Track and get the runner, experiment, and helper pods log 
      - apiGroups: [""]
        resources: ["pods/log"]
        verbs: ["get","list","watch"]  
      # for creating and managing to execute comands inside target container
      - apiGroups: [""]
        resources: ["pods/exec"]
        verbs: ["get","list","create"]
      # deriving the parent/owner details of the pod(if parent is anyof {deployment, statefulset, daemonsets})
      - apiGroups: ["apps"]
        resources: ["deployments","statefulsets","replicasets", "daemonsets"]
        verbs: ["list","get"]
      # deriving the parent/owner details of the pod(if parent is deploymentConfig)  
      - apiGroups: ["apps.openshift.io"]
        resources: ["deploymentconfigs"]
        verbs: ["list","get"]
      # deriving the parent/owner details of the pod(if parent is deploymentConfig)
      - apiGroups: [""]
        resources: ["replicationcontrollers"]
        verbs: ["get","list"]
      # deriving the parent/owner details of the pod(if parent is argo-rollouts)
      - apiGroups: ["argoproj.io"]
        resources: ["rollouts"]
        verbs: ["list","get"]
      # for configuring and monitor the experiment job by the chaos-runner pod
      - apiGroups: ["batch"]
        resources: ["jobs"]
        verbs: ["create","list","get","delete","deletecollection"]
      # for creation, status polling and deletion of litmus chaos resources used within a chaos workflow
      - apiGroups: ["litmuschaos.io"]
        resources: ["chaosengines","chaosexperiments","chaosresults"]
        verbs: ["create","list","get","patch","update","delete"]
    image: "litmuschaos/go-runner:2.6.0"
    imagePullPolicy: Always
    args:
    - -c
    - ./experiments -name pod-delete
    command:
    - /bin/bash
    env:

    - name: TOTAL_CHAOS_DURATION
      value: '15'

    # Period to wait before and after injection of chaos in sec
    - name: RAMP_TIME
      value: ''

    - name: FORCE
      value: 'true'

    - name: CHAOS_INTERVAL
      value: '5'

    ## percentage of total pods to target
    - name: PODS_AFFECTED_PERC
      value: ''

    - name: LIB
      value: 'litmus'    

    - name: TARGET_PODS
      value: ''

    ## it defines the sequence of chaos execution for multiple target pods
    ## supported values: serial, parallel
    - name: SEQUENCE
      value: 'parallel'
      
    labels:
      name: pod-delete
      app.kubernetes.io/part-of: litmus
      app.kubernetes.io/component: experiment-job
      app.kubernetes.io/version: 2.6.0
```

**Setup Service Account (RBAC)**

Create a service account using the following command

```bash
**kubectl apply -f https://hub.litmuschaos.io/api/chaos/2.6.0?file=charts/generic/pod-delete/rbac.yaml**
```

In case you want to customize or download the yaml,  you can use  the code below

```bash
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-delete-sa
  namespace: default
  labels:
    name: pod-delete-sa
    app.kubernetes.io/part-of: litmus
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-delete-sa
  namespace: default
  labels:
    name: pod-delete-sa
    app.kubernetes.io/part-of: litmus
rules:
  # Create and monitor the experiment & helper pods
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update", "deletecollection"]
  # Performs CRUD operations on the events inside chaosengine and chaosresult
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create","get","list","patch","update"]
  # Fetch configmaps details and mount it to the experiment pod (if specified)
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get","list",]
  # Track and get the runner, experiment, and helper pods log 
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]  
  # for creating and managing to execute comands inside target container
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["get","list","create"]
  # deriving the parent/owner details of the pod(if parent is anyof {deployment, statefulset, daemonsets})
  - apiGroups: ["apps"]
    resources: ["deployments","statefulsets","replicasets", "daemonsets"]
    verbs: ["list","get"]
  # deriving the parent/owner details of the pod(if parent is deploymentConfig)  
  - apiGroups: ["apps.openshift.io"]
    resources: ["deploymentconfigs"]
    verbs: ["list","get"]
  # deriving the parent/owner details of the pod(if parent is deploymentConfig)
  - apiGroups: [""]
    resources: ["replicationcontrollers"]
    verbs: ["get","list"]
  # deriving the parent/owner details of the pod(if parent is argo-rollouts)
  - apiGroups: ["argoproj.io"]
    resources: ["rollouts"]
    verbs: ["list","get"]
  # for configuring and monitor the experiment job by the chaos-runner pod
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["create","list","get","delete","deletecollection"]
  # for creation, status polling and deletion of litmus chaos resources used within a chaos workflow
  - apiGroups: ["litmuschaos.io"]
    resources: ["chaosengines","chaosexperiments","chaosresults"]
    verbs: ["create","list","get","patch","update","delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-delete-sa
  namespace: default
  labels:
    name: pod-delete-sa
    app.kubernetes.io/part-of: litmus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-delete-sa
subjects:
- kind: ServiceAccount
  name: pod-delete-sa
  namespace: default
```

**Sample Chaos Engine**

Create a file and name it **engine.yaml**

Place the below code in the **engine.yaml** file

```bash
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: nginx-chaos
  namespace: default
spec:
  appinfo:
    appns: 'default'
    applabel: 'app=nginx'
    appkind: 'deployment'
  # It can be active/stop
  engineState: 'active'
  chaosServiceAccount: pod-delete-sa
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            # set chaos duration (in sec) as desired
            - name: TOTAL_CHAOS_DURATION
              value: '30'

            # set chaos interval (in sec) as desired
            - name: CHAOS_INTERVAL
              value: '10'
              
            # pod failures without '--force' & default terminationGracePeriodSeconds
            - name: FORCE
              value: 'false'

             ## percentage of total pods to target
            - name: PODS_AFFECTED_PERC
              value: ''
```

Once you download the yaml you can apply the yaml using the below command

```bash
**kubectl apply -f engine.yaml**
```

### Node-drain

Drain the node where application pod is scheduled

![](/images5/chaos-process.png)

**PRE-REQUISITE:**

**Install Litmus Operator**: a tool for injecting Chaos Experiments

**Install this Chaos Experiment**

You can install the Chaos Experiment using the following command

```bash
**kubectl apply -f https://hub.litmuschaos.io/api/chaos/2.6.0?file=charts/generic/node-drain/experiment.yaml**
```

In case you want to customize or download the yaml you can use  the code below

```bash
---
apiVersion: litmuschaos.io/v1alpha1
description:
  message: |
    Drain the node where application pod is scheduled
kind: ChaosExperiment
metadata:
  name: node-drain
  labels:
    name: node-drain
    app.kubernetes.io/part-of: litmus
    app.kubernetes.io/component: chaosexperiment
    app.kubernetes.io/version: 2.6.0
spec:
  definition:
    scope: Cluster
    permissions:
      # Create and monitor the experiment & helper pods
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["create","delete","get","list","patch","update", "deletecollection"]
      # Performs CRUD operations on the events inside chaosengine and chaosresult
      - apiGroups: [""]
        resources: ["events"]
        verbs: ["create","get","list","patch","update"]
      # Fetch configmaps details and mount it to the experiment pod (if specified)
      - apiGroups: [""]
        resources: ["configmaps"]
        verbs: ["get","list",]
      # Track and get the runner, experiment, and helper pods log 
      - apiGroups: [""]
        resources: ["pods/log"]
        verbs: ["get","list","watch"]  
      # for creating and managing to execute comands inside target container
      - apiGroups: [""]
        resources: ["pods/exec","pods/eviction"]
        verbs: ["get","list","create"]
      # ignore daemonsets while draining the node
      - apiGroups: ["apps"]
        resources: ["daemonsets"]
        verbs: ["list","get","delete"]
      # for configuring and monitor the experiment job by the chaos-runner pod
      - apiGroups: ["batch"]
        resources: ["jobs"]
        verbs: ["create","list","get","delete","deletecollection"]
      # for creation, status polling and deletion of litmus chaos resources used within a chaos workflow
      - apiGroups: ["litmuschaos.io"]
        resources: ["chaosengines","chaosexperiments","chaosresults"]
        verbs: ["create","list","get","patch","update","delete"]
      # for experiment to perform node status checks
      - apiGroups: [""]
        resources: ["nodes"]
        verbs: ["get","list","patch"]
    image: "litmuschaos/go-runner:2.6.0"
    imagePullPolicy: Always
    args:
    - -c
    - ./experiments -name node-drain
    command:
    - /bin/bash
    env:
    
    - name: TARGET_NODE
      value: ''

    - name: NODE_LABEL
      value: ''

    - name: TOTAL_CHAOS_DURATION
      value: '60'

    # Provide the LIB here
    # Only litmus supported
    - name: LIB
      value: 'litmus'

    # Period to wait before and after injection of chaos in sec
    - name: RAMP_TIME
      value: ''
      
    labels:
      name: node-drain
      app.kubernetes.io/part-of: litmus
      app.kubernetes.io/component: experiment-job
      app.kubernetes.io/version: 2.6.0
```

**Setup Service Account (RBAC)**

Create a service account using the following command

```bash
**kubectl apply -f https://hub.litmuschaos.io/api/chaos/2.6.0?file=charts/generic/node-drain/rbac.yaml**
```

In case you want to customize or download the yaml,  you can use from the code below

```bash
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-drain-sa
  namespace: default
  labels:
    name: node-drain-sa
    app.kubernetes.io/part-of: litmus
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-drain-sa
  labels:
    name: node-drain-sa
    app.kubernetes.io/part-of: litmus
rules:
 # Create and monitor the experiment & helper pods
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update", "deletecollection"]
  # Performs CRUD operations on the events inside chaosengine and chaosresult
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create","get","list","patch","update"]
  # Fetch configmaps details and mount it to the experiment pod (if specified)
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get","list",]
  # Track and get the runner, experiment, and helper pods log 
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]  
  # for creating and managing to execute comands inside target container
  - apiGroups: [""]
    resources: ["pods/exec","pods/eviction"]
    verbs: ["get","list","create"]
  # ignore daemonsets while draining the node
  - apiGroups: ["apps"]
    resources: ["daemonsets"]
    verbs: ["list","get","delete"]
  # for configuring and monitor the experiment job by the chaos-runner pod
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["create","list","get","delete","deletecollection"]
  # for creation, status polling and deletion of litmus chaos resources used within a chaos workflow
  - apiGroups: ["litmuschaos.io"]
    resources: ["chaosengines","chaosexperiments","chaosresults"]
    verbs: ["create","list","get","patch","update","delete"]
  # for experiment to perform node status checks
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get","list","patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-drain-sa
  labels:
    name: node-drain-sa
    app.kubernetes.io/part-of: litmus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-drain-sa
subjects:
- kind: ServiceAccount
  name: node-drain-sa
  namespace: default
```

Create a file and name it generic**.yaml**

Place the below code in the generic**.yaml** file

```bash
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: nginx-chaos
  namespace: default
spec:
  # It can be active/stop
  engineState: 'active'
  #ex. values: ns1:name=percona,ns2:run=nginx 
  auxiliaryAppInfo: ''
  chaosServiceAccount: node-drain-sa
  experiments:
    - name: node-drain
      spec:
        components:
        # nodeSelector: 
        #   # provide the node labels
        #   kubernetes.io/hostname: 'node02'        
          env:
            - name: TOTAL_CHAOS_DURATION
              value: '60'
              
            # enter the target node name
            - name: TARGET_NODE
              value: ''
```

Once you download the yaml you can apply the yaml using the below command

```bash
**kubectl apply -f generic.yaml**
```

# **node-cpu-hog**

Node CPU hog contains chaos to disrupt the state of Kubernetes resources.

Node CPU hog contains chaos to disrupt the state of Kubernetes resources. Experiments can inject a CPU spike on a node where the application pod is scheduled.

- CPU hog on a particular node where the application deployment is available.
- After test, the recovery should be manual for the application pod and node in case they are not in an appropriate state.

**PRE-REQUISITE:**

**Install Litmus Operator**: a tool for injecting Chaos Experiments

**Install this Chaos Experiment**

You can install the Chaos Experiment using the following command

```bash
**kubectl apply -f https://hub.litmuschaos.io/api/chaos/2.6.0?file=charts/generic/node-cpu-hog/experiment.yaml**
```

In case you want to customize or download the yaml,  you can use from the code below

```bash
apiVersion: litmuschaos.io/v1alpha1
description:
  message: |
    Give a cpu spike on a node belonging to a deployment
kind: ChaosExperiment
metadata:
  name: node-cpu-hog
  labels:
    name: node-cpu-hog
    app.kubernetes.io/part-of: litmus
    app.kubernetes.io/component: chaosexperiment
    app.kubernetes.io/version: 2.6.0
spec:
  definition:
    scope: Cluster
    permissions:
      # Create and monitor the experiment & helper pods
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["create","delete","get","list","patch","update", "deletecollection"]
      # Performs CRUD operations on the events inside chaosengine and chaosresult
      - apiGroups: [""]
        resources: ["events"]
        verbs: ["create","get","list","patch","update"]
      # Fetch configmaps details and mount it to the experiment pod (if specified)
      - apiGroups: [""]
        resources: ["configmaps"]
        verbs: ["get","list",]
      # Track and get the runner, experiment, and helper pods log 
      - apiGroups: [""]
        resources: ["pods/log"]
        verbs: ["get","list","watch"]  
      # for creating and managing to execute comands inside target container
      - apiGroups: [""]
        resources: ["pods/exec"]
        verbs: ["get","list","create"]
      # for configuring and monitor the experiment job by the chaos-runner pod
      - apiGroups: ["batch"]
        resources: ["jobs"]
        verbs: ["create","list","get","delete","deletecollection"]
      # for creation, status polling and deletion of litmus chaos resources used within a chaos workflow
      - apiGroups: ["litmuschaos.io"]
        resources: ["chaosengines","chaosexperiments","chaosresults"]
        verbs: ["create","list","get","patch","update","delete"]
      # for experiment to perform node status checks
      - apiGroups: [""]
        resources: ["nodes"]
        verbs: ["get","list"]
    image: "litmuschaos/go-runner:2.6.0"
    imagePullPolicy: Always
    args:
    - -c
    - ./experiments -name node-cpu-hog
    command:
    - /bin/bash
    env:

    - name: TOTAL_CHAOS_DURATION
      value: '60'

    # Period to wait before and after injection of chaos in sec
    - name: RAMP_TIME
      value: ''

    ## ENTER THE NUMBER OF CORES OF CPU FOR CPU HOGGING
    ## OPTIONAL VALUE IN CASE OF EMPTY VALUE IT WILL TAKE NODE CPU CAPACITY 
    - name: NODE_CPU_CORE
      value: ''

    ## LOAD CPU WITH GIVEN PERCENT LOADING FOR THE CPU STRESS WORKERS. 
    ## 0 IS EFFECTIVELY A SLEEP (NO LOAD) AND 100 IS FULL LOADING
    - name: CPU_LOAD
      value: '100'

    # ENTER THE COMMA SEPARATED TARGET NODES NAME
    - name: TARGET_NODES
      value: ''

    - name: NODE_LABEL
      value: ''

    # PROVIDE THE LIB HERE
    # ONLY LITMUS SUPPORTED
    - name: LIB
      value: 'litmus'

    # provide lib image
    - name: LIB_IMAGE
      value: 'litmuschaos/go-runner:2.6.0' 

    ## percentage of total nodes to target
    - name: NODES_AFFECTED_PERC
      value: ''

    ## it defines the sequence of chaos execution for multiple target nodes
    ## supported values: serial, parallel
    - name: SEQUENCE
      value: 'parallel'
   
    labels:
      name: node-cpu-hog
      app.kubernetes.io/part-of: litmus
      app.kubernetes.io/component: experiment-job
      app.kubernetes.io/version: 2.6.0
```

**Setup Service Account (RBAC)**

Create a service account using the following command

```bash
kubectl apply -f https://hub.litmuschaos.io/api/chaos/2.6.0?file=charts/generic/node-cpu-hog/rbac.yaml
```

In case you want to customize or download the yaml,  you can use from the code below

```bash
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-cpu-hog-sa
  namespace: default
  labels:
    name: node-cpu-hog-sa
    app.kubernetes.io/part-of: litmus
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-cpu-hog-sa
  labels:
    name: node-cpu-hog-sa
    app.kubernetes.io/part-of: litmus
rules:
  # Create and monitor the experiment & helper pods
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update", "deletecollection"]
  # Performs CRUD operations on the events inside chaosengine and chaosresult
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create","get","list","patch","update"]
  # Fetch configmaps details and mount it to the experiment pod (if specified)
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get","list",]
  # Track and get the runner, experiment, and helper pods log 
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]  
  # for creating and managing to execute comands inside target container
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["get","list","create"]
  # for configuring and monitor the experiment job by the chaos-runner pod
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["create","list","get","delete","deletecollection"]
  # for creation, status polling and deletion of litmus chaos resources used within a chaos workflow
  - apiGroups: ["litmuschaos.io"]
    resources: ["chaosengines","chaosexperiments","chaosresults"]
    verbs: ["create","list","get","patch","update","delete"]
  # for experiment to perform node status checks
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-cpu-hog-sa
  labels:
    name: node-cpu-hog-sa
    app.kubernetes.io/part-of: litmus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: node-cpu-hog-sa
subjects:
- kind: ServiceAccount
  name: node-cpu-hog-sa
  namespace: default
```

Create a file and name it node-cpu**.yaml**

Place the below code in the node-cpu**.yaml** file

```bash
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: nginx-chaos
  namespace: default
spec:
  # It can be active/stop
  engineState: 'active'
  #ex. values: ns1:name=percona,ns2:run=nginx 
  auxiliaryAppInfo: ''
  chaosServiceAccount: node-cpu-hog-sa
  experiments:
    - name: node-cpu-hog
      spec:
        components:
          env:
            # set chaos duration (in sec) as desired
            - name: TOTAL_CHAOS_DURATION
              value: '60'
            
            - name: NODE_CPU_CORE
              value: ''
            
            ## percentage of total nodes to target
            - name: NODES_AFFECTED_PERC
              value: ''

            # provide the comma separated target node names
            - name: TARGET_NODES
              value: ''
```

Once you download the yaml you can apply the yaml using the below command

```bash
**kubectl apply -f node-cpu.yaml**
```