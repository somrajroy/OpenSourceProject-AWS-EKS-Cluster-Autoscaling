## AWS EKS Cluster Autoscaling
This repo demonstrates the cluster autoscaling feature of K8S. <br/><br/>
* To adjust to changing application demands clusters often need a way to automatically scale. In AWS EKS clusters automatic scaling is done primarily in two ways -  Cluster Autoscaler and Horizental Pod Autoscaler<br/>
  * The cluster autoscaling (CA) watches for pods that can't be scheduled on nodes because of resource constraints.  If a node doesn't have sufficient compute resources to run a requested pod, that pod can't progress through the scheduling process. The pod can't start unless additional compute resources are available within the node pool. The cluster then automatically increases the number of nodes when it notices the resource constraints.The below diagrams makes it clear about cluster autoscaler functionality.<br/>
  * Kubernetes uses the horizontal pod autoscaler (HPA) to monitor the resource demand and automatically scale the number of replicas. By default, the horizontal pod autoscaler checks the Metrics API. The horizontal pod autoscaler uses the Metrics Server in a Kubernetes cluster to monitor the resource demand of pods. If an application needs more resources, the number of pods is automatically increased to meet the demand.<br/>
  * The below diagrams makes it clear about HPA & CA functionality. <br/>
    ![image](https://user-images.githubusercontent.com/92582005/202852434-dbf37c5a-e2a7-4783-b379-dbb693d729bd.png) <br/><br/>
    ![image](https://user-images.githubusercontent.com/92582005/204074338-7fd0e500-7a19-4216-b084-087362471888.png) <br/>
   * The cluster and horizontal pod autoscalers can work together, and are often both deployed in a cluster. When combined, the horizontal pod autoscaler is focused on running the number of pods required to meet application demand. The cluster autoscaler is focused on running the number of nodes required to support the scheduled pods. <br/>
* [This official AWS Documentation mentions the steps to implement EKS Cluster Autoscaling. In this article we will use Kubernetes Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html) <br/>
* Some steps does not work properly from the above documentation which will be highlighted here <br/>
#### Steps for Cluster Autoscaling with Kubernetes Cluster Autoscaler <br/>
* Region us-east-1 gives some issues with availability zones. It maybe convinient to choose some other AWS region. Here us-west-2 is chosen (aws configure) <br/>
* Create a cluster with below command. It would create 2 m5.large EC2 instances in region configured in your CLI (AWS configure).<br/>
  $ eksctl create cluster --name my-cluster --version 1.23 --managed --asg-access <br/>
* Create or update a kubeconfig file for the cluster. Replace region-code with the AWS Region that the cluster is in and replace my-cluster with the name of the cluster (then kubectl commands can be run on the created cluster). <br/>
  $ aws eks update-kubeconfig --region region-code --name my-cluster <br/>
* Please complete the prerequisites as mentioned in official AWS documentation shared above. <br/>
* (Optional)Create an IAM policy (AmazonEKSClusterAutoscalerPolicy) and paste the JSON code from the text file "AmazonEKSClusterAutoscalerPolicy.txt" <br/>
* Create IAM role and service account with below command (replace the --attach-policy-arn with the ARN of the policy created by EKSTL or above created role)<br/>
* If you created your node groups using the --asg-access option, then replace the name of the IAM policy with that eksctl created for you. The policy name is similar to eksctl-my-cluster-nodegroup-ng-xxxxxxxx-PolicyAutoScaling. (Optionally) If you created policy AmazonEKSClusterAutoscalerPolicy then use that. <br/>
  * Replace the account ID with the one that is being used. <br/>
  * Replace the policy name "eksctl-my-cluster-nodegroup-ng-ae84bd0e-PolicyAutoScaling" with the one eksctl created for you <br/>
  $ eksctl create iamserviceaccount --cluster=my-cluster --namespace=kube-system --name=cluster-autoscaler --attach-policy-arn=arn:aws:iam::<<-your account id->>:policy/eksctl-my-cluster-nodegroup-ng-ae84bd0e-PolicyAutoScaling --override-existing-serviceaccounts --approve <br/>
* Check created service account and role. Check annotations and role should be there <br/>
  $ kubectl describe sa -n kube-system cluster-autoscaler <br/>
* [Deploy the Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html#Deploy%20the%20Cluster%20Autoscaler:~:text=for%20Linux%20Instances.-,Deploy%20the%20Cluster%20Autoscaler,-Complete%20the%20following)<br/>
* Annotate the cluster-autoscaler service account with the ARN of the IAM role that was created earlier as described in the AWS documentation. Check the service account  (kubectl describe sa) and if it is already present then skip this step <br/>
* To add the cluster-autoscaler.kubernetes.io/safe-to-evict annotation, execute below command (the patch command given in official documentation sometimes does not work)<br/>
  $ kubectl -n kube-system annotate deployment.apps/cluster-autoscaler cluster-autoscaler.kubernetes.io/safe-to-evict="false" <br/>
* As mentioned in official documentation edit the Cluster Autoscaler deployment with options "--balance-similar-node-groups" and "--skip-nodes-with-system-pods=false" <br/>
* (Optional) Verify the deployment.<br/>
  $ kubectl describe deployment -n kube-system cluster-autoscaler <br/>
* Set the Cluster Autoscaler image tag as from the official Github page as mentioned in the official AWS documentation <br/>
  $ kubectl set image deployment cluster-autoscaler -n kube-system cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:v1.23.0 <br/>
* View and verify the cluster autoscaler logs to ensure its working properly & monitoring the cluster load  <br/>
* Navigate to "Auto Scaling Group" in AWS EC2 console and update maximum capacity in autoscaling group to 6. (all values would be set to 2 - desired, minimum and maximum). Update the maximum value <br/>
* Apply the deployment "php-apache.yaml" from this repo with below command. It would create 1 replica <br/>
  $ kubectl apply -f php-apache.yaml <br/>
* We can see that 2 EC2 m5.large machines are in cluster. <br/>
* Edit the "php-apache.yaml" file and update the replicas to 20 and reapply executing below command <br/>
  $ kubectl apply -f php-apache.yaml <br/> 
* Now 20 pods are required with 500m CPU which is is equivalent to 5 m5.large EC2 instances. Many pods will be in pending status (not running). This would trigger CA<br/>
* Check the activities tab in the autoscaling group and it can be seen that new instances are created. It checks the cluster load every 10 seconds <br/>
* In few mins it can be seen in EC2 console that 5/6 m5.large machines are created by cluster autoscaler. The same can be seen in terminal by executing below command <br/>
  $ kubectl get nodes <br/>
* Edit the "php-apache.yaml" file and update the replicas back to 1 and reapply executing below command. It would take 10 mins for the scale down operation <br/>
  $ kubectl apply -f php-apache.yaml <br/> 
* Clean up AWS enviornment <br/>
  $ eksctl delete cluster --name my-cluster <br/>
* By default, cluster autoscaler will wait 10 minutes between scale down operations, which can be adjusted using the --scale-down-delay-after-add, --scale-down-delay-after-delete, and --scale-down-delay-after-failure flag. E.g. --scale-down-delay-after-add=5m to decrease the scale down delay to 5 minutes after a node has been added. <br/>

#### Further references <br/>
* [Cluster Autoscaler on AWS](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md)<br/>
* [Kubernetes Cluster Autoscaler - EKS Best Practices Guides](https://aws.github.io/aws-eks-best-practices/cluster-autoscaling/)<br/>
