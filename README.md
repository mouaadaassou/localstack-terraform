# Localstack for Local Development:

## Introduction:
Provisioning AWS resources for your applications/organisation can be complex - Creating the AWS infrastructures (SQS, SNS, Lambda, S3, ...) with a fine grained permissions model - and then try to integrate 
your infrastructure with your applications will take time until you test to ensure the whole flow is working as expected ...  

### PreRequisite:
* Docker installed
* Docker-compose binary

### Starting And Managing Localstack:
You can set up localstack in different ways - install its binary script that starts a docker container,
and set all the infrastructure needed - or using Docker or docker-compose. for the full list of the available options [you can check this list](https://docs.localstack.cloud/get-started/)

In our Lab we will use docker-compose to lunch localstack, you can use the [docker-compose associated with tha lab](./docker/docker-compose.yaml).
I choose docker because I don't need to install any additional binary into my machine, I can lunch it and drop the container any time. 

### LocalStack with Docker and AWS CLI:
You can use [the official docker-compose file from Localstack repo](https://github.com/localstack/localstack/blob/master/docker-compose.yml), you can remove the unnecessary ports - only required for Pro - as we are using the free localstack.

the minimalist docker-compose.yaml [can be found here](./docker/docker-compose.yaml), and in order to lunch the service, just execute the following command - I am running it in the background (-d parameter)
```bash
docker compose -f docker/docker-compose.yaml up -d
```

## LocalStack Lab:
### Scenario:
In This Lab, We will create an SNS that has an SQS subscribed to it, and a Lambda that starts listens on that SQS, and then process the message and upload it to an S3. 
the lab will use AWS CLI to provision the infrastructure as a first option, then, we will try to do the same using terraform to create all these resources on both environments - local - using localstack, and on AWS.

![SNS-SQS-Lambda-S3-Scenario](./img/scenario.png)


#### Setting Up the AWS Configuration for Localstack:
We already lunched localstack container, you can check that the container is up and running using the command:
```bash
> docker container ls
> docker container logs localstack
...
2022-10-30T19:01:14.628  WARN --- [-functhread5] hypercorn.error            : ASGI Framework Lifespan error, continuing without Lifespan support
2022-10-30T19:01:14.628  WARN --- [-functhread5] hypercorn.error            : ASGI Framework Lifespan error, continuing without Lifespan support
2022-10-30 19:01:14,629 INFO success: infra entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2022-10-30T19:01:14.630  INFO --- [-functhread5] hypercorn.error            : Running on https://0.0.0.0:4566 (CTRL + C to quit)
2022-10-30T19:01:14.630  INFO --- [-functhread5] hypercorn.error            : Running on https://0.0.0.0:4566 (CTRL + C to quit)
Ready. <- the container is ready to accept requests
```

for each aws command, the aws binary will use the specific endpoint for that service -e.g: EC2 endpoint is https://ec2.eu-central-1.awsamazon.com
we have to override this endpoint in case of localstack, thus we have to specify the --endpoint-url each time you execute the aws command, for that I will create an alias to make life easier to interact with AWS CLI: 
```bash
alias awsls='aws --endpoint-url http://localhost:4566'
```
I am also setting the AWS PAGER, so that It will send you command result into a pager
```bash
export AWS_PAGER=""
```

We can check our configuration by listing the topics:
```bash
> awsls sns list-topics
{
    "Topics": []
}
```

#### Creating Lab Components:
We will start by creating the SQS queue that will be sitting between the SNS and our lambda, 
use the following command to create it:
```bash
> awsls sns create-topic --name "localstack-lab-sns.fifo" --attributes FifoTopic=true,ContentBasedDeduplication=false
{
    "TopicArn": "arn:aws:sns:eu-central-1:000000000000:localstack-lab-sns.fifo"
}
```

Now if you try to list the topics in localstack, you will find the one created in the previous command:
```bash
> awsls sns list-topics
{
    "Topics": [
        {
            "TopicArn": "arn:aws:sns:eu-central-1:000000000000:localstack-lab-sns.fifo"
        }
    ]
}
```

the SNS topic is created, we can create the SQS queue, for that I will start first by creating the dead-letter queue that will back our SQS:
```bash
> awsls sqs create-queue --queue-name localstack-lab-sqs-dlq.fifo --attributes FifoQueue=true,ContentBasedDeduplication=false,DelaySeconds=0
{
    "QueueUrl": "http://localhost:4566/000000000000/localstack-lab-sqs-dlq.fifo"
}
```

The SQS queue will be a fifo queue - a fifo queues don't introduce duplicate message within an interval of 5 minutes, also it guarantees the ordering of the message - for more details check [FIFO queues in the documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html)
```bash
> awsls sqs create-queue --queue-name localstack-lab-sqs.fifo --attributes file://code/queue-attributes.json
{
    "QueueUrl": "http://localhost:4566/000000000000/localstack-lab-sqs.fifo"
}
```

After creating the SNS, SQS, and the SQS DLQ, we need to subscribe SQS to the SNS topic, but for that, we will get the queue arn first:
```bash
> awsls sqs get-queue-attributes --queue-url http://localhost:4566/000000000000/localstack-lab-sqs.fifo --attribute-names All
{
    "Attributes": {
        "ApproximateNumberOfMessages": "0",
        "ApproximateNumberOfMessagesNotVisible": "0",
        "ApproximateNumberOfMessagesDelayed": "0",
        "CreatedTimestamp": "1667168817",
        "DelaySeconds": "0",
        "LastModifiedTimestamp": "1667168817",
        "MaximumMessageSize": "262144",
        "MessageRetentionPeriod": "1209600",
        "QueueArn": "arn:aws:sqs:eu-central-1:000000000000:localstack-lab-sqs.fifo",
        "ReceiveMessageWaitTimeSeconds": "0",
        "VisibilityTimeout": "900",
        "SqsManagedSseEnabled": "false",
        "ContentBasedDeduplication": "false",
        "DeduplicationScope": "queue",
        "FifoThroughputLimit": "perQueue",
        "FifoQueue": "true",
        "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:eu-central-1:000000000000:localstack-lab-sqs-dlq.fifo\",\"maxReceiveCount\":\"3\"}"
    }
}
```

after getting the SQS arn, we can subscribe the queue to the SNS as follow:
```bash
> awsls sns subscribe --topic-arn arn:aws:sns:eu-central-1:000000000000:localstack-lab-sns.fifo --protocol sqs --notification-endpoint  arn:aws:sqs:eu-central-1:000000000000:localstack-lab-sqs.fifo
{
    "SubscriptionArn": "arn:aws:sns:eu-central-1:000000000000:localstack-lab-sns.fifo:a3be6641-636d-4c23-affe-8aed78b26151"
}
```

Now we can test the SQS subscription by publishing a message to the topic, and it must be received it by the SQS: 
```bash
> awsls sns publish --topic-arn arn:aws:sns:eu-central-1:000000000000:localstack-lab-sns.fifo --message-group-id='test' --message-deduplication-id='1111'  --message file://code/sqs-message.json
{
    "MessageId": "babff1a5-993f-42c2-99d5-0551c7f1083d"
}
```

Calling the SQS's receive-message to check if we have received the message from the SNS:
```bash
> awsls sqs receive-message --queue-url http://localhost:4566/000000000000/localstack-lab-sqs.fifo
{
    "Messages": [
        {
            "MessageId": "689ac063-2628-485b-92f6-1c613c2bf943",
            "ReceiptHandle": "ZTU1N2IzODItOTE0ZC00MjNlLTg4OTAtOGFkOWE4OTRkYTU0IGFybjphd3M6c3FzOmV1LWNlbnRyYWwtMTowMDAwMDAwMDAwMDA6bG9jYWxzdGFjay1sYWItc3FzLmZpZm8gNjg5YWMwNjMtMjYyOC00ODViLTkyZjYtMWM2MTNjMmJmOTQzIDE2NjcxNjA1MDkuMzMxMjg3Ng==",
            "MD5OfBody": "492cc3f4db48b802af619016ca18939d",
            "Body": "{\"Type\": \"Notification\", \"MessageId\": \"3166be17-1ad2-4a92-81ea-46bfd085334e\", \"TopicArn\": \"arn:aws:sns:eu-central-1:000000000000:localstack-lab-sns.fifo\", \"Message\": \"'{\\n  \\\"id\\\": 1,\\n  \\\"title\\\": \\\"iPhone 9\\\",\\n  \\\"description\\\": \\\"An apple mobile which is nothing like apple\\\",\\n  \\\"price\\\": 549,\\n  \\\"discountPercentage\\\": 12.96,\\n  \\\"rating\\\": 4.69,\\n  \\\"stock\\\": 94,\\n  \\\"brand\\\": \\\"Apple\\\",\\n  \\\"category\\\": \\\"smartphones\\\"\\n}'\", \"Timestamp\": \"2022-10-30T20:08:11.572Z\", \"SignatureVersion\": \"1\", \"Signature\": \"EXAMPLEpH+..\", \"SigningCertURL\": \"https://sns.us-east-1.amazonaws.com/SimpleNotificationService-0000000000000000000000.pem\", \"UnsubscribeURL\": \"http://localhost:4566/?Action=Unsubscribe&SubscriptionArn=arn:aws:sns:eu-central-1:000000000000:localstack-lab-sns.fifo:d8398760-c768-4867-b950-6035ec226c9d\"}"
        }
    ]
}
```

So far, the SNS & SQS integration is working properly - as the messages published to SNS are sent to the SQS. but we need to process the messages by a lambda and moves them to an s3 bucket,
so let's create the s3 bucket first:
```bash
> awsls s3 mb s3://localstack-lab-bucket
make_bucket: localstack-lab-bucket
```

we can list all the existing s3 buckets in localstack as follows:
```bash
> awsls s3 ls
2022-10-30 08:51:03 localstack-lab-bucket
```

the next step is to create a Lambda function that get triggered by the SQS - SQS as source for the lambda function, and saves the messages to a file on S3 bucket.
for that, there already a simple [lambda function](./lambda.py) that get triggered whenever a new message is in the SQS.
We need to zip the script and use it to create lambda function:
```bash
# first step, zip the function handler:
> zip lambda-handler.zip lambda.py
```
now, we can use that zip file to create our lambda function handler - we are giving the bucket where to put the messages as an env var:
```bash
# create the lambda
> awsls lambda create-function --function-name queue-reader --zip-file fileb://lambda-handler.zip --handler lambda.lambda_handler --runtime python3.8 --role arn:aws:iam::000000000000:role/fake-role-role --environment Variables={bucket_name=localstack-lab-bucket}
{
    "FunctionName": "queue-reader",
    "FunctionArn": "arn:aws:lambda:eu-central-1:000000000000:function:queue-reader",
    "Runtime": "python3.8",
    "Role": "arn:aws:iam::000000000000:role/fake-role-role",
    "Handler": "lambda.lambda_handler",
    "CodeSize": 522,
    "Description": "",
    "Timeout": 3,
    "LastModified": "2022-10-30T22:28:28.221+0000",
    "CodeSha256": "TG2Q4/k6sbNqqvL+FphBRHXiK8qv3WnrSZaNtjc6rCY=",
    "Version": "$LATEST",
    "VpcConfig": {},
    "Environment": {
        "Variables": {
            "bucket_name": "localstack-lab-bucket"
        }
    },
    "TracingConfig": {
        "Mode": "PassThrough"
    },
    "RevisionId": "708001e0-d764-47dd-8272-881eaacf7777",
    "State": "Active",
    "LastUpdateStatus": "Successful",
    "PackageType": "Zip",
    "Architectures": [
        "x86_64"
    ]
}
```

```
NOTE: In the previous command - while creating the lambda function, we are using a fake role that we did not create. we will discuss this point in the limitation section of localstack.
```

Now the lambda function is created, but it won't do anything, as still it is not using the SQS as source. to do so, we need to create a mapping between an event source and the Lambda function like follow:
```bash
> awsls lambda create-event-source-mapping --function-name queue-reader --batch-size 5 --maximum-batching-window-in-seconds 60  --event-source-arn arn:aws:sqs:eu-central-1:000000000000:localstack-lab-sqs.fifo
{
    "UUID": "81ecc36a-c054-4e7f-8018-4586f39afea3",
    "StartingPosition": "LATEST",
    "BatchSize": 5,
    "ParallelizationFactor": 1,
    "EventSourceArn": "arn:aws:sqs:eu-central-1:000000000000:localstack-lab-sqs.fifo",
    "FunctionArn": "arn:aws:lambda:eu-central-1:000000000000:function:queue-reader",
    "LastModified": "2022-10-30T21:09:31.984637+01:00",
    "LastProcessingResult": "OK",
    "State": "Enabled",
    "StateTransitionReason": "User action",
    "Topics": [],
    "MaximumRetryAttempts": -1
}
```

try to send again an event to the SNS, and check the logs of your localstack container to see if there are any error in the logs:
```bash
> awsls sns publish --topic-arn arn:aws:sns:eu-central-1:000000000000:localstack-lab-sns.fifo --message-group-id='test' --message-deduplication-id='1111'  --message file://code/sqs-message.json
```

now double check that the lambda function get triggered by listing the content of our bucket:

```bash
> awsls s3 --recursive ls s3://localstack-lab-bucket
2022-10-30 23:31:45        819 /processed/command-2022_10_30-10_31_45.txt
```

## Testing AllTogether:
You can use [this bash script](code/lunch-stack.sh) to lunch the whole stack along with the AWS resources, you can find all the previous commands there, with an easy-to-use script.

```bash
> ./code/./lunch-stack.sh
```

## LocalStack Limitation:
So far, LocalStack does not enforce/validate the IAM policies, so don't rely on Localstack to test  your IAM policies and roles... they are providing a pro/enterprise version - 24$/month - that provide this feature
of validating and enforcing IAM permissions - Even if you add the ENFORCE_IAM flag to docker-compose stack, it will not work.
