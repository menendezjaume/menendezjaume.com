+++
banner = "images/AWS_CloudTrail_CloudWatch_SNS_Terraform_module.webp"
date = "2020-04-26T11:47:08+02:00"
tags = ["aws", "cloudtrail", "cloudwatch", "logging", "monitoring", "Terraform"]
title = "Monitor and Notify on AWS Account Root User Activity and Other Security Metrics"
short_description = "Use CloudTrail, CloudWatch and SNS to get notified when things are happening in you AWS Account."
+++

Inspired by a piece of work we've recently done at work, where we pipe all our cloud API logs to Elasticsearch and create alerts based on user and service activity, I wanted to share the _budget_ version of that that I use in my private AWS account.

To deploy all my infrastructure, I use Terraform, and at the end of this article, I share the code in the form of a Terraform module. 

First of all, I'll start by describing what AWS CloudTrail and AWS CloudWatch are.
AWS CloudTrail is a service where actions taken by a user, role, or an AWS service are recorded as events. In other words, CloudTrail is a _logging_ service.
AWS CloudWatch is a service that allows you to respond to logs by creating metrics, events, or visualising them using automated dashboards. In other words, CloudWatch is a _monitoring_ service.
Now, the implementation steps are simple, yet a bit convoluted due to the number of pieces we have to put together.

Here is a diagram describing the flow that our logs and alerts will follow.

{{< figure src="/images/AWS_CloudTrail_CloudWatch_SNS_Terraform_module.webp" class="image fit" >}}

As you can see, I stop at the creation of an SNS and its topic. The reason being that you might want to subscribe via email (not currently supported by Terraform), consume it via an SQS queue and send it to some processing engine, or put a lambda function behind for faster response time. Also, it is important to note that CloudTrail logs can - and in an enterprise-grade deployment probably should - be encrypted. I'm not doing so in this write and module as I wanted it to budget, so it is virtually free.

To begin with, we need to create the bucket where we'll send our CloudTrail logs to. This is a requirement as you can't enable a trail without referencing the destination bucket. Also, we'll make sure the bucket is completely private and create the policy for CloudTrail to push the logs.

```terraform
esource "aws_s3_bucket" "logging_bucket" {
  bucket        = var.s3_bucket_name
  force_destroy = true
  acl           = "private"

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }

  policy = data.aws_iam_policy_document.cloudtrail_policy.json

  tags = var.tags
}

resource "aws_s3_bucket_public_access_block" "block" {
  bucket = aws_s3_bucket.logging_bucket.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

data "aws_iam_policy_document" "cloudtrail_policy" {
  statement {
    sid = "AWSCloudTrailAclCheck"
    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }
    actions   = ["s3:GetBucketAcl"]
    resources = ["arn:aws:s3:::${var.s3_bucket_name}"]
  }

  statement {
    sid = "AWSCloudTrailWrite"
    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }
    actions   = ["s3:PutObject"]
    resources = ["arn:aws:s3:::${var.s3_bucket_name}${local.prefix}/AWSLogs/${data.aws_caller_identity.current.account_id}/*"]
    condition {
      test     = "StringEquals"
      variable = "s3:x-amz-acl"
      values = [
        "bucket-owner-full-control",
      ]
    }
  }
}
```

Next, we will enable CloudTrail in our account. Ideally, we want to make it multi-regional and include global services' events (such as IAM). If we have an organisation, we will also want to make it an organisational trail. Because I haven't got that, I'm not doing it.

```terraform
resource "aws_cloudtrail" "cloudtrail" {
  name                          = var.cloudtrail_name
  s3_bucket_name                = aws_s3_bucket.logging_bucket.id
  s3_key_prefix                 = var.prefix # Using the var.prefix as we don't want the "/" injected in local
  include_global_service_events = true
  enable_logging                = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  cloud_watch_logs_group_arn    = aws_cloudwatch_log_group.cloudtrail.arn
  cloud_watch_logs_role_arn     = aws_iam_role.cloudtrail_to_cloudwatch.arn

  tags = var.tags
}
```

Once that's done, we have to create a log group in CloudWatch. This log group acts as a pool that CloudWatch uses to perform actions on. 

```terraform
resource "aws_cloudwatch_log_group" "cloudtrail" {
  name = "${var.namespace}_cloudtrail"

  retention_in_days = var.retention_in_days
  tags              = var.tags
}

```

Now, we need to link our CloudTrail and our CloudWatch log group. To do so, we need to create a role so CloudTrail can access the CloudWatch log group, and create a log stream to push the logs to.

```terraform
resource "aws_iam_role" "cloudtrail_to_cloudwatch" {
  name = "${var.namespace}_cloudtrail-to-cloudwatch"

  assume_role_policy = data.aws_iam_policy_document.cloudtrail_assume_role_policy_document.json
}

data "aws_iam_policy_document" "cloudtrail_assume_role_policy_document" {
  statement {
    principals {
      type        = "Service"
      identifiers = ["cloudtrail.amazonaws.com"]
    }
    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role_policy" "cloudtrail_role_policy" {
  name = "cloudtrail-role-policy"
  role = aws_iam_role.cloudtrail_to_cloudwatch.id

  policy = data.aws_iam_policy_document.cloudtrail_role_policy_document.json
}

data "aws_iam_policy_document" "cloudtrail_role_policy_document" {
  statement {
    sid       = "AWSCloudTrailCreateLogStream"
    actions   = ["logs:CreateLogStream"]
    resources = [aws_cloudwatch_log_group.cloudtrail.arn]
  }
  statement {
    sid       = "AWSCloudTrailPutLogEvents"
    actions   = ["logs:PutLogEvents"]
    resources = [aws_cloudwatch_log_group.cloudtrail.arn]
  }
}
```

We have our CloudTrail logs into CloudWatch! 

Next, we will create an SNS topic, which easy enough to do.

```terraform
resource "aws_sns_topic" "security_alerts" {
  name         = "security-alerts-topic"
  display_name = "Security Alerts"

  tags = var.tags
}
```

Once we have the logs in CloudWatch, and our SNS topic created, we will create two more CloudWatch objects. First, we will create a metric filter that will keep only the logs that are relevant to us. In this example, I am creating a metric filter for the detection of root user login.

```terraform
resource "aws_cloudwatch_log_metric_filter" "root_login" {
  name           = "root-access"
  pattern        = "{$.userIdentity.type = Root}"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name      = "RootAccessCount"
    namespace = "CloudTrail"
    value     = "1"
  }
}
```

Once this metric filter is created, we will, in turn, create a metric alarm, and push this one out to our SNS topic. This way, every time our metric alarm crosses a given threshold, a message will be pushed to SNS, and from there, well, it will go to whatever has subscribed to that topic. 

```terraform
resource "aws_cloudwatch_metric_alarm" "root_login" {
  alarm_name          = "root-access-${data.aws_region.current.name}"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = "1"
  metric_name         = "RootAccessCount"
  namespace           = "CloudTrail"
  period              = "60"
  statistic           = "Sum"
  threshold           = "1"
  alarm_description   = "Use of the root account has been detected"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}
```

In the code above, you can see that our threshold is `1`. We've done it this way because we want to be notified as soon as there is one instance of the above rule (which means after one root user login).

And that's it! Now you can consume this notification from SNS by subscribing to the topic. This could be an email address, an SQS queue, a lambda function and so on.

To wrap up, and as promised, [here](https://github.com/menendezjaume/terraform-cloudtrail-cloudwatch-sns) is the link to the Terraform module I published on Github, with all these snippets of code together.

This module, as well as creating the metric filter and alert for root user login, will also create metric filters and alerts for the following actions:

- Root login
- Console login without MFA
- Action without MFA
- Illegal use of a KMS key
- Use of a KMS Key to decrypt
- Changes in security groups
- Changes in IAM
- Changes in route tables
- Changes in NACL

And it's usage is very simple:

```terraform
module "security_alerts" {
  source = "github.com/menendezjaume/terraform-cloudtrail-cloudwatch-sns"

  s3_bucket_name  = "my-bucket-name"
  namespace       = "my-namespace"
  cloudtrail_name = "my-cloudtrail-names"
}
```

What does your logging and monitoring infrastructure look like? I'm very interested to know. Also, If you've got any questions or problems with the code above, feel free to reach out, I'm more than happy to help.
