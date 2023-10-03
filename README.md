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
5. The client will connect to the pre-signed signaling channel URL as a WebSocket based connection 

The illustration below details what this solution will look like once fully implemented.

<img src="./assets/Solution%20Overview.png" />

<br /> 

### Prerequisites
To follow through this repository, you will need an <a href="https://console.aws.amazon.com/" >AWS account</a>, an <a href="https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/" >Amazon Kinesis Video Streams supported region</a>, permissions to create <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html" > AWS Identity and Access Management (IAM) roles and policies</a>, create <a href="https://docs.aws.amazon.com/lambda/latest/dg/getting-started.html#getting-started-create-function"> AWS Lambda Functions</a>, create <a href="https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-http-tutorials.html"> HTTP APIs in Amazon API Gateway</a>, create <a href="https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/getting-started-step-1.html"> tables in Amazon DynamoDB</a>, create <a href="https://docs.aws.amazon.com/kinesisvideostreams-webrtc-dg/latest/devguide/gs-createchannel.html"> signaling channels in Amazon Kinesis Video Streams</a> and access to the <a href="https://aws.amazon.com/cli/">AWS CLI</a>. We also assume you have familiar with the basics of Linux bash commands.


### Step 1: Create the AWS Lambda Authorizer function (AWS CLI)
1. Create your Amazon SNS topic by issuing the <b>create-topic</b> command
    ```   
    aws sns create-topic --name "aws_iot_core_logging_levels_demo_topic"
    ```
## Cleaning up
Be sure to remove the resources created in this repository to avoid charges. Run the following commands to delete these resources:

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

