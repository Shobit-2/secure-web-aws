Secure a Web Service on AWS

One small nginx website on Amazon ECS (Fargate), reached through an internet-facing Application Load Balancer. A direct call to a container fails. 

Everything is described in Terraform (`main.tf`).

Region: `ap-south-1` (Mumbai)
Tooling: Terraform >= 1.5, AWS provider ~> 5.0, AWS CLI v2
Site URL while deployed: `http://secure-web-alb-1492229862.ap-south-1.elb.amazonaws.com/` (the stack has been destroyed, see Teardown result)

1. Architecture

                         Internet
                            |  TCP 80
                            v
        +---------------------------------------------+
        |  Application Load Balancer (internet-facing) |
        |  ALB security group: in TCP 80 from 0.0.0.0/0|
        |  Listener :80 -> Target group (type ip)      |
        |  Health check: GET /                         |
        +---------------------------------------------+
              |                              |
   public subnet 10.20.1.0/24       public subnet 10.20.2.0/24
        ap-south-1a                      ap-south-1b
              |                              |
        +-------------+              +-------------+
        | Fargate task|              | Fargate task|
        | nginx :80   |              | nginx :80   |
        | Task SG: in |              | Task SG: in |
        | 80 from ALB |              | 80 from ALB |
        | SG only     |              | SG only     |
        +-------------+              +-------------+
              |  outbound via Internet Gateway (image pull)
              v
   public.ecr.aws/docker/library/nginx:stable-alpine
   

Component                     	Detail

VPC	                           `10.20.0.0/16`
Public subnets	               `10.20.1.0/24` (ap-south-1a) and `10.20.2.0/24` (ap-south-1b)
Internet gateway	             Attached to the VPC; route table sends `0.0.0.0/0` to it; associated with both subnets
ALB security group	           Inbound TCP 80 from `0.0.0.0/0`; attached to the load balancer
Task security group	           Inbound TCP 80 from the ALB security group only; attached to the tasks
ECS cluster	                   `secure-web-cluster`
Task definition	               Fargate, 0.25 vCPU, 0.5 GB, `awslogs` driver to CloudWatch Logs `/ecs/secure-web`
Execution role	              `secure-web-task-exec-role` with `AmazonECSTaskExecutionRolePolicy` (log writes and image pull)
ECS service	                   Desired count 2, Fargate, task security group, public IPv4 on each task, both public subnets
Load balancer	                 Internet-facing ALB in both public subnets
Target group	                 Target type `ip`, health check on `/`
Listener	                     Port 80, forwards to the target group


2. Deploy commands

aws configure                  # region ap-south-1
terraform init
terraform fmt
terraform validate
terraform plan -out tfplan
terraform apply tfplan         # waits until both targets are healthy
terraform output

The service sets `wait_for_steady_state = true`, so `apply` returns only after both tasks are running and healthy in the target group. A USD 5 budget alert was created before applying.

3. Test evidence

Evidence 1: `curl -sS -D - -o /dev/null http://<alb-dns>/` returns 200

$ALB = terraform output -raw alb_dns_name
curl.exe -sS -D - -o NUL "http://$ALB/"

<img width="1150" height="361" alt="image" src="https://github.com/user-attachments/assets/a0a40409-479f-4ee3-8cb3-4e43d5ac21ea" />

Evidence 2: Task security group rule, inbound TCP 80 allowed from the ALB security group

aws ec2 describe-security-groups --region ap-south-1 --group-ids (terraform output -raw task_sg_id) --query "SecurityGroups[0].IpPermissions"

<img width="1234" height="511" alt="image" src="https://github.com/user-attachments/assets/dd4c06af-d78b-4510-87b2-ecb454266339" />

The source is the ALB security group (`sg-03df34e33eb425bbd`) in `UserIdGroupPairs`, and `IpRanges` is empty. Task security group: `sg-0f7b0761d89e9010b`.

Evidence 3: `curl http://<task-public-ip>/` fails with a timeout or connection refused

curl.exe -sS -m 10 "http://<task-public-ip>/"

<img width="567" height="29" alt="image" src="https://github.com/user-attachments/assets/827ca4a3-6abf-431d-9e5f-d3b8550fffd4" />


Extra evidence: two healthy targets, in two subnets

<img width="1210" height="460" alt="image" src="https://github.com/user-attachments/assets/562f0fe0-5948-4830-a47e-749ae72c2b17" />


<img width="1600" height="739" alt="image" src="https://github.com/user-attachments/assets/a2418454-963b-4bd3-9b5f-91cea8ccc8c4" />



Evidence 4: Screenshots from the AWS console and browser

What it shows	Screenshot

VPC `10.20.0.0/16`	
<img width="1600" height="739" alt="image" src="https://github.com/user-attachments/assets/9fe8788f-03cc-478b-814d-30fcb3d9aa90" />


Two public subnets in two Availability Zones	


<img width="1600" height="776" alt="image" src="https://github.com/user-attachments/assets/3bbb9302-d480-4bfa-bd22-3dc24569aaa5" />


Route table: `0.0.0.0/0` to the internet gateway, both subnets associated	


<img width="1600" height="749" alt="image" src="https://github.com/user-attachments/assets/d8003ea7-d3af-4095-932a-d8e364f3c9bb" />


Task security group: TCP 80 from the ALB security group


<img width="1234" height="511" alt="image" src="https://github.com/user-attachments/assets/397e457d-ed34-481b-a08c-6ca40faf48f6" />


Application Load Balancer, internet-facing, two AZs	
<img width="1600" height="783" alt="image" src="https://github.com/user-attachments/assets/731f7a80-e135-49f3-8e55-293620e378f6" />



Target group: two healthy targets	


<img width="1600" height="739" alt="image" src="https://github.com/user-attachments/assets/bac26f81-dca4-4d17-a593-ae6ddf6757ea" />


ECS service: desired 2, running 2	


<img width="1600" height="743" alt="image" src="https://github.com/user-attachments/assets/d2255e57-06e3-478f-b6e4-0ae9649f6677" />


nginx welcome page through the ALB DNS name	


<img width="1600" height="593" alt="image" src="https://github.com/user-attachments/assets/04a1be4a-0f21-4340-b1fc-c14e881731ad" />


CloudWatch log stream from the containers	


<img width="1600" height="780" alt="image" src="https://github.com/user-attachments/assets/96da1529-f4d9-4acf-b928-67e954ecf6cf" />



4. Design note

One successful request, browser to nginx. The browser resolves the ALB DNS name and opens TCP 80 to an ALB node in a public subnet. The ALB security group (`secure-web-alb-sg`) allows inbound 80 from anywhere. The listener on port 80 forwards the request to the target group (`secure-web-tg`), which picks a healthy task IP. The ALB connects to that Fargate task on port 80, and the task security group (`secure-web-task-sg`) accepts it because the source is the ALB security group. The nginx container in the task answers, and the response returns through the ALB.

Why the task's public IP does not open the site. The public IP gives the task a route out, but inbound traffic is still filtered by the task security group. Its only inbound rule is TCP 80 from the ALB security group, and a user's computer is not a member of that group. Security groups deny inbound traffic by default, so the packet is dropped and the client times out (Evidence 3).

How the task pulls the image. Here the task has a public IPv4 address in a public subnet, so it reaches `public.ecr.aws` through the internet gateway, and the task security group allows all outbound traffic. With only a private IP there is no public address for the internet gateway to use, so the task needs either a NAT gateway in a public subnet (private subnets route `0.0.0.0/0` to it), or VPC endpoints: `ecr.api` and `ecr.dkr` interface endpoints, an S3 gateway endpoint for image layers, and a `logs` endpoint. Pulling from public ECR usually still needs NAT unless the image is copied to a private ECR repository first.

If one Availability Zone is down. The ALB keeps sending traffic to the task in the other AZ, so the site stays up at half capacity while ECS starts a replacement in the healthy subnet. The design does not protect against a regional outage, a bad image or configuration that breaks both tasks, a load balancer or DNS misconfiguration, a traffic spike (no autoscaling), the public registry being unavailable, abuse or DDoS (no WAF), or data loss (nothing here stores state).

Before production.
HTTPS: ACM certificate, port 443 listener with a modern TLS policy, port 80 redirecting to 443, Route 53 alias to the ALB.
Logging: ALB access logs to S3, longer CloudWatch retention, alarms on 5xx, unhealthy hosts and task restarts, VPC flow logs, CloudTrail.

Autoscaling: ECS service auto scaling on CPU and ALB requests per target, minimum 2 tasks across both AZs, a maximum, and the deployment circuit breaker on.
Pipeline: remote state in S3 with locking, CI using an OIDC role instead of static keys, pull requests run `fmt`, `validate`, `plan` and a security scan with the plan posted for review, apply on merge to main after approval, image pinned by digest or from a private ECR repository, separate dev and prod environments.

5. Teardown result
Destroyed the same day the test output was saved:
terraform destroy -auto-approve


<img width="582" height="251" alt="image" src="https://github.com/user-attachments/assets/6ab44292-62de-49fb-a0d0-27fca2ec2aad" />





Repository contents
.
├── main.tf                 # Terraform: creates and destroys the whole stack
├── .terraform.lock.hcl     # provider version lock
├── .gitignore              # keeps state, plans and tfvars out of Git
├── README.md               # this file
├── proofs/                 # command output for the proofs and destroy
└── screenshots/            # AWS console and browser screenshots
```
