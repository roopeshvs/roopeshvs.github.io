+++
title = "Automatically Scaling Down Lambda Provisioned Concurrency"
tags = ["aws", "lambda", "cloudwatch", "problems-at-scale"]
date = 2024-03-01
+++

If you are using [AWS Lambda](https://aws.amazon.com/lambda/) to serve real-time traffic and your Lambda initialization times are high, minimizing response time becomes crucial. One option to achieve this is by utilizing provisioned concurrency. [Provisioned Concurrency](https://aws.amazon.com/blogs/aws/new-provisioned-concurrency-for-lambda-functions/) refers to the number of pre-initialized execution environments allocated to your Lambda function.

Maintaining a constant provisioned concurrency capacity of 200 for a 512 MB Lambda for a month can cost approximately **$1,116 USD**! This cost is in addition to the pricing for requests and duration. AWS offers the option to scale the provisioned concurrency of Lambda based on schedule and demand. We implemented provisioned concurrency scaling for our Lambdas based on the `ProvisionedConcurrencyUtilization` metric.

> **Note:** At the time of writing this, **provisioned concurrency scaling** is not available from the AWS Management Console but can be configured using APIs, SDKs, or CLIs.

One of our Lambdas has an initialization time of over 40 seconds as it loads several ML models as pickled Python objects. Without a pre-initialized Lambda instance available, any request coming through the API Gateway would fail due to the API Gateway timeout of 29 seconds.

We configured the provisioned concurrency capacity to scale from a minimum of 10 to a maximum of 200. This ensures that there are at least 10 pre-initialized Lambda instances ready to serve requests at all times, with the capacity scaling up to 200 pre-initialized instances when necessary. During a quick load test, the provisioned concurrency scaled up effectively, maintaining the Lambda response time at less than 100 ms throughout. Even with 30,000 requests per minute, only about 100 provisioned concurrency capacity was required.

**Everything seemed to be working well, but there was a catch:** While it was reassuring to see the provisioned concurrency scale up to 100 to handle traffic without errors, it did not automatically scale down afterward. This was concerning, and we only noticed it 48 hours later, resulting in unnecessary costs during periods of low or no traffic. To address this, we quickly ran an AWS CLI command in a loop for all Lambdas to bring down their provisioned concurrency to the minimum value:

```bash
aws lambda put-provisioned-concurrency-config \
    --function-name generic-function-name \
    --qualifier latest \
    --provisioned-concurrent-executions 10
```

**Why didn't the provisioned concurrency of the Lambdas automatically scale down when there were no requests?** Provisioned concurrency scaling works with CloudWatch Alarms managed by AWS when we create the scaling policy. Upon reviewing the alarms, we found that the action to scale down was not triggered because the alarm for low provisioned concurrency utilization never activated due to "insufficient data." When there are no requests and provisioned concurrency is not being utilized, the alarms transition to the "insufficient data" stat and alarm actions do not get triggered in  this state. There is an option in CloudWatch Alarms to [set how to treat missing data](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html#alarms-and-missing-data). Unfortunately, this configuration option is not available when creating an autoscaling policy, only when creating an alarm directly. Also, editing the alarm directly is not recommended by AWS for alarms created with auto-scaling target tracking policies.

After extensive research, we found limited discussion on this issue online. Broadening our search, we looked for cases where CloudWatch alarms did not trigger due to insufficient data when there were no metrics to report. We came across a [Stack Overflow answer](https://stackoverflow.com/a/66331752) suggesting the use of the [FILL function](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/using-metric-math.html) to create a new metric that returns 0 whenever the original metric returns "insufficient data." This idea seemed promising.

We created the following Target Tracking policy JSON to implement this solution:

```json
// target-tracking.json
{
    "CustomizedMetricSpecification": {
        "Metrics": [
            {
                "Label": "ProvisionedConcurrencyUtilization",
                "Id": "m1",
                "MetricStat": {
                    "Metric": {
                        "MetricName": "ProvisionedConcurrencyUtilization",
                        "Namespace": "AWS/Lambda",
                        "Dimensions": [
                            {
                                "Name": "FunctionName",
                                "Value": "generic-function-name"
                            },
                            {
                                "Name": "Resource",
                                "Value": "generic-function-name:latest"
                            }
                        ]
                    },
                    "Stat": "Maximum"
                },
                "ReturnData": false
            },
            {
                "Label": "ProvisionedConcurrencyUtilization where Missing Data = 0",
                "Id": "e1",
                "Expression": "FILL(m1, 0)",
                "ReturnData": true
            }
        ]
    },
    "TargetValue": 0.7,
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 60,
    "DisableScaleIn": false
}
```

We tested the Lambda function using the following `put-scaling-policy` AWS CLI command:

```bash
aws application-autoscaling put-scaling-policy --service-namespace lambda \
    --scalable-dimension lambda:function:ProvisionedConcurrency \
    --resource-id generic-function-name \
    --policy-name generic-target-tracking-scaling-policy --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration file://target-tracking.json
```

Upon reviewing AWS CloudWatch Alarms and filtering for `generic-function-name`, we were pleased to see the new metric `ProvisionedConcurrencyUtilization where Missing Data = 0` populating 0 even when `ProvisionedConcurrencyUtilization` returned "insufficient data." Running another load test, this time waiting for no requests, the alarm triggered as expected, gracefully bringing down the provisioned concurrency capacity.

With the solution confirmed to be working, we updated our Terraform Lambda module to apply this change to all Lambdas. You can find a snippet of the Terraform resource `aws_appautoscaling_policy` [here](https://gist.github.com/roopeshvs/e2a6d8ca10cf7fe5087f3878e6e08882).

Now, even when there are no incoming requests, the CloudWatch alarm for provisioned concurrency autoscaling triggers using the new metric, automatically scaling down and leading to lower cloud bills! 🍻
