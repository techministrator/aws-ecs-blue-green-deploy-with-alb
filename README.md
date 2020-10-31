# LAB ECS NGINX EC2 Containers with ALB and Blue/Green CodeDeploy

### Step 0: IAM Roles and Policies

Role: Lambda Lifecycle Event Hook
- `AWSLambdaBasicExecutionRole` Policy
- `codedeploy:PutLifecycleEventHookExecutionStatus` Permision

Service-Linked Role: `AWSServiceRoleForECS`
- `AmazonECSServiceRolePolicy` Policy

Role: ecsInstanceRole - For EC2 ECS-Optimized Instances to register itself to ECS Cluster
- `AmazonEC2ContainerServiceforEC2Role` Policy

Role: ecsTaskExecutionRole - For ECS Service to 
- `AmazonECSTaskExecutionRolePolicy` Policy - For ECS Service to pull Docker and ECR Images and push Logs to CloudWatch Logs and Metrics to CloudWatch
- `AWSCodeDeployRoleForECS` Policy - Allow CodeDeploy and ECS to perform Blue/Green Deployment


### Step 1: Create an Application Load Balancer

Create ALB
```bash
$ aws elbv2 create-load-balancer --name test-codedeploy-alb --subnets subnet-00d774ad15e1b13c4 subnet-093d7628d7c4b87db --security-groups sg-03972bfee70c70f81 --region ap-northeast-2
```

Create Target Groups
```bash
$ aws elbv2 create-target-group --name test-codedeploy-alb-tg-1 --protocol HTTP --port 80 --target-type ip --vpc-id vpc-01942f3f1fb48a27c --region ap-northeast-2

$ aws elbv2 create-target-group --name test-codedeploy-alb-tg-2 --protocol HTTP --port 80 --target-type ip --vpc-id vpc-01942f3f1fb48a27c --region ap-northeast-2
```

Create Listeners
```bash
$ aws elbv2 create-listener --load-balancer-arn arn:aws:elasticloadbalancing:ap-northeast-2:807231263819:loadbalancer/app/test-codedeploy-alb/e733bec8371eba49 --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:ap-northeast-2:807231263819:targetgroup/test-codedeploy-alb-tg-1/ac251d587255d123 --region ap-northeast-2

$ aws elbv2 create-listener --load-balancer-arn arn:aws:elasticloadbalancing:ap-northeast-2:807231263819:loadbalancer/app/test-codedeploy-alb/e733bec8371eba49 --protocol HTTP --port 8080 --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:ap-northeast-2:807231263819:targetgroup/test-codedeploy-alb-tg-2/36ec7aff13b3eee6 --region ap-northeast-2
```

### Step 2: Create ECS Auto Scaling Group

Create ASG Launch Configuration
```bash
$ aws autoscaling create-launch-configuration --cli-input-json file://test-ecs-asg-launch-config.json --user-data file://test-ecs-asg-launch-config-user-data.txt --region ap-northeast-2
```

Create ASG
```bash
$ aws autoscaling create-auto-scaling-group --cli-input-json file://test-ecs-asg.json --region ap-northeast-2
```

### Step 3: Create ECS Cluster

Make sure the cluster name matched with the ASG Launch Configuration User Data config
```bash
$ aws ecs create-cluster --cluster-name test-ecs-cluster --region ap-northeast-2
```

### Step 4: Create ECS NGINX Task Definition

- Create ECS Task Definition and ECS Service based on the templates in this directory.
  - **Important** Make sure to create ECS Service by CLI using the template here. Using GUI will force you to create target groups. 

```bash
$ aws ecs register-task-definition --cli-input-json file://test-ecs-nginx-task-def.json --region ap-northeast-2
```

### Step 5: Create ECS NGINX Service

```bash
$ aws ecs create-service --cli-input-json file://test-ecs-nginx-service.json --region ap-northeast-2
```

### Step 6: Create a Lifecycle Hook Lambda Function

```bash
$ zip AfterAllowTestTraffic.zip AfterAllowTestTraffic.js

$ aws lambda create-function --function-name test-lambda-hook-after-allow-test-traffic --zip-file fileb://AfterAllowTestTraffic.zip --handler AfterAllowTestTraffic.handler --runtime nodejs10.x --role arn:aws:iam::807231263819:role/ecsLifecycleEventHookForLambda --region ap-northeast-2
```

### Step 7: Create AppSpec File

Upload `appspec.yml` file
```bash
$ aws s3 cp appspec.yml s3://test-ecs-blue-green-deployment-bucket/
```

### Step 8: Create CodeDeploy Application and Deployment Group

Create CodeDeploy Application
```bash
$ aws deploy create-application --application-name test-ecs-codedeploy-app --compute-platform ECS --region ap-northeast-2
```

Create CodeDeploy Deployment Group
```bash
$ aws deploy create-deployment-group --cli-input-json file://test-ecs-codedeploy-deployment-group.json --region ap-northeast-2
```

### Step 9: Change Task Definition and Create CodeDeploy Deployment

**Important** Make sure to create change to the earlier created Task Definition like add more text to the `index.html` in Container Definition section. Then grab the newest Task Definition Version Number and add to the `appspec.yml` file:
```yml
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: 'arn:aws:ecs:ap-northeast-2:807231263819:task-definition/test-ecs-nginx-task-def:2'  # Add New Version here
        LoadBalancerInfo:
          ContainerName: 'nginx'
          ContainerPort: 80
```

Upload the edited `appspec.yml` file to S3 Bucket
```bash
$ aws s3 cp appspec.yml s3://test-ecs-blue-green-deployment-bucket/
```

Perform CodeDeploy Deployment
```bash
$ aws deploy create-deployment --cli-input-json file://test-ecs-codedeploy-deployment.json --region ap-northeast-2
```


### (Optional) Other Deployment Group Options

In Step 9, the deployment configuration option is `CodeDeployDefault.ECSAllAtOnce` and the **ReRoute to Production Traffic** happens immediately after the Lambda Function Lifecycle `AfterAllowTestTraffic` Hook succeeds (always for testing purpose).

If you want to have traffic to be rerouted manually by you. Which means you have to go to CodeDeploy Deployment to press the **Reroute** button by yourself. Then create deployment group using `other-deployment-group-options/test-ecs-codedeploy-deployment-group-manual-reroute.json` file. 

As contrast, the `other-deployment-group-options/test-ecs-codedeploy-deployment-group-auto-reroute.json` file is the same as the `test-ecs-codedeploy-deployment-group.json` in the root dir. But I put it there for comparison purpose. 