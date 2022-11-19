# AWS EKS Cluster Autoscaling
This repo demonstrates the cluster autoscaling feature of K8S. Together with HPA a production level K8S cluster can be deployed in AWS EKS with very less operational/management overhead. <br/><br/>
* To adjust to changing application demands clusters often need a way to automatically scale. In AWS EKS clusters automatic scaling is done primarily in one of two ways: <br/>
  * The cluster autoscaler watches for pods that can't be scheduled on nodes because of resource constraints. The cluster then automatically increases the number of nodes. <br/>
    ![image](https://user-images.githubusercontent.com/92582005/202852434-dbf37c5a-e2a7-4783-b379-dbb693d729bd.png) <br/>
  * Kubernetes uses the horizontal pod autoscaler (HPA) to monitor the resource demand and automatically scale the number of replicas. By default, the horizontal pod autoscaler checks the Metrics API. The horizontal pod autoscaler uses the Metrics Server in a Kubernetes cluster to monitor the resource demand of pods. If an application needs more resources, the number of pods is automatically increased to meet the demand.<br/>
  ![image](https://user-images.githubusercontent.com/92582005/202852404-b34021e2-856d-440b-8e0c-eaaf0fc5322f.png) <br/>
* [This official AWS Documentation mentions the steps to implement EKS Cluster Autoscaling. In this article we will use Kubernetes Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html) <br/>
* Some steps does not work properly from the above documentation which will be highlighted here <br/>
#### Steps for Cluster Autoscaling with Kubernetes Cluster Autoscaler <br/>

* Create a cluster with below command. It would create 2 m5 EC2 instances in region configured in your CLI (AWS configure).<br/>
  $ eksctl create cluster --name my-cluster --version 1.23 --managed --asg-access <br/>
* Please complete the prerequisites as mentioned in official AWS documentation shared above. <br/>
* Create an IAM policy (AmazonEKSClusterAutoscalerPolicy) and paste the JSON code from the text file "AmazonEKSClusterAutoscalerPolicy.txt" <br/>
* Create IAM role and service account with below command (replace the --attach-policy-arn with the ARN of the policy created above)<br/>
  $ eksctl create iamserviceaccount --cluster=my-cluster --namespace=kube-system --name=cluster-autoscaler --attach-policy-arn=arn:aws:iam::127538279091:policy/AmazonEKSClusterAutoscalerPolicy --override-existing-serviceaccounts --approve <br/>
* Check created service account and role. Check annotations and role should be there <br/>
  $ kubectl describe sa -n kube-system cluster-autoscaler <br/>
* [Deploy the Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html#Deploy%20the%20Cluster%20Autoscaler:~:text=for%20Linux%20Instances.-,Deploy%20the%20Cluster%20Autoscaler,-Complete%20the%20following)<br/>
* Annotate the cluster-autoscaler service account with the ARN of the IAM role that was created earlier as described in the AWS documentation. Check the service account  (kubectl describe sa) and if it is already present then skip this step <br/>
* To add the cluster-autoscaler.kubernetes.io/safe-to-evict annotation, execute below command (the patch command given in official documentation sometimes does not work)<br/>
  $ kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false" <br/>
* As mentioned in official documentation edit the Cluster Autoscaler deployment with options "--balance-similar-node-groups" and "--skip-nodes-with-system-pods=false" <br/>
* Set the Cluster Autoscaler image tag as from the official Github page as mentioned in the official AWS documentation <br/>
* View and verify the cluster autoscaler logs to ensure its working properly & monitoring the cluster load  <br/>
* Navigate to "Auto Scaling Group" in AWS EC2 console and update maximum capacity in autoscaling group to 6. (all values would be set to 2 - desired, minimum and maximum). Update the maximum value <br/>
* Apply the deployment "php-apache.yaml" from this repo with below command. It would create 1 replica <br/>
  $ kubectl apply -f php-apache.yaml <br/>
* We can see that 2 EC2 m5.large machines are in cluster. <br/>
* Edit the "php-apache.yaml" file and update the replicas to 20 and reapply executing below command <br/>
  $ kubectl apply -f php-apache.yaml <br/> 
* Now 20 pods are required with 500m CPU which is is equivalent to 5 m5.large EC2 instances. <br/>
* Check the activities tab in the autoscaling group and it can be seen that new instances are created. It checks the cluster load every 10 seconds <br/>
* In few mins it can be seen in EC2 console that 5/6 m5.large machines are created by cluster autoscaler. The same can be seen in terminal by executing below command <br/>
  $ kubectl get nodes <br/>
* Edit the "php-apache.yaml" file and update the replicas back to 1 and reapply executing below command <br/>
  $ kubectl apply -f php-apache.yaml <br/> 
* While scaling down the cluster autoscaler has a cool off period of 10 mins. So after ~10+ mins it can be seen that the cluster is scaled down to 2 m5.large EC2 instances. <br/>
