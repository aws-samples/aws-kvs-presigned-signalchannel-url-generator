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


<b>NOTE)</b> There are multiple steps that have a placeholder named ```<REPLACE_ME_WITH...>```. You will need to replace these placeholders with their respected values. For example, ```<REPLACE_ME_WITH_AWS_ACCOUNT_ID>``` will need to be replaced with your AWS Account ID before the step can be fully completed.  

### Step 1: Create the AWS Lambda authorizer function (AWS CLI)
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
    --role arn:aws:iam::<REPLACE_ME_WITH_AWS_ACCOUNT_ID>:role/lambda-kvs_sigv4_URL_generator_custom_authorizer-role
    ```

### Step 2: Create the AWS Lambda pre-signed signaling channel URL function (AWS CLI)
1. Create the IAM execution role for the Lambda function by issuing the <b>create-role</b> command
    ```   
    aws iam create-role --role-name "lambda-kvs_sigv4_URL_generator-role" \
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
                "kinesisvideo:*",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
            }
            ]
        }' > lambda_kvs_sigv4_URL_generator_policy.json
    ```
    <b>NOTE)</b> This IAM policy is to be use for development purposes only. We recommend following the best practice of least privilege with your IAM policy. You should consider scoping this down if deploying into a production account. 

3. Attach the policy to the role by issuing the <b>put-role-policy</b> command

    ```
    aws iam put-role-policy \
    --role-name "lambda-kvs_sigv4_URL_generator-role" \
    --policy-name "lambda-kvs_sigv4_URL_generator-policy" \
    --policy-document file://lambda_kvs_sigv4_URL_generator_policy.json
    ```

4. Create the Lambda function by issuing the <b>create-function</b> command

    ```
    aws lambda create-function --function-name kvs_sigv4_URL_generator \
    --zip-file fileb://lambda_functions/kvs_sigv4_URL_generator.zip --handler index.handler --runtime nodejs18.x \
    --role arn:aws:iam::<REPLACE_ME_WITH_AWS_ACCOUNT_ID>:role/lambda-kvs_sigv4_URL_generator-role
    ```

### Step 3: Create the Amazon KVS signaling channel (AWS CLI)
1. Create your Amazon KVS signaling channel by issuing the <b>create-signaling-channel</b> command
    ```   
    aws kinesisvideo create-signaling-channel --channel-name "aws_kvs_test_channel"
    ```

### Step 4: Create the Amazon API Gateway HTTP API (AWS CLI)
1. Create your Amazon API Gateway HTTP API by issuing the <b>create-api</b> command
    ```   
    aws apigatewayv2 create-api --name kvs_signed_url_generator --protocol-type HTTP --target <REPLACE_ME_WITH_ARN_FOR_kvs_sigv4_URL_generator_LAMBDA_FUNCTION>
    ```
2. Set your Amazon API Gateway HTTP API's Lambda authorizer by issuing the <b>create-authorizer</b> and <b>update-route</b> commands
    ```   
    aws apigatewayv2 create-authorizer \
    --api-id <REPLACE_ME_WITH_API_ID> \
    --authorizer-type REQUEST \
    --identity-source '$request.header.Authorization' \
    --name lambda-authorizer \
    --authorizer-uri 'arn:aws:apigateway:<REPLACE_ME_WITH_AWS_REGION>:lambda:path/2015-03-31/functions/<REPLACE_ME_WITH_ARN_FOR_kvs_sigv4_URL_generator_custom_authorizer_LAMBDA_FUNCTION>/invocations' \
    --authorizer-payload-format-version '2.0' \
    --enable-simple-responses

    #ENSURE TO RECORD THE "AuthorizerId" VALUE FROM THE OUTPUT OF THE COMMAND ABOVE

    aws apigatewayv2 update-route \
    --api-id <REPLACE_ME_WITH_API_ID> \
    --route-id <REPLACE_ME_WITH_API_ROUTE_ID> \
    --authorization-type CUSTOM \
    --authorizer-id <REPLACE_ME_WITH_API_AUTHORIZER_ID_FROM_PREVIOUS_COMMAND>
    ```
3. Add resource permissions so your Amazon API Gateway HTTP API can invoke your Lambda functions by issuing the <b>add-permission</b> command
    ```   
    aws lambda add-permission \
    --statement-id <REPLACE_ME_WITH_GUID> \
    --action lambda:InvokeFunction \
    --function-name "kvs_sigv4_URL_generator" \
    --principal apigateway.amazonaws.com \
    --source-arn 'arn:aws:execute-api:<REPLACE_ME_WITH_AWS_REGION>:<REPLACE_ME_WITH_AWS_ACCOUNTID>:<REPLACE_ME_WITH_API_ID>/*/$default'

    aws lambda add-permission \
    --function-name kvs_sigv4_URL_generator_custom_authorizer \
    --statement-id <REPLACE_ME_WITH_GUID> \
    --action lambda:InvokeFunction \
    --principal apigateway.amazonaws.com \
    --source-arn "arn:aws:execute-api:<REPLACE_ME_WITH_AWS_REGION>:<REPLACE_ME_WITH_AWS_ACCOUNTID>:<REPLACE_ME_WITH_API_ID>/authorizers/<REPLACE_ME_WITH_AUTHORIZER_ID>"
    ```

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
    HTTP_API_GATEWAY_ENDPOINT=https://<REPLACE_ME_WITH_API_ID>.execute-api.<REPLACE_ME_WITH_AWS_REGION>.amazonaws.com?channelName=aws_kvs_test_channel
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
4. ```aws lambda delete-function --function-name "kvs_sigv4_URL_generator_custom_authorizer"```
5. ```aws iam delete-role-policy --role-name "lambda-kvs_sigv4_URL_generator-role" --policy-name "lambda-kvs_sigv4_URL_generator-policy"```
6. ```rm lambda_kvs_sigv4_URL_generator_policy.json```
7. ```aws iam delete-role --role-name "lambda-kvs_sigv4_URL_generator-role"```
8. ```aws lambda delete-function --function-name "kvs_sigv4_URL_generator"```
9. ```aws apigatewayv2 delete-api --api-id "<REPLACE_ME_WITH_API_ID>"```
10.  ```aws kinesisvideo delete-signaling-channel --channel-arn "<REPLACE_ME_WITH_ARN_FOR_aws_kvs_test_channel>"```
11. ```aws dynamodb delete-table --table-name Users```

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

