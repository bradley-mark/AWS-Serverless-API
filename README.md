# AWS-Serverless-API

High Level Design

An Amazon API Gateway is a collection of resources and methods. For this tutorial, you create one resource (DynamoDBManager) and define one method (POST) on it. The method is backed by a Lambda function (LambdaFunctionOverHttps). That is, when you call the API through an HTTPS endpoint, Amazon API Gateway invokes the Lambda function.

The POST method on the DynamoDBManager resource supports the following DynamoDB operations:

Create, update, and delete an item.
Read an item.
Scan an item.
Other operations (echo, ping), not related to DynamoDB, that you can use for testing.
The request payload you send in the POST request identifies the DynamoDB operation and provides necessary data. For example:

The following is a sample request payload for a DynamoDB create item operation:

    {
        "operation": "create",
        "tableName": "lambda-apigateway",
        "payload": {
            "Item": {
                "id": "1",
                "name": "Bob"
            }
        }
    }

The following is a sample request payload for a DynamoDB read item operation:

    {
        "operation": "read",
        "tableName": "lambda-apigateway",
        "payload": {
            "Key": {
                "id": "1"
            }
        }
    }
    
# Setup

**Create Lambda IAM Role**

Create the execution role that gives your function permission to access AWS resources.

1. Open the IAM console **https://console.aws.amazon.com/iamv2**
2. In the navigation pane, choose **Roles**
3. Choose **Create role**
4. **Select trusted entity** - **AWS service** **Lambda** **Next**
5. Choose **Create policy** for custom policy with permission to DynamoDB and CloudWatch Logs
 
This custom policy has the permissions that the function needs to write data to DynamoDB and upload logs

    {
    "Version": "2012-10-17",
    "Statement": [
    {
      "Sid": "Stmt1428341300017",
      "Action": [
        "dynamodb:DeleteItem",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:UpdateItem"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Sid": "",
      "Resource": "*",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Effect": "Allow"
    }
    ]    
    }
    
6. **Next tags**  **Next Review**  Name **lambda-apigateway-policy**
7. Choose **Create policy**

#Create Lambda Function

