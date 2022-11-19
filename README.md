# AWS EKS Cluster Autoscaling
This repo demonstrates the cluster autoscaling feature of K8S. Together with HPA a production level K8S cluster can be deployed in AWS EKS with very less operational/management overhead. <br/><br/>
* To adjust to changing application demands clusters often need a way to automatically scale. In AWS EKS clusters automatic scaling is done primarily in one of two ways: <br/>
  * The cluster autoscaler watches for pods that can't be scheduled on nodes because of resource constraints. The cluster then automatically increases the number of nodes. <br/>
    ![image](https://user-images.githubusercontent.com/92582005/202852434-dbf37c5a-e2a7-4783-b379-dbb693d729bd.png) <br/>
  * Kubernetes uses the horizontal pod autoscaler (HPA) to monitor the resource demand and automatically scale the number of replicas. By default, the horizontal pod autoscaler checks the Metrics API. The horizontal pod autoscaler uses the Metrics Server in a Kubernetes cluster to monitor the resource demand of pods. If an application needs more resources, the number of pods is automatically increased to meet the demand.<br/>
  ![image](https://user-images.githubusercontent.com/92582005/202852404-b34021e2-856d-440b-8e0c-eaaf0fc5322f.png) <br/>
