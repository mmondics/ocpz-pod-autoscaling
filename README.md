# OpenShift Pod Autoscaling on IBM Z and LinuxONE

## Table of Contents

- [OpenShift Pod Autoscaling on IBM Z and LinuxONE](#openshift-pod-autoscaling-on-ibm-z-and-linuxone)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
  - [Overview of Pod Autoscaling](#overview-of-pod-autoscaling)
  - [Horizontal Pod Autoscaling](#horizontal-pod-autoscaling)
  - [Vertical Pod Autoscaling](#vertical-pod-autoscaling)

## Prerequisites

1. OpenShift 4.10+ on IBM Z or LinuxONE
1. Cluster administrator privileges
1. Basic OpenShift knowledge including how to log into OpenShift through the web console and CLI, navigation and use of each.

## Overview of Pod Autoscaling
Scalability is one of the primary reasons the IT world is moving large portions of its workloads to Kubernetes and container platforms such as Red Hat OpenShift. Kubernetes' ability to rapidly scale containerized applications to suit user demand provides developers and cloud operators the flexibility to deploy an application without knowing exactly how many resources will be consumed or when they will be consumed. 

Two of the ways that Kubernetes provides rapid scalability are Horizontal Pod Autoscaling and Vertical Pod Autoscaling. 

**Horizontal Pod Autoscaling** automatically changes the *number of Pods* in a workload resource, such as a Deployment. When Kubernetes notices that there is an increased load on an application, it will automatically scale up Pods, and when the load decreases, it will automatically scale down the number of Pods.

**Vertical Pod Autoscaling**, on the other hand, will *automatically adjust the resource requests and limits* for a workload resource, such as a Deployment. Instead of scaling the number of Pods up or down, the number of Pods stays the same, but each is allowed to use more CPU or memory than originally set.

With OpenShift 4.10, both the Horizontal and Vertical Pod Autoscalers are now [supported on IBM zSystems and LinuxONE OpenShift clusters](https://docs.openshift.com/container-platform/4.10/release_notes/ocp-4-10-release-notes.html#ocp-4-10-ibm-z).

## Horizontal Pod Autoscaling
Operators and cloud administrators might be hesitant to give an OpenShift application free reign to scale its number of pods based on user demand. Imagine if a business critial application is suddenly hit with far more requests than normal due to an event like Black Friday shopping, a viral video that references a webpage, or a malicious DDOS attack. One can imagine thousands of pods being scaled up, surpassing the quantity that can be supported by the OpenShift cluster or the hardware it's running on. 

Fortunately, as you'll see in the rest of this section, the OpenShift (Kubernetes) Pod Autoscalers require certain parameters be set that avoid the possiblity of catastrophe, such as maximum/minimun number of Pods, resource limits, resource requests, and percentage of resource requests that prompts the Autoscaler to take action.

To get started with the demonstration, create an application on the OpenShift cluster that will be used to show the autoscaling functions. 

*Please note that all of the following steps can be completed either in the OpenShift web console or through the OpenShift command line. This demonstration will swap between the two methods.*

1. Log into your OpenShift cluster using OpenShift command line

  You can find credentials and all necessary information from the *OpenShift on IBM Z - Bastion with Single Master Node* tile in your TechZone [My Reservations Page](https://techzone.ibm.com/my/reservations). Further details can be found within your *Project Kit*, which is linked on your reservation page.

1. Create a new project:

  ```text
  oc new-project autoscaling-demo
  ```

1. In the new project, deploy an Open Liberty container image:

  ```text
  oc new-app --image=quay.io/mmondics/open-liberty:latest --name=open-liberty
  ```

  <details>
  <summary>Click to expand sample output</summary>

  ```text
  --> Found container image 0805ec4 (5 days old) from quay.io for "quay.io/mmondics/open-liberty:latest"

  * An image stream tag will be created as "open-liberty:latest" that will track this image
  
  --> Creating resources ...
      imagestream.image.openshift.io "open-liberty" created
      deployment.apps "open-liberty" created
      service "open-liberty" created
  --> Success
      Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
      'oc expose service/open-liberty'
      Run 'oc status' to view your app.
  ```
  </details>

1. Expose the Service to create a Route:

  ```text
  oc expose service/open-liberty
  ```

  By default, you have one Open Liberty Pod running in your project that is responsible for all requests that hit the application.

1. Look at your one pod:

  ```text
  oc get pods
  ```

  ```text
  NAME                           READY   STATUS    RESTARTS   AGE
  open-liberty-96db8b4b5-x2ncc   1/1     Running   0          8m20s 
  ```

  Because there is only one replica of the Open Liberty application running, it is solely responsible for all requests that come into the cluster through its exposed Route. Furthermore, because no resource requests or limits have been set for this application, it has the capability to use as many resources as it needs with no upper bound. This is clearly not a viable setup for production workloads or any situation where high availability and performance are needed. For this reason, setting resource requests and limits is considered a best practice. Furthermore, they are required for horizontal and vertical Pod autoscaling, so the next step is to set these for the Open Liberty Deployment.

1. Navigate to your OpenShift web console and find your Open Liberty Deployment. 

  Under the *Administrator* perspective, navigate to *Worklaods* -> *Deployments* and make sure you're in the *autoscaling-demo* project.

  ![autoscaling-deployment](/images/autoscaling-deployment.png)
  
1. Click on the three dots associated with the Open Liberty Deployment, then click Edit resource limits:

  ![autoscaling-deployment-2](images/autoscaling-deployment-2.png)

1. Set the following resource requests and limits:

  - CPU Request: 5 millicores
  - CPU Limit: 10 millicores
  - Memory Request: 250 Mi
  - Memory Limit: 500 Mi

  ![settling-resource-limits](/images/setting-resource-limits.png)

  These requests and limits tell OpenShift that each Pod in the Open Liberty Deployment is guaranteed access to 2 millicores of CPU and 250 Mi of memory. On the other hand, each Pod has a limit of 4 millicores CPU and 500 Mi memory. For more information about Kubernetes resource requests and limmits, you can read the documentation [here](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/). 

1. Click Save for the changes to be applied to the Deployment and for a new Pod to spin up.

  Next, you will need to create a HorizontalPodAutoscaler that defines the boundries and triggers for autoscaling.

1. Create a HorizontalPodAutoscaler using either the OpenShift console or CLI with the following parameters:

  - Minimum Replicas: 1
  - Maximum Replicas: 5
  - Target CPU Percent: 10%

  *Note that the target CPU percentage refers to the desired percent of the CPU Request (5 milicores) for each pod. This is intentionally set very low so as to ensure the deployment will scale up to 5 replicas without the need to simulate workload on the application.*

  You can create this in one of two ways. In the OpenShift CLI, you can run the following command:

  ```text
  oc autoscale deployment open-liberty --min=1 --max=5 --cpu-percent=10
  ```

  In the OpenShift web console, you can click the three dots associated with the Open Liberty Deployment once again, then select Add HorizontalPodAutoscaler and fill in the fields as appropriate.

  ![add-hpa-console](/images/add-hpa-console.png)

1. Navigate to the Pods page in the OpenShift console to watch new Pods being created.

  ![autoscaling-pods](/images/autoscaling-pods.png)

  Alternatively, you can run the command `watch oc get pods` in the CLI.

  ```text
  NAME                            READY   STATUS              RESTARTS   AGE
  open-liberty-69cbbf58c9-5pnrh   1/1     Running             0          22s
  open-liberty-69cbbf58c9-7f8vw   1/1     Running             0          53m
  open-liberty-69cbbf58c9-bnqgc   1/1     Running             0          22s
  open-liberty-69cbbf58c9-fg8xl   0/1     ContainerCreating   0          7s
  open-liberty-69cbbf58c9-zkb44   1/1     Running             0          22s
  ```

  Because each pod is using much more than 10% of the CPU resource request (5 Mi), OpenShift will scale up the deployment to the set maximum of 5 pods. If you noticed that it took a little while for the 

The HorizontalPodAutoscaler object created in the autoscaling-demo project can be modified with more parameters to customize the behavior of the autoscaling. For example, Scaling Policies can be used to control the rate at which the pods scale up or down by setting a specific number or specific percentage to scale in a specified period of time. You can read more about HPA customization in the OpenShift documentation [here](https://docs.openshift.com/container-platform/4.10/nodes/pods/nodes-pods-autoscaling.html#nodes-pods-autoscaling-best-practices-hpa_nodes-pods-autoscaling).

## Vertical Pod Autoscaling
