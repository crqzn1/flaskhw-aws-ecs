# Flask & AWS ECR ECS


## Prepare Docker Image

### Prepare Flask app
```bash
$ flask run
	 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
$ python app.py
	 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```
>Check localhost:5000

### Build docker image locally
```bash
$ docker build -t flaskhw:0.0.2 .
$ docker run -d --rm --name flaskhw -p 5000:5000 flaskhw:0.0.2
$ docker ps
	CONTAINER ID   IMAGE           COMMAND           CREATED              STATUS              PORTS      NAMES
	a99280781017   flaskhw:0.0.2   "python app.py"   About a minute ago   Up About a minute   5000/tcp   flaskhw
$ docker inspect a99280781017 | grep IPAddress
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.2",
                    "IPAddress": "172.17.0.2",
$ docker stop a99280781017
```


## Push to AWS ECR

### tag local image
```bash
$ docker images | grep flaskhw
	flaskhw             0.0.2             9767d6cea2a4   4 minutes ago   182MB
$ docker tag 9767d6cea2a4 111122223333.dkr.ecr.ap-southeast-2.amazonaws.com/flaskhw:0.0.2
```
### create a repository in ECR - refer doc

### login & push
```bash
$ aws ecr get-login-password --region ap-southeast-2 | docker login --username AWS --password-stdin 111122223333.dkr.ecr.ap-southeast-2.amazonaws.com
	login successfully
$ docker push 111122223333.dkr.ecr.ap-southeast-2.amazonaws.com/flaskhw:0.0.2
```


## Config ECS

### install ecs-cli - refer doc

### config ecs-cli
```bash
$ ecs-cli configure --cluster my-fargate-cluster --default-launch-type FARGATE --config-name my-fargate-config --region ap-southeast-2
$ ecs-cli configure default --config-name my-fargate-config
```
```bash
$ ecs-cli configure profile --profile-name my-fargate-prof --access-key xxx --secret-key xxx
$ ecs-cli configure profile default --profile-name my-fargate-prof
```

### Config execution role
```bash
$ aws iam --region ap-southeast-2 create-role --role-name my-ecsTaskExecutionRole --assume-role-policy-document file://task-execution-assume-role.json
$ aws iam --region ap-southeast-2 attach-role-policy --role-name my-ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```


## Fargate

### start cluster
```bash
$ ecs-cli up
	VPC created: vpc-1111222233334444x
	Subnet created: subnet-1111222233334444a
	Subnet created: subnet-1111222233334444b
```
>update subnets & security_groups in ecs-params.yml

### update security group to allow tcp traffice on port 5000
```bash
$ aws ec2 describe-security-groups --filters Name=vpc-id,Values=vpc-1111222233334444x
	sg-1111222233334444c
$ aws ec2 authorize-security-group-ingress --group-id sg-1111222233334444c --protocol tcp --port 5000 --cidr 0.0.0.0/0
```

### start service
```bash
$ ecs-cli compose --project-name my-fargate-proj2 service up
	INFO[0011] (service my-fargate-proj2) has started 1 tasks: (task 7a31635b615942e6be5d029ce19b8a38).  timestamp="2021-11-08 14:17:42 +0000 UTC"
```

### check service
```bash
$ ecs-cli compose --project-name my-fargate-proj3 service ps
	Name                                                      State    Ports                        TaskDefinition       Health
	my-fargate-cluster/7a31635b615942e6be5d029ce19b8a38/web  RUNNING  3.24.214.111:5000->5000/tcp  my-fargate-proj2:1  UNKNOWN
```
>check 3.24.214.111:5000

### Scale Service
```bash
$ ecs-cli compose --project-name my-fargate-proj2 service scale 2
    INFO[0000] Updated ECS service successfully              desiredCount=2 force-deployment=false service=my-fargate-proj3
$ ecs-cli compose --project-name my-fargate-proj3 service ps
    Name                                                      State    Ports                        TaskDefinition      Health
    my-fargate-cluster/2642b3f954164097bfe5d32e7ebf0820/web  RUNNING  3.26.36.133:5000->5000/tcp   my-fargate-proj3:2  UNKNOWN
    my-fargate-cluster/e5308d85acef457895417bb6529ca728/web  RUNNING  3.24.135.156:5000->5000/tcp  my-fargate-proj3:2  UNKNOWN
```

### Use CloudWatch
> update docker-compose.yml add
```yml
    logging:
      driver: awslogs
      options: 
        awslogs-group: flask-ecs-log
        awslogs-region: ap-southeast-2
        awslogs-stream-prefix: web
```
```bash
$ ecs-cli compose --project-name my-fargate-proj4 service up --create-log-groups
```

### End Service
```bash
$ ecs-cli compose --project-name my-fargate-proj2 service down
	INFO[0005] (service my-fargate-proj2) has stopped 1 running tasks: (task 7a31635b615942e6be5d029ce19b8a38).  timestamp="2021-11-08 14:20:04 +0000 UTC"
```

### End Cluster
```bash
$ ecs-cli down --force
	INFO[0091] Deleted cluster                               cluster=my-fargate-cluster
```