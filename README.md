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

**Create Lambda IAM Policy**

Create the execution role that gives your function permission to access AWS resources.

1. Open the IAM console **https://console.aws.amazon.com/iamv2**
2. In the navigation pane, choose **Policies**
3. Choose **Create policy** for custom policy with permission to DynamoDB and CloudWatch Logs
 
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
    
7. **Next tags**  **Next Review**  Name **lambda-apigateway-policy**
8. Choose **Create policy**

**Create Lambda IAM Role**

1. In the navigation pane, choose **Roles**
2. Choose **Create role**
3. Choose **AWS service** and **Lambda** - **Next**
4. Select Permissions policy **lambda-apigateway-policy** - **Next**
5. Enter Role name **lambda-apigateway-role**
6. Choose **Create role**

# Create Lambda Function

1. Open the Lambda console **https://console.aws.amazon.com/lambda/home**
2. Choose **Create a function**
3. Select **Author from scratch**
4. **Basic information** Function name **LambdaFunctionOverHttps** Runtime **Python 3.7** Architecture **x86_64**
5. Permissions **Use an existing role** and select **lambda-apigateway-role**
6. Choose **Create function**
7. Replace **Code** **Code source** with the following code snippet and click **Deploy**

        from __future__ import print_function

        import boto3
        import json
      
        print('Loading function')
      
      
        def lambda_handler(event, context):
            '''Provide an event that contains the following keys:
      
              - operation: one of the operations in the operations dict below
              - tableName: required for operations that interact with DynamoDB
              - payload: a parameter to pass to the operation being performed
            '''
            #print("Received event: " + json.dumps(event, indent=2))

        operation = event['operation']

        if 'tableName' in event:
            dynamo = boto3.resource('dynamodb').Table(event['tableName'])

        operations = {
            'create': lambda x: dynamo.put_item(**x),
            'read': lambda x: dynamo.get_item(**x),
            'update': lambda x: dynamo.update_item(**x),
            'delete': lambda x: dynamo.delete_item(**x),
            'list': lambda x: dynamo.scan(**x),
            'echo': lambda x: x,
            'ping': lambda x: 'pong'
        }

        if operation in operations:
            return operations[operation](event.get('payload'))
        else:
            raise ValueError('Unrecognized operation "{}"'.format(operation))
        
  
# Test Lambda Function

We haven't created DynamoDB and the API yet, so we'll do a sample echo operation. The function should output whatever input we pass.

1. Choose **Test**
2. **Create new event** Event name **echotest**
3. Copy/paste JSON

      {
          "operation": "echo",
          "payload": {
              "somekey1": "somevalue1",
              "somekey2": "somevalue2"
          }
      }

4. Click **Save**
5. Click **Test**

Now we are ready to creeate DynamoDB and an API using Lambda as the backend

# Creating DynamoDB Table

