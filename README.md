# Creating pre-signed signaling channel URLs for Amazon Kinesis Video Streams (KVS) with WebRTC

## Introduction
In this repository, we'll describe and demonstrate a solution for creating pre-signed signaling channel URLs. The example use-case for this solution is we have users that need to connect only to cameras they own using Amazon Kinesis Video Streams with WebRTC and we need to offload the signaling channel URL generation to the cloud. This solution enables a flexible pattern for users to dynamically access their camera's video stream using Amazon Kinesis Video Streams with WebRTC, without the need for their clients to know to how to work with AWS Signature Version 4.

## Walkthrough
For the demonstration in this repository, you will send an HTTPS request to Amazon API Gateway to dynamically generate a pre-signed signaling channel URL for Amazon KVS with WebRTC. The following process occurs after the request is received:
1. An AWS Lambda authorizer processes the value of the authorization header provided in the request
   1. If the header contains the correct value, the request proceeds. If not, a 403 forbidden response is sent back to the client.
2. The AWS Lambda authorizer queries an Amazon DynamoDB table using the values of the email and cameraName headers provided in the request
   1. If the query response shows that the email exists and has access to the cameraName, the request proceeds. If not, a 403 forbidden response is sent back to the client.
3. Amazon API Gateway will call an AWS Lambda function that will generate the pre-signed signaling channel URL using the value of the channelName query parameter provided in the request
4. The AWS Lambda function will respond with the pre-signed signaling channel URL to the client as JSON
5. The client can then connect to the pre-signed signaling channel URL as a WebSocket based connection 

The illustration below details what this solution will look like once fully implemented.

<img src="./assets/Solution%20Overview.png" />

<br /> 

### Prerequisites
To follow through this repository, you will need an <a href="https://console.aws.amazon.com/" >AWS account</a>, an <a href="https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/" >Amazon Kinesis Video Streams supported region</a>, permissions to create <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html" > AWS Identity and Access Management (IAM) roles and policies</a>, create <a href="https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html#getting-started-create-function"> AWS Lambda Functions</a>, create <a href="https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-http-tutorials.html"> HTTP APIs in Amazon API Gateway</a>, create <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/getting-started-step-1.html"> tables in Amazon DynamoDB</a>, create <a href="https://docs.aws.amazon.com/kinesisvideostreams-webrtc-dg/latest/devguide/gs-createchannel.html"> signaling channels in Amazon Kinesis Video Streams</a> and access to the <a href="https://aws.amazon.com/cli/">AWS CLI</a>. We also assume you have familiar with the basics of Linux bash commands.


### Step 1: Create the AWS Lambda Authorizer function (AWS CLI)
1. Create the IAM execution role for the Lambda function by issuing the <b>create-role</b> command
    ```   
    aws iam create-role --role-name "lambda-kvs_sigv4_URL_generator_custom_authorizer-role" \
    --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "lambda.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }'
    ```
2. Create the policy document for the role’s permissions, run the command below to create the policy document

    ```
    echo '{
            "Version": "2012-10-17",
            "Statement": [
                {
            "Effect": "Allow",
            "Action": [
                "dynamodb:*",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
            }
            ]
        }' > lambda_customauthorizer_policy.json
    ```
    <b>NOTE)</b> This IAM policy is to be use for development purposes only. We recommend following the best practice of least privilege with your IAM policy. You should consider scoping this down if deploying into a production account. 

3. Attach the policy to the role by issuing the <b>put-role-policy</b> command

    ```
    aws iam put-role-policy \
    --role-name "lambda-kvs_sigv4_URL_generator_custom_authorizer-role" \
    --policy-name "lambda-kvs_sigv4_URL_generator_custom_authorizer-policy" \
    --policy-document file://lambda_customauthorizer_policy.json
    ```

4. Create the Lambda function by issuing the <b>create-function</b> command

    ```
    aws lambda create-function --function-name kvs_sigv4_URL_generator_custom_authorizer \
    --zip-file fileb://lambda_functions/kvs_sigv4_URL_generator_custom_authorizer.zip --handler index.handler --runtime nodejs18.x \
    --role arn:aws:iam::REPLACE_ME_WITH_AWS_ACCOUNT_ID:role/lambda-kvs_sigv4_URL_generator_custom_authorizer-role
    ```

### Step 2: Create the AWS Lambda pre-signed signaling channel URL function (AWS CLI)

### Step 3: Create the Amazon KVS signaling channel (AWS CLI)
1. Create your Amazon KVS signaling channel by issuing the <b>create-signaling-channel</b> command
    ```   
    aws kinesisvideo create-signaling-channel --channel-name "aws_kvs_test_channel"
    ```

### Step 4: Create the Amazon API Gateway HTTP API (AWS CLI)

### Step 5: Create the Amazon DynamoDB table (AWS CLI)
1. Create your Amazon DynamoDB table by issuing the <b>create-table</b> command
    ```   
    aws dynamodb create-table \
    --table-name Users \
    --attribute-definitions AttributeName=email,AttributeType=S \
    --key-schema AttributeName=email,KeyType=HASH \
    --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1
    ```
2. Put a user record into your table by issuing the <b>put-item</b> command
    ```   
    aws dynamodb put-item \
    --table-name Users \
    --item '{
        "email": {"S": "test@myemail.com"},
        "Cameras": {"L": [ {"S": "Garage Camera One"} , {"S": "Garage Camera Two"}]}
    }' 
    ```


### Step 6: Test the API Gateway HTTP API

1. Test your HTTP API by issuing the following <b>Bash</b> commands
    ```
    HTTP_API_GATEWAY_ENDPOINT=REPLACE_ME_WITH_HTTP_API_ENDPOINT
    curl $HTTP_API_GATEWAY_ENDPOINT -X GET -H 'authorization: secretToken' -H 'email:test@myemail.com' -H 'cameraName:Garage Camera One'
    ```

    The output from these commands should look similar to the following:
    ```
    {"signedURL":"WSS_PRESIGNED_SIGNALING_CHANNEL_URL"}
    ```

## Cleaning up
Be sure to remove the resources created in this repository to avoid charges. Run the following commands to delete these resources:
1. ```aws iam delete-role-policy --role-name "lambda-kvs_sigv4_URL_generator_custom_authorizer-role" --policy-name "lambda-kvs_sigv4_URL_generator_custom_authorizer-policy"```
2. ```rm lambda_customauthorizer_policy.json```
3. ```aws iam delete-role --role-name "lambda-kvs_sigv4_URL_generator_custom_authorizer-role"```
4. ```aws kinesisvideo delete-signaling-channel --channel-arn "REPLACE_ME_WITH_ARN_FOR_aws_kvs_test_channel"```
5. ```aws dynamodb delete-table --table-name Users```

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

