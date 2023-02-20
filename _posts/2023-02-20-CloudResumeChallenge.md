---
layout: post
title: Cloud Resume Challenge
published: true
category: development
---
# Background
The [Cloud Resume Challenge](https://cloudresumechallenge.dev/docs/the-challenge/aws/) is a 16-step challenge designed to 
develop your cloud skills, while making your resume more publicly/easily accessible.

The challenge has Azure, GCP, and AWS variants that hone your skills in their respective cloud platform.

I will be completing the challenge in AWS to align my skills with upcoming work projects.

After completion of the project, I will have a highly-available, browser-renderable resume with an HTTP API backend.

# Implementation
The project consists of an HTML/JavaScript/CSS frontend hosted by S3 and CloudFront,
and a Golang API backend leveraging Lambda, DynamoDB, and API Gateway.

All AWS infrastructure is provisioned and managed via Terraform.

Terraform remote state is hosted in S3, with DynamoDB state locks.

All code is source controlled in GitHub repositories. No secrets are committed.

Github Actions handles golang testing/building, and deployment of AWS infrastructure.

## Frontend
Code repository: [https://github.com/mehlj/resume-frontend](https://github.com/mehlj/resume-frontend)
### App
The resume site is written in HTML and styled in CSS.

A small snippet of JavaScript interacts with the backend API to display a visitor counter: the amount of overall visitors to the site.
```javascript
  <div id="counter"></div>

  <script>
    fetch('https://0n0sxlzu80.execute-api.us-east-1.amazonaws.com/resume_lambda_stage/increment', {
      method: 'POST',
      headers: {
          'Accept': 'application/json',
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': 'https://resume.justenmehl.com',
          'Access-Control-Allow-Headers': 'Content-Type',
          'Access-Control-Allow-Methods': 'POST,GET'
      }
    })
    .then(response => response.json())
    .then(response => document.getElementById("counter").innerHTML = "Visitor Count: " + (JSON.stringify(response)))
  </script>
```

### Infra
All website files are stored in an S3 bucket.

The S3 bucket is configured with static website hosting.
```bash
resource "aws_s3_bucket_website_configuration" "mehlj-resume-host" {
  bucket = aws_s3_bucket.mehlj-resume.bucket

  index_document {
    suffix = "resume.html"
  }
}
```

This would normally be sufficient, but there are a couple of issues:
* The site is still using the default S3 domain name
* The site is HTTP only (no TLS)

CloudFront will allow us to globally cache our site files using edge computing, while terminating TLS connections to the bucket. 

CloudFront uses a default domain and certificate, but both can be replaced using Route53 and ACM.

With the proper certificate and domain in place (via Terraform as well), we can create a CloudFront distribution that:
* routes traffic to the S3 bucket (while caching where possible)
* terminates TLS using a custom ACM certificate
* uses a custom domain name (resume.justenmehl.com)

```bash
resource "aws_cloudfront_distribution" "s3_distribution" {
  origin {
    domain_name = aws_s3_bucket.mehlj-resume.bucket_regional_domain_name
    origin_id   = local.s3_origin_id
  }

  aliases = ["resume.justenmehl.com"]

  <snip>

  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate_validation.cert_validate.certificate_arn
    ssl_support_method  = "sni-only"
  }
}
```

With all this in place, our frontend is complete. We still, however, need our backend to implement an atomic counter (visitor count).

## Backend
Code repository: [https://github.com/mehlj/resume-backend](https://github.com/mehlj/resume-backend)
### App
The backend consists of:
* an HTTP API layer to communicate with a database
* a NoSQL database to track the count

The API is written in Golang (complete with unit tests). It interacts with DynamoDB to increment an `N`-type data set (`counterValue`): a single integer to track the amount of visits to the site.

```go
  input := &dynamodb.UpdateItemInput {
    TableName: aws.String(tableName),
    Key: map[string]*dynamodb.AttributeValue{
      "primaryKey": {
          S: aws.String("VisitorCounter"),
      },
    },
    ExpressionAttributeValues: map[string]*dynamodb.AttributeValue{
      ":num": {
          N: aws.String("1"),
      },
    },
    ReturnValues:     aws.String("UPDATED_NEW"),
    UpdateExpression: aws.String("set counterValue = counterValue + :num"),
  }
```

Since the API has little overhead, Lambda is a good choice to execute the code. 

We can simply import the Lambda golang library and trigger our API function whenever Lambda executes this code.
```go
import (
  <snip>
  "github.com/aws/aws-lambda-go/lambda"
)

func incrementCounter() {
<snip>
}

func main() {
  lambda.Start(incrementCounter)
}
```

### Infra
The DynamoDB table + initial counter value are provisioned using Terraform:
```bash
resource "aws_dynamodb_table_item" "visitor-counter" {

  # ignore changes to counterValue - this will change over time
  lifecycle {
    ignore_changes = [
      item,
    ]
  }

  table_name = aws_dynamodb_table.resume-counter.name
  hash_key   = aws_dynamodb_table.resume-counter.hash_key

  item = <<ITEM
{
  "primaryKey": {"S": "VisitorCounter"},
  "counterValue": {"N": "0"}
}
ITEM
}

resource "aws_dynamodb_table" "resume-counter" {
  name         = "resume-counter"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "primaryKey"

  attribute {
    name = "primaryKey"
    type = "S"
  }
}
```

The Lambda is provisioned using Terraform:
```bash
resource "aws_lambda_function" "function" {
  filename         = "lambda-handler.zip"
  function_name    = "resume-visitor-counter"
  source_code_hash = filebase64sha256("lambda-handler.zip")
  role             = aws_iam_role.lambda-role.arn

  handler = "main"
  runtime = "go1.x"
}
```
The API Gateway and the Lambda integration are provisioned with Terraform as well:
```bash
resource "aws_apigatewayv2_api" "lambda" {
  name          = "resume_lambda_gw"
  protocol_type = "HTTP"

  cors_configuration {
    allow_origins     = ["https://resume.justenmehl.com"]
    allow_credentials = "true"
    allow_headers     = ["*"]
    allow_methods     = ["*"]
    max_age           = 300
  }
}
resource "aws_apigatewayv2_integration" "lambda" {
  api_id           = aws_apigatewayv2_api.lambda.id
  integration_type = "AWS_PROXY"

  integration_uri    = aws_lambda_function.function.invoke_arn
  integration_method = "POST"
}

resource "aws_apigatewayv2_route" "increment" {
  api_id = aws_apigatewayv2_api.lambda.id

  route_key = "POST /increment"
  target    = "integrations/${aws_apigatewayv2_integration.lambda.id}"
}
<snip>
```

The API Gateway route will allow for `POST` requests to `https://url/increment`, which increments the visitor counter and returns the new value back to the user.

# Lessons Learned

## Domain Registration
Most of the Terraform modules I read to develop my knowledge on this tech stack had an external requirement of a pre-registered Route53 domain name.

Seeing as I was trying to perform every AWS interaction with IaC, I tried creating a Route53 zone for a domain name that wasn't registered/purchased yet (`justenmehl.com`).

While the zone created successfully, any interaction with the domain failed (lookups, etc.).

I noticed this initially when I tried to use the `aws_acm_certificate_validation` resource to programatically assert that I own the domain.

This resource timed out every time. This was resolved only after I went into the Route53 AWS Console and manually registered my domain name, while entering in my contact information. 

After the domain was registered and AWS sent me the validation email, the `aws_acm_certificate_validation` worked.

I can only surmise that domain registration is something that shouldn't be represented as code - or perhaps Route53 isn't suited for that yet. For my use-case, registering the domain once and enabling auto-renewal was sufficient.

## API Gateway Request/Return Format
At a certain point, my Lambda was working properly and returning the correct information. However, when routing my `POST /increment` request through API Gateway, the server always returned `500 Internal Server Error`.

Despite this error, the counter in DynamoDB still incremented.

This ruled out a permission issue, and led me to believe it was more of a API formatting issue.

After some research, I discovered that API Gateway HTTP APis require a certain format, similar to:
```json
StatusCode: 200
Body: some_text (string)
```

Rather than fiddle with trial and error, the golang library `github.com/aws/aws-lambda-go/events` provides some pre-made Types that work out of the box:
* `events.APIGatewayProxyRequest`
* `events.APIGatewayProxyResponse`

To fit this into the Lambda workflow, we can edit our Handler function to accept an `APIGatewayProxyRequest` as an argument, and return a `APIGatewayProxyResponse` - the format that APIGateway is expecting.

```golang
func incrementCounter(req events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
  <snip>

  return events.APIGatewayProxyResponse{
    StatusCode: http.StatusOK,
    Body:       strconv.Itoa(getCounter()),
  }, nil
```

After this addition, `POST` requests to the API Gateway endpoint returned the proper information without a `500` error.

## CORS
Once the API Gateway was configured and properly accepting requests, CORS was preventing the domain `resume.justenmehl.com` from performing a `POST` request to the API Gateway domain.

This is intended behavior in the browser, as these are two distinct domains.

In order to solve this issue, CORS must be enabled on the API Gateway side:
```bash
  cors_configuration {
    allow_origins     = ["https://resume.justenmehl.com"]
    allow_credentials = "true"
    allow_headers     = ["*"]
    allow_methods     = ["*"]
    max_age           = 300
  }
```

The Lambda must have a certain response format as well, but if we are using `events.APIGatewayProxyRequest` and `events.APIGatewayProxyResponse`, this is proactively solved.

After adding these CORS exceptions, the visitor counter could be rendered on `resume.justenmehl.com`.