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

Component	Detail
VPC	`10.20.0.0/16`
Public subnets	`10.20.1.0/24` (ap-south-1a) and `10.20.2.0/24` (ap-south-1b)
Internet gateway	Attached to the VPC; route table sends `0.0.0.0/0` to it; associated with both subnets
ALB security group	Inbound TCP 80 from `0.0.0.0/0`; attached to the load balancer
Task security group	Inbound TCP 80 from the ALB security group only; attached to the tasks
ECS cluster	`secure-web-cluster`
Task definition	Fargate, 0.25 vCPU, 0.5 GB, `awslogs` driver to CloudWatch Logs `/ecs/secure-web`
Execution role	`secure-web-task-exec-role` with `AmazonECSTaskExecutionRolePolicy` (log writes and image pull)
ECS service	Desired count 2, Fargate, task security group, public IPv4 on each task, both public subnets
Load balancer	Internet-facing ALB in both public subnets
Target group	Target type `ip`, health check on `/`
Listener	Port 80, forwards to the target group

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

PASTE proofs/proof1-alb-200.txt here (should start with HTTP/1.1 200 OK)

Evidence 2: Task security group rule, inbound TCP 80 allowed from the ALB security group

aws ec2 describe-security-groups --region ap-south-1 --group-ids (terraform output -raw task_sg_id) --query "SecurityGroups[0].IpPermissions"

PASTE proofs/proof2-task-sg.json here

The source is the ALB security group (`sg-03df34e33eb425bbd`) in `UserIdGroupPairs`, and `IpRanges` is empty. Task security group: `sg-0f7b0761d89e9010b`.

Evidence 3: `curl http://<task-public-ip>/` fails with a timeout or connection refused

curl.exe -sS -m 10 "http://<task-public-ip>/"

PASTE proofs/proof3-direct-task.txt here (should end with a timeout, for example: curl: (28) Connection timed out)

Extra evidence: two healthy targets, in two subnets
PASTE proofs/proof4-targets-healthy.json here

PASTE proofs/proof5-tasks-az.json here

Evidence 4: Screenshots from the AWS console and browser

What it shows	Screenshot
VPC `10.20.0.0/16`	![VPC](screenshots/01-vpc.png)
Two public subnets in two Availability Zones	![Subnets](screenshots/02-subnets.png)
Route table: `0.0.0.0/0` to the internet gateway, both subnets associated	![Route table](screenshots/03-route-table.png)
Task security group: TCP 80 from the ALB security group	![Task SG](screenshots/04-task-sg.png)
Application Load Balancer, internet-facing, two AZs	![ALB](screenshots/05-alb.png)
Target group: two healthy targets	![Targets](screenshots/06-targets-healthy.png)
ECS service: desired 2, running 2	![ECS service](screenshots/07-ecs-service.png)
nginx welcome page through the ALB DNS name	![Browser](screenshots/08-browser.png)
CloudWatch log stream from the containers	![Logs](screenshots/09-logs.png)
4. Design note
1. One successful request, browser to nginx. The browser resolves the ALB DNS name and opens TCP 80 to an ALB node in a public subnet. The ALB security group (`secure-web-alb-sg`) allows inbound 80 from anywhere. The listener on port 80 forwards the request to the target group (`secure-web-tg`), which picks a healthy task IP. The ALB connects to that Fargate task on port 80, and the task security group (`secure-web-task-sg`) accepts it because the source is the ALB security group. The nginx container in the task answers, and the response returns through the ALB.
2. Why the task's public IP does not open the site. The public IP gives the task a route out, but inbound traffic is still filtered by the task security group. Its only inbound rule is TCP 80 from the ALB security group, and a user's computer is not a member of that group. Security groups deny inbound traffic by default, so the packet is dropped and the client times out (Evidence 3).
3. How the task pulls the image. Here the task has a public IPv4 address in a public subnet, so it reaches `public.ecr.aws` through the internet gateway, and the task security group allows all outbound traffic. With only a private IP there is no public address for the internet gateway to use, so the task needs either a NAT gateway in a public subnet (private subnets route `0.0.0.0/0` to it), or VPC endpoints: `ecr.api` and `ecr.dkr` interface endpoints, an S3 gateway endpoint for image layers, and a `logs` endpoint. Pulling from public ECR usually still needs NAT unless the image is copied to a private ECR repository first.
4. If one Availability Zone is down. The ALB keeps sending traffic to the task in the other AZ, so the site stays up at half capacity while ECS starts a replacement in the healthy subnet. The design does not protect against a regional outage, a bad image or configuration that breaks both tasks, a load balancer or DNS misconfiguration, a traffic spike (no autoscaling), the public registry being unavailable, abuse or DDoS (no WAF), or data loss (nothing here stores state).
5. Before production.
HTTPS: ACM certificate, port 443 listener with a modern TLS policy, port 80 redirecting to 443, Route 53 alias to the ALB.
Logging: ALB access logs to S3, longer CloudWatch retention, alarms on 5xx, unhealthy hosts and task restarts, VPC flow logs, CloudTrail.
Autoscaling: ECS service auto scaling on CPU and ALB requests per target, minimum 2 tasks across both AZs, a maximum, and the deployment circuit breaker on.
Pipeline: remote state in S3 with locking, CI using an OIDC role instead of static keys, pull requests run `fmt`, `validate`, `plan` and a security scan with the plan posted for review, apply on merge to main after approval, image pinned by digest or from a private ECR repository, separate dev and prod environments.
5. Teardown result
Destroyed the same day the test output was saved:
```powershell
terraform destroy -auto-approve
```
```text
PASTE the last lines of proofs/destroy-output.txt here (should include: Destroy complete! Resources: N destroyed.)
```
Check that nothing is left (load balancer, ECS cluster, VPC):
```text
PASTE proofs/destroy-verification.txt here (all lists empty)
```
Did destroy finish? WRITE: Yes / No, and the date
Same-day cost seen in Billing: WRITE the amount, or "not yet visible, expected around USD 1"
6. Done looks like
Requirement	Where it is shown
`http://<alb-dns>/` returns 200 from nginx	Evidence 1 and the browser screenshot
Two tasks are healthy, in two subnets	Extra evidence and the target group and ECS screenshots
Task security group allows port 80 only from the ALB security group	Evidence 2 and the task security group screenshot
README answers the five questions	Design note
The submitted template can destroy the stack	Teardown result
Repository contents
```
.
├── main.tf                 # Terraform: creates and destroys the whole stack
├── .terraform.lock.hcl     # provider version lock
├── .gitignore              # keeps state, plans and tfvars out of Git
├── README.md               # this file
├── proofs/                 # command output for the proofs and destroy
└── screenshots/            # AWS console and browser screenshots
```
