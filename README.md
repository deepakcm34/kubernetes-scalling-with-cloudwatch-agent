# kubernetes-scalling-with-cloudwatch-agent (Both hap [pod] and node scalling) (Memory base)
  
  
 #Step 1:
 ----------------
 
1: Edit cluster and add necessary connfiguration/permissions :
 
     kops edit cluster --name clustername --state s3-bucket-state-store
     
 [I] Add below configuration to your cluster configuration (For hpa) : 

    kubelet:
        anonymousAuth: false
        authorizationMode: Webhook
        authenticationTokenWebhook: true
        
  [II] Add the following policies to the nodes (For node scalling):
   
    kind: Cluster 
    ... 
    spec: 
      additionalPolicies:
        node: |
          [
            {
              "Effect": "Allow",
              "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "cloudwatch:PutMetricData",
                "ec2:DescribeVolumes",
                "ec2:DescribeTags",
                "logs:PutLogEvents",
                "logs:DescribeLogStreams",
                "logs:DescribeLogGroups",
                "logs:CreateLogStream",
                "logs:CreateLogGroup"
              ],
              "Resource": ["*"]
            }
          ]


2: Edit node instances group (For node scalling):

    kops edit instancegroups nodes  --name clustername --state s3-bucket-state-store
    
  [I] Add the following cloud labels, and modify the for the name of your cluster; Also set the maxSize and minSize, for example, my worker's nodes are going to scale-down up to 2 instances and scale-up up to 10 instances.
  
    kind: InstanceGroup 
    ... 
    spec: 
      cloudLabels: 
        service: k8s_node 
        k8s.io/cluster-autoscaler/enabled: "" 
        k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>: "" 
      maxSize: 10 
      minSize: 2 
    ...
  
3: After changing the above setup, update the cluster and then run the rolling-update :

    kops update cluster --name clustername --state s3-bucket-state-store --yes
    
    kops rolling-update cluster --name clustername --state s3-bucket-state-store --yes
#------------------------------------------------------------------------------------------------------------------

    
    
# Step 2
--------------

After updating the cluster, deploy the metrics-server,hpa,cloudwatch agent 


# Deploy metrics-server:

Kubernetes 1.8+ <= 1.15

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/kops/master/addons/metrics-server/v1.8.x.yaml
    
Kubernetes 1.16+

    kubectl apply -f https://raw.githubusercontent.com/kubernetes/kops/master/addons/metrics-server/v1.16.x.yaml
    
Also i am attaching the v1.16.x.yaml as the metrics-server.yaml in this repo.


It will take some time to detect the metrics.

Test the metrics:
 
    kubectl top node
    
# Deploy a test deployment for hpa testing

    kubectl apply -f php.apache.yaml
    
# Deploy hpa (Horizontal Pod Autoscaler)

The hpa is based on the application "php-apache" which deployed:

    kubectl apply -f hpa.yaml
    
Once done, you can check the hpa:

     kubectl get hpa


    NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
    php-apache   Deployment/php-apache   0%/50%    1         10        1          37m
    
You can test the working of hpa by using the apache-jmeter in your local end, you can use the loadbalancer URL of the php-apache service.

Once the application and hpa is deployed, we can go for the cloudwatch agent deployemts



# Follow the below steps to deploy the cloudwatch agent:



1: Create namespace for cloudwatch agent:

    kubectl apply -f namespace.yaml
    
2: Create a Service Account and ClusterRole for cloudwatch agent in the Cluster:

    kubectl apply -f serviceaccount-clusterrole.yaml
    
3: Create a ConfigMap for the CloudWatch Agent :

    kubectl apply -f cwa-configmap.yaml
    
4: Deploy the CloudWatch Agent as a DaemonSet :

    kubectl apply -f cwa-daemonset.yaml
    
#------------------------------------------------------------------------------------------------------------------

Once everything is done, check if the Metrics is showing in the AWS cloudwatch (It might get some delay for detecting the metrics from the agent for the first time) :

    Login to AWS >> Cloudwatch >> Metrics >> All metrics >> CWAgent >> AutoScalingGroupName 
    
If everything is working, create the alarm using the memory metrics and use the alarm in the Autoscalling group of the cluster in-order to create the scalling policy.


# How to test the scalling:

Install the apache-jmeter in your local end and access the same.

Use the application loadbalancer service URL and send high threads to the applications. The pod will start to scale as per the hpa configuration and when the node does not have any resource to create the pods, the memory of the node will increase and the cloud watch alarm will trigger. 

Once the cloudwatch alarm will triggered, the autoscale group will scale the nodes as per the policy created.
    
