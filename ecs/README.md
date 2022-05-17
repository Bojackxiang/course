>网址：https://www.bilibili.com/video/BV1nR4y1N72u?p=17



### 创建简单的vpc设置

这边直接在 cli 中运行会报错，所以这边我直接在 console 中运行，

选择 cloudformation 然后上传就可以了

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC and subnets as base for an ECS cluster
Parameters:
  EnvironmentName:
    Type: String
    Default: ecs-course

Mappings:
  SubnetConfig:
    VPC:
      CIDR: '172.16.0.0/16'
    PublicOne:
      CIDR: '172.16.0.0/24'
    PublicTwo:
      CIDR: '172.16.1.0/24'

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      EnableDnsSupport: true
      EnableDnsHostnames: true
      CidrBlock: !FindInMap ['SubnetConfig', 'VPC', 'CIDR']

  PublicSubnetOne:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone:
         Fn::Select:
         - 0
         - Fn::GetAZs: {Ref: 'AWS::Region'}
      VpcId: !Ref 'VPC'
      CidrBlock: !FindInMap ['SubnetConfig', 'PublicOne', 'CIDR']
      MapPublicIpOnLaunch: true
  PublicSubnetTwo:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone:
         Fn::Select:
         - 1
         - Fn::GetAZs: {Ref: 'AWS::Region'}
      VpcId: !Ref 'VPC'
      CidrBlock: !FindInMap ['SubnetConfig', 'PublicTwo', 'CIDR']
      MapPublicIpOnLaunch: true

  InternetGateway:
    Type: AWS::EC2::InternetGateway
  GatewayAttachement:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref 'VPC'
      InternetGatewayId: !Ref 'InternetGateway'
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref 'VPC'
  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: GatewayAttachement
    Properties:
      RouteTableId: !Ref 'PublicRouteTable'
      DestinationCidrBlock: '0.0.0.0/0'
      GatewayId: !Ref 'InternetGateway'
  PublicSubnetOneRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetOne
      RouteTableId: !Ref PublicRouteTable
  PublicSubnetTwoRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetTwo
      RouteTableId: !Ref PublicRouteTable

Outputs:
  VpcId:
    Description: The ID of the VPC that this stack is deployed in
    Value: !Ref 'VPC'
    Export:
      Name: !Sub ${EnvironmentName}:VpcId
  PublicSubnetOne:
    Description: Public subnet one
    Value: !Ref 'PublicSubnetOne'
    Export:
      Name: !Sub ${EnvironmentName}:PublicSubnetOne
  PublicSubnetTwo:
    Description: Public subnet two
    Value: !Ref 'PublicSubnetTwo'
    Export:
      Name: !Sub ${EnvironmentName}:PublicSubnetTwo


```

在 aws console 的 cloud formation 中，选择 outputs

通过这个所传健的所有信息都在 outputs 里面能够找到



### 创建两种不同的 cluster 

有两种 cluster， 一个是基于 ec2 的，还有一种是直接基于 farget的，这种事不会生成 ec2 的

fargate 的不会创建 ec2 instance 



### 通过 cloudformation 来创建 cluster 

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: ECS cluster launchtype EC2.
Parameters:
  EnvironmentName:
    Type: String
    Default: ecs-course
    Description: "A name that will be used for namespacing all cluster resources."
  InstanceType:
    Description: EC2 instance type
    Type: String
    Default: t2.small
    Description: Class of EC2 instance used to host containers. Choose t2 for testing, m5 for general purpose, c5 for CPU intensive services, and r5 for memory intensive services
    AllowedValues: [ t2.micro, t2.small, t2.medium, t2.large, t2.xlarge, t2.2xlarge,
     m5.large, m5.xlarge, m5.2large, m5.4xlarge, m5.12xlarge, m5.24large,
     c5.large, c5.xlarge, c5.2xlarge, c5.4xlarge, c5.9xlarge, c5.18xlarge,
     r5.large, r5.xlarge, r5.2xlarge, r5.4xlarge, r5.12xlarge, r5.24xlarge ]
    ConstraintDescription: Please choose a valid instance type.
  DesiredCapacity:
    Type: Number
    Default: '1'
    Description: Number of EC2 instances to launch in your ECS cluster.
  MaxSize:
    Type: Number
    Default: '6'
    Description: Maximum number of EC2 instances that can be launched in your ECS cluster.
  ECSAMI:
    Description: AMI ID
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ecs/optimized-ami/amazon-linux-2/recommended/image_id
    Description: The Amazon Machine Image ID used for the cluster, leave it as the default value to get the latest AMI

Resources:
  
  # ECS Resources
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Sub ${EnvironmentName}-ec2

  # A security group for the EC2 hosts that will run the containers.
  # Rules are added based on what ingress you choose to add to the cluster.
  ContainerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Access to the ECS hosts that run containers
      VpcId: 
        Fn::ImportValue: !Sub ${EnvironmentName}:VpcId

  # Autoscaling group. This launches the actual EC2 instances that will register
  # themselves as members of the cluster, and run the docker containers.
  ECSAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier:
        - Fn::ImportValue: !Sub ${EnvironmentName}:PublicSubnetOne
        - Fn::ImportValue: !Sub ${EnvironmentName}:PublicSubnetTwo
      LaunchConfigurationName: !Ref 'ContainerInstances'
      MinSize: '1'
      MaxSize: !Ref 'MaxSize'
      DesiredCapacity: !Ref 'DesiredCapacity'
    CreationPolicy:
      ResourceSignal:
        Timeout: PT15M
    UpdatePolicy:
      AutoScalingReplacingUpdate:
        WillReplace: 'true'
  ContainerInstances:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      ImageId: !Ref 'ECSAMI'
      SecurityGroups: [!Ref 'ContainerSecurityGroup']
      InstanceType: !Ref 'InstanceType'
      IamInstanceProfile: !Ref 'EC2InstanceProfile'
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -xe
          echo ECS_CLUSTER=${ECSCluster} >> /etc/ecs/ecs.config
          yum install -y aws-cfn-bootstrap
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource ECSAutoScalingGroup --region ${AWS::Region}
  EC2InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Path: /
      Roles: [ 'ecsInstanceRole' ]

Outputs:
  ClusterName:
    Description: The name of the ECS cluster
    Value: !Ref 'ECSCluster'
    Export:
      Name: !Sub ${EnvironmentName}:ClusterName

```

#### Cloudformation: 创建 基于 fargate 的 cluster 

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: ECS cluster launchtype Fargate.
Parameters:
  EnvironmentName:
    Type: String
    Default: ecs-course
    Description: "A name that will be used for namespacing our cluster resources."
Resources:    
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Sub ${EnvironmentName}-fargate
  ContainerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Access to the ECS hosts that run containers
      VpcId: 
        Fn::ImportValue: !Sub ${EnvironmentName}:VpcId
Outputs:
  ClusterName:
    Description: The name of the ECS cluster
    Value: !Ref 'ECSCluster'

```



#### 创建 ECS task definition 

sample image: gkoenig/simplehttp:latest 

**后面的 这个 latest 非常重要**

- create a task definition (Fargate)

  - run task definition in the cluster 

  - run service 

  - >create TD 的时候
    >
    >environment essential: 不用check, essential 的意思是：如果一个 container 挂了，其他的 task 都会 挂

- create a task defition (EC2)

  - >这里选择了一个 bridge 而不是 awsvpc，这样运行的 container 就会有一个 随机的 port，
    >
    >所以这边打开 external link 也不会有任何的反应，因为 security group 不会让他仅需，所以需要修改 security group 

⚠️： 如果有多个 container 运行的话， environment essential: 一定要check， essential 的意思是： 如果一个 container 挂了，其他的 task 都会 

⚠️：awsvpc required for fargate, optional for ec2

#### 运行 task definition 

- running in task directly 

- running in service 

  > revision： 选择 latest 
  >
  > Launch type： 选择 EC2
  >
  > Deploy type: 选择 rolloing update
  >
  > ⚠️在运行 service 的时候，因为我们的 cluster 就是 一个 ec2，所以我们在运行 service 的时候也要 选择 ec2
  >
  > ⚠️ 因为我们在创建 task definition 的时候，我们选择了 bridge，所以无论 service 和 task 如何运行，那么都会是 random assign 的port
  >
  > **但是 dynamic port 的问题可以在 load balance 中解决**



#### ESC set up (real world with public & private network)

VPC 创建流程

>1. 创建 VPC
>
>2. 创建 subnet 
>
>3. subnet 会自动创建 gateway
>4. gateway 会直接绑定在 VPC 上面



对 set up infrastructure .yaml 进行了解析，

> 放我们删除了 cloud formation，由这个 cloudformation 创建的所有资源都会被直接删除



#### Load Balancing

我们使用了 load balance 之后，我们可以让 subnet 只允许 load balance 的请求进来，但是不需要在把 很多的port 对外公开  ·

我们还能 给 load balance 一个 static 的weblink，这样别人就能直接访问我们的资源

分析 alb external yaml 文件

>#### !Sub
>
>****
>
>```
>!Sub ${EnvironmentName}:ExternalUrl
>```
>
>**!Sub 是连接 两个 string 的意思**
>
>#### Fn::ImportValue
>
>```
>Fn::ImportValue: !Sub ${EnvironmentName}:VpcId # 这个 sub 应该就是把这两个 value 连起来
>```
>
>**Fn::ImportValue: 的意思就是别的stack 已经创建了的value （在 resource 中能够找到的）拿到这个stack 里面来使用**



#### Use laod balancer with the Farget + EC2

⚠️：还是 fargate 的 cluster 智能运行 farget 的 service， ec2 的 service 智能运行在ec2 的cluster 里面

>**Fargate case** 
>
>VPC 选对 才能有正确的load balancer 出来
>
>设置 security group 
>
>container to load balancer 之间的设置
>
>production port 钥匙 80:http 
>
>target group name 要和 load balancer 的target group name 一样



>**EC2 case**
>
>在 load balancer 的时候选择 选择 application **load balancer** 
>
>选择 role 和 load balancer name
>
>Click add load balancer, 在 target group name 种选择刚刚穿件的 load balancer 



#### ECR

LAB ： 将一个 dockerhub 里面的 image 拿出来然后放到 ECR 里面

step 1 直接创建一个 repo 

>aws ecr create-repository --repository-name sample-http --region ap-southeast-1 --image-scanning-configuration scanOnPush=true

Step 2 将已经存在的目标 image 下载下来

> doxker pull yeasy/simple-web

Step 3 直接在 container repo 里面找 push 的command

> aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin 786766101582.dkr.ecr.ap-southeast-2.amazonaws.com
>
> docker tag <local image name> 786766101582.dkr.ecr.ap-southeast-2.amazonaws.com/sambuildpracticee49b74ff/helloworldfunction19d43fc4repo:latest
>
> docker push 786766101582.dkr.ecr.ap-southeast-2.amazonaws.com/sambuildpracticee49b74ff/helloworldfunction19d43fc4repo:latest



docker tag yeasy/simple-web 786766101582.dkr.ecr.ap-southeast-2.amazonaws.com/sambuildpracticee49b74ff/helloworldfunction19d43fc4repo:latest