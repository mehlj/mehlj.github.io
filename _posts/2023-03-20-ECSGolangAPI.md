---
layout: post
title: ECS Golang REST API
published: true
category: development
---
# Background
This project involves deploying a simple Golang REST API into AWS, using managed services like ECS (with the Fargate launch type) and RDS.

After completion of the project, I will have a highly-available, scalable, and containerized REST API available on public internet.

Additionally, I will have a pipeline that allows for simple blue/green updates to the containerized application in order to support future development efforts.

# Implementation
Code repository: [https://github.com/mehlj/ecs-golang-api](https://github.com/mehlj/ecs-golang-api)

The project consists of application code (Golang) that comprises the API, and declarative IaC that defines the infrastructure necessary to support the API (Terraform).

All AWS infrastructure is provisioned and managed via Terraform.

Terraform remote state is hosted in S3, with DynamoDB state locks.

All code is source controlled in GitHub repositories. The limited usage of secrets are restricted to AWS Secrets Manager.

Github Actions handles deployment of the application and dependent AWS infrastructure.

## Application
The REST API is written entirely in Go.

It interacts with a Postgres backend, defined via a standard DSN/connection string environment variable `PG_DSN`.

It idempotently configures the database with a single table in the format the application is expecting.

Interaction with the API revolves around simple `CRUD` operations:
```
CREATE: POST /product
READ:   GET /products or GET /product?name=object
UPDATE: PUT /product
DELETE: DELETE /product
```

The API uses Go libraries `net/http` and `github.com/gorilla/mux` to serve HTTP:
```go
// URL : /product?name=apple
func QueryProduct(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")

	k := r.URL.Query().Get("name")

	p := QueryRow(k)
	json.NewEncoder(w).Encode(p)
}
```
```go
<snip>
	r := mux.NewRouter().StrictSlash(true)

	r.HandleFunc("/", DefaultHandler)
	r.HandleFunc("/product", QueryProduct).Methods("GET")
<snip>
```

SQL interaction uses prepared statements and leverages the libraries `database/sql` and `github.com/lib/pq`:
```go
<snip>
func RemoveRow(product Product) {
	// open connection
	db, err := sql.Open("postgres", os.Getenv("PG_DSN"))
	checkSQLError(err)

	// delete product
	_, err = db.Exec("DELETE FROM products WHERE name=$1", product.Name)
	checkSQLError(err)
}

func checkSQLError(err error) {
	if err != nil {
		panic(err)
	}
}
```

The application is packaged into a Docker image using a simple `Dockerfile` at the root of the repository:
```Dockerfile
FROM golang:1.15-buster

WORKDIR /opt/mehlj-pipeline/api/

COPY api/* ./

RUN go mod download

RUN go build main.go sql.go

CMD ["./main"]
```


## Infrastructure
The AWS infrastructure that supports this application stack mainly consists of ECR, ECS, RDS, a VPC carved into private/public subnets, and an ALB to handle ingress to the application.

A GitHub Actions pipeline provisions infrastructure (declared by Terraform IaC) and deploys the application in the following general order.

### Container Image Repository
Before the Docker image of the API is built, a repository/registry for the image must exist.

Since we are on AWS, usage of ECR makes sense, as it is a managed service that integrates nicely with security groups and IAM.

We can define our ECR repository using the `aws_ecr_repository` resource:
```bash
resource "aws_ecr_repository" "ecr" {
  name = "mehlj-pipeline"
}
```

Along with repository, lifecycle, and IAM policies necessary to access and configure the repository:
```bash
data "aws_iam_policy_document" "ecrpolicy" {
  <snip>
}
resource "aws_ecr_repository_policy" "ecrpolicy" {
  repository = aws_ecr_repository.ecr.name
  policy     = data.aws_iam_policy_document.ecrpolicy.json
}
resource "aws_ecr_lifecycle_policy" "repositoryPolicy" {
  <snip>
}
```
A Terraform output is defined, such that when this resource is provisioned, it will output + store the unique/random repository URL that is needed to push images.
```bash
output "repository_url" {
  description = "URL for the ECR repository."

  value = aws_ecr_repository.ecr.repository_url
}
```

### Database
The API uses Postgres-style prepared statements, so we will provision an RDS instance with the Postgres engine.

We can provision a database with the `aws_db_instance` resource:
```bash
<snip>
resource "aws_db_instance" "mehlj-pipeline" {
  identifier             = "mehlj-pipeline"
  db_name                = "ecspoc"
  instance_class         = "db.t3.micro"
  allocated_storage      = 5
  engine                 = "postgres"
  engine_version         = "14.1"
  username               = "dbuser"
  password               = jsondecode(data.aws_secretsmanager_secret_version.current.secret_string)["vault"]
  db_subnet_group_name   = aws_db_subnet_group.mehlj-pipeline.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  parameter_group_name   = aws_db_parameter_group.mehlj-pipeline.name
  skip_final_snapshot    = true
}
```
Again, we will define outputs that allow us to track the resource-unique URLs that are necessary to connect to our database later:
```bash
output "rds_hostname" {
  description = "RDS instance hostname"
  value       = aws_db_instance.mehlj-pipeline.address
  sensitive   = true
}

output "rds_port" {
  description = "RDS instance port"
  value       = aws_db_instance.mehlj-pipeline.port
  sensitive   = true
}

output "rds_username" {
  description = "RDS instance root username"
  value       = aws_db_instance.mehlj-pipeline.username
  sensitive   = true
}
```

### Build
To build our container image, we simply need to read our `Dockerfile` and do a standard `docker build` and `docker push` workflow.

Since we are pushing to an ECR registry, we need to ensure we are logged into ECR first. 

Amazon provides an Action for this common task, which we can leverage in our pipeline:
```yaml
    - name: Login to Amazon ECR
      uses: aws-actions/amazon-ecr-login@v1
```

After login, we can build and push our image with the commit SHA as the image tag:
```yaml
<snip>
    outputs:
      FULL_IMAGE_TAG: ${{ steps.get_image_tag.outputs.FULL_IMAGE_TAG }}
<snip>

    - name: Build, tag, and push the image to Amazon ECR
      id: get_image_tag
      env:
        ECR_REPOSITORY: ${{needs.create_repo.outputs.REPOSITORY_URL}}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REPOSITORY:$IMAGE_TAG
        echo "FULL_IMAGE_TAG=$ECR_REPOSITORY:$IMAGE_TAG" >> "$GITHUB_OUTPUT"
```

Using [Job Outputs](https://docs.github.com/en/actions/using-jobs/defining-outputs-for-jobs), we can pass the full image tag onto a later stage of the pipeline. Namely - the deployment stage will need it, since it is responsible for deploying said container image.


### Deploy
This stage is responsible for provisioning all other resources that the application depends on. Namely, the VPC, subnets, the ECS cluster/service/task definition, and the frontend ALB.

#### VPC/Subnets
To simulate a production environment, and better protect our AWS resources, a VPC with multiple subnets and varying levels of accessibility was provisioned.

A large private IP block was instantiated, with the intent of carving it up:
```bash
resource "aws_vpc" "ecs_vpc" {
  cidr_block = "10.128.0.0/16"
}
```

One group of subnets within this VPC must be publicly accessible, since the application itself will be publicly available.

These subnets will house the publicly-facing ALB and provide outbound internet access via NAT Gateways. 
```bash
data "aws_availability_zones" "available_zones" {
  state = "available"
}

<snip>

resource "aws_subnet" "public" {
  count             = 2
  cidr_block        = cidrsubnet(aws_vpc.ecs_vpc.cidr_block, 8, 2 + count.index)
  vpc_id            = aws_vpc.ecs_vpc.id
  availability_zone = data.aws_availability_zones.available_zones.names[count.index]

  # instances launched in subnet get assigned a public IP
  map_public_ip_on_launch = true
}
```

The other group of subnets should not be publicly exposed.

These subnets will house all other IP resources, including ECS tasks and RDS.
```bash
resource "aws_subnet" "private" {
  count             = 2
  cidr_block        = cidrsubnet(aws_vpc.ecs_vpc.cidr_block, 8, count.index)
  vpc_id            = aws_vpc.ecs_vpc.id
  availability_zone = data.aws_availability_zones.available_zones.names[count.index]
}
```

#### ECS
First, the ECS cluster must be provisioned:
```bash
resource "aws_ecs_cluster" "ecs_cluster" {
  name = "poc-cluster"
}
```

Next, we must create a JSON "task definition" that ECS will use to define container specifications, such as CPU, memory, the requested image, environment variables, etc.

We will leverage Fargate to limit the toil involved in managing dedicated ECS cluster members, and again, to simulate a production environment.

```bash
resource "aws_ecs_task_definition" "taskdef" {
  family       = "mehlj-pipeline"
  network_mode = "awsvpc"

  # EC2 backing vs fargate
  requires_compatibilities = ["FARGATE"]
  cpu                      = 1024
  memory                   = 2048

  # Summary: Execution Roles grant the ECS agents permission to make AWS API calls.
  # I.e.: The task is able to send container logs to CloudWatch or pull an image from ECR.
  # -----
  # allows ECS agent to pull image from ECR
  execution_role_arn = aws_iam_role.ecsTaskExecution_role.arn

  container_definitions = jsonencode([
    {
      name      = "mehlj-pipeline"
      image     = var.image_tag
      cpu       = 1024
      memory    = 2048
      essential = true
      environment = [
        {
          name  = "PG_DSN"
          value = var.pg_dsn
        }
      ]
      network_mode = "awsvpc"
      portMappings = [
        {
          containerPort = 80
          hostPort      = 80
        }
      ]
      # needs IAM as well
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = "ecs-poc"
          awslogs-region        = "us-east-1"
          awslogs-create-group  = "true"
          awslogs-stream-prefix = "mehlj-pipeline"
        }
      }
    },
  ])
}
```

The cluster and the container specifications are defined, but we still need to tell ECS "how" to deploy that container(s) into the cluster.

There are a few methods of doing this, but in general, for long-lived servers, the resource `aws_ecs_service` is best used.

Defining an ECS Service tells ECS to keep that set of containers alive and healthy at all times, and at the `desired_count` we specify.
```bash
resource "aws_ecs_service" "service" {
  name            = "mehlj-pipeline"
  cluster         = aws_ecs_cluster.ecs_cluster.id
  task_definition = aws_ecs_task_definition.taskdef.arn
  desired_count   = 3
  launch_type     = "FARGATE"

  network_configuration {
    security_groups = [aws_security_group.task_sg.id]
    subnets         = aws_subnet.private.*.id
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.asg-tg.arn
    container_name   = "mehlj-pipeline"
    container_port   = 80
  }

  depends_on = [aws_lb_listener.ecs-lb-list]
}
```

#### ALB
The containers and underlying infrastructure is defined, but the ECS tasks are still being provisioned into the private VPC subnet.

Rather than exposing the containers directly to the public subnet, we can add an abstraction layer while adding industry-standard load balancing features to the stack.

Said layer can be represented as an Application Load Balancer. The ALB will be provisioned in the public subnet, and security groups/IAM will be configured to allow communication to the protected ECS tasks.

We can define the load balancer using the `aws_lb` resource:
```bash
resource "aws_lb" "ecs-lb" {
  name            = "ecs-lb"
  subnets         = aws_subnet.public.*.id
  security_groups = [aws_security_group.ecs_alb_sg.id]

  enable_deletion_protection = false
}
```

We can then configure our load balancer to listen on port 80 and use HTTP:
```bash
resource "aws_lb_target_group" "asg-tg" {
  name        = "asg-tg"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.ecs_vpc.id
  target_type = "ip"
}

resource "aws_lb_listener" "ecs-lb-list" {
  load_balancer_arn = aws_lb.ecs-lb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.asg-tg.arn
  }
}
```
Note: the integration of this load balancer with our ECS service is done in earlier code:
```bash
resource "aws_ecs_service" "service" {
<snip>
  load_balancer {
    target_group_arn = aws_lb_target_group.asg-tg.arn
    container_name   = "mehlj-pipeline"
    container_port   = 80
  }
<snip>
}
```


Once again, we can define a Terraform output to determine/track the unique ALB URL that is needed for browsers to communicate with our public application.

```bash
output "alb_domain_name" {
  value = aws_lb.ecs-lb.dns_name
}
```

### Validation
This last pipeline stage simply runs `curl` against the public ALB URL until the application starts responding with `HTTP 200`.

This stage allows one to quickly glean how the previous stages went, and how well the application is responding.

```bash
  validate:
    name: Ensure app is responding publicly
    runs-on: ubuntu-22.04
    needs: deploy

    steps:
      - name: cURL for 5 minutes until errors out
        env:
          ALB_URL: ${{needs.deploy.outputs.ALB_URL}}
        run: curl --head -X GET -v --retry 20 --retry-all-errors --retry-max-time 300 $ALB_URL 
```

After all pipeline stages run, the API container image will be built, all infrastructure supporting the application will be provisioned, and the API will be deployed into said infrastructure.


# Interaction
```bash
# Create
[justen.mehl@workstation ~]$ curl ecs-lb-350925889.us-east-1.elb.amazonaws.com
Hello World!!

[justen.mehl@workstation ~]$ curl -d '{"Name":"ball","quantity":4}' -X POST ecs-lb-350925889.us-east-1.elb.amazonaws.com/product
{"Name":"ball","quantity":4}

# Read
[justen.mehl@workstation ~]$ curl -d '{"Name":"oranges","quantity":2}' -X POST ecs-lb-350925889.us-east-1.elb.amazonaws.com/product
{"Name":"oranges","quantity":2}

[justen.mehl@workstation ~]$ curl ecs-lb-350925889.us-east-1.elb.amazonaws.com/product?name=oranges
{"Name":"oranges","quantity":2}

# Update
[justen.mehl@workstation ~]$ curl -d '{"Name":"ball","quantity":5}' -X PUT ecs-lb-350925889.us-east-1.elb.amazonaws.com/product
{"Name":"ball","quantity":5}

# Delete
[justen.mehl@workstation ~]$ curl -d '{"Name":"ball","quantity":5}' -X DELETE ecs-lb-350925889.us-east-1.elb.amazonaws.com/product
{"Name":"ball","quantity":5}
```




# Lessons Learned

## RDS Table Configuration - Best Practices
I reached a certain point where I had provisioned an RDS database with a Postgres engine, but I needed to create a table with two columns that my application expected.

I provisioned the database itself with Terraform, so my natural inclination was to configure the table with Terraform as well.

After more research, it looks like that workflow is an anti-pattern, and application-unique configuration such as that table should be performed in a different manner than Terraform.

At that point, we enter configuration management territory, so Ansible was an option. AWS Lambda was another option if I wanted to stay strictly within AWS tooling.

Ultimately, I opted to add the table configuration responsibility to my Go API itself:
```go
func CreateTable() {
	// open connection
	db, err := sql.Open("postgres", os.Getenv("PG_DSN"))
	checkSQLError(err)

	// format table properly
	_, err = db.Exec("CREATE TABLE IF NOT EXISTS products(name TEXT, quantity INT)")
	checkSQLError(err)
}

<snip>
func main() {
	// prep database
	CreateTable()

	r := mux.NewRouter().StrictSlash(true)

	r.HandleFunc("/", DefaultHandler)
<snip>
}
```

The API containers already had firewall rules and IAM permissions in place to interact with RDS, thus making this a simple option.

Using `IF NOT EXISTS` allowed me to make this operation idempotent, which was my main concern.

Before serving the HTTP API, the application ensures that the table format is in place.

## Docker Image Tagging Strategy
I ultimately wanted a pipeline that, upon committing ANY changes to my API code, would rebuild the application and properly roll out that change to the ECS cluster.

My first hiccup with this was when I began writing my ECS task definition and had to define my image tag:
```bash
<snip>
  container_definitions = jsonencode([
    {
      name      = "mehlj-pipeline"
      image     = ?
<snip>
```

I wanted this image tag to change after every commit. How could I hardcode this value and still have it update every time?

My first thought was to hardcode this line to use `latest`, and in my pipeline, always tag my image `latest` and push it up. That way, my task definition code stays the same, but I'm always receiving the latest image in the cluster.
```bash
<snip>
  container_definitions = jsonencode([
    {
      name      = "mehlj-pipeline"
      image     = <ECR_URL>:latest
<snip>
```

This functionally worked okay, but when I first pushed up minor changes to the API code, the containers in the cluster did not change at all, despite the pipeline running successfully.

It turns out, Terraform cannot determine if there was a change to the `latest` image. I could have configured the pipeline to force an update to the cluster, similar to `kubectl rollout`, but that seemed unclean.

My final solution was to use a Terraform variable as the value for `image` in the task definition.
```bash
variable "image_tag" {
  description = "Container image tag, corresponds to commit SHA"
  type        = string
  default     = "placeholder"
}
```
```bash
<snip>
  container_definitions = jsonencode([
    {
      name      = "mehlj-pipeline"
      image     = var.image_tag
<snip>
```

This value can change on every pipeline run, so long as I provide that image tag value through an environment variable called `TF_VAR_image_tag`.

Before I provision my resource that depends on that Terraform variable, I must supply the environment variable using a Job Output from the earlier `Build` stage.

```
<snip>
    - run: terraform apply -auto-approve
      env:
        TF_VAR_image_tag: ${{needs.build.outputs.FULL_IMAGE_TAG}}
<snip>
```

Ultimately, the full image tag corresponds to the unique SHA of the commit, so the ECS task definition is always updated upon a commit.

And when a task definition is updated (new revision), the ECS service handles proper blue/green deployment of the change to the cluster.

## Parameter Placeholders (Postgres vs SQLite)
Before this project was re-engineered to fit a cloud deployment, it was deployed into a lab Kubernetes environment and used a SQLite database mounted locally on a `Volume`.

Originally, all of my SQL prepared statements used `?` as the parameter placeholder value.
```go
stmt, err := db.Prepare("INSERT INTO products(name, quantity)  values(?,?)")
```

This works for SQLite, as well as MySQL (and some others), but evidently, Postgres uses another value.

It requires `$1`, and for multiple parameters, `$1`, `$2`, and so on.

After reworking my statements to fit this syntax, my SQL statements functioned properly.
```go
stmt, err := db.Prepare("INSERT INTO products(name, quantity)  values($1,$2)")
```

## GitHub Actions - Job Outputs and Secrets
At one point during development, I was building my Postgres DSN/connection string in one `Job` (the RDS deployment job), and trying to pass that value as an `Output` to a later `Job` (the deployment job).

Problem was, the connection string contains a sensitive combination of the RDS URL and the credentials to the database.

I missed the section of the [GitHub Actions documentation](https://docs.github.com/en/actions/using-jobs/defining-outputs-for-jobs) that stated this:
```
Outputs containing secrets are redacted on the runner and not sent to GitHub Actions.
```

I would run my pipeline, and everything looked successful, but my ECS tasks would fail because the environment variable `TF_VAR_pg_dsn` was empty, and it should have contained the Postgres DSN.

After a lot of debugging, I finally found a small warning in my pipeline:
```
Warning: Skip output 'TF_VAR_pg_dsn' since it may contain secret.
```

GitHub Actions performs secret detection, and if it detects any in an `Output`, it will silently fail the passing of said output.

Understandably so, since passing information between `Jobs` involves sending/storing the data on GitHub's infrastructure, albeit temporarily.

The fix to this problem was to properly use a Secret Manager, in this case, AWS Secrets Manager. And since that secret is highly available, I don't have to pass the output between jobs, I can simply build the DSN right before running `terraform apply`.

```yaml
    - name: Read database secrets from AWS Secrets Manager into environment variables
      uses: abhilash1in/aws-secrets-manager-action@v2.1.0
      with:
        secrets: mehlj_lab_creds
        parse-json: true

    - run: terraform init

    - name: Build Postgres DSN string
      run: echo TF_VAR_pg_dsn="postgres://$(terraform output -raw rds_username):${MEHLJ_LAB_CREDS_VAULT}@$(terraform output -raw rds_hostname):5432/ecspoc" >> $GITHUB_ENV

    - run: terraform fmt -check
    - run: terraform apply -auto-approve
```

`$GITHUB_ENV` stores temporary and job-local environment variables, and has no problem storing secrets, since they are lost after the job ends.

The ultimate best practice is to not inject this DSN as an environment variable to the ECS tasks at all - and instead store that information entirely in a secret manager, and have the ECS tasks read directly from said manager.

My first thought towards this approach revealed a challenge - that DSN URL includes a unique Terraform output, and I don't know the value until that pipeline Job executes. There may have been a way to programatically generate and store the secret, but my research stopped there. This solution seemed suitable for an educational project such as this.