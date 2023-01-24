# AWS-Serverless-API

High Level Design

An Amazon API Gateway is a collection of resources and methods. We will create one resource (DynamoDBManager) and define one method (POST) on it. The method is backed by a Lambda function (LambdaFunctionOverHttps). When you call the API through an HTTPS endpoint, Amazon API Gateway invokes the Lambda function.

The POST method on the DynamoDBManager resource supports the following DynamoDB operations:

- Create, update, and delete an item.
- Read an item.
- Scan an item.
- Other operations (echo, ping), not related to DynamoDB, that you can use for testing.

The request payload you send in the POST request identifies the DynamoDB operation and provides necessary data. 

For example:
The following is a sample request payload for a DynamoDB create item operation:

```yaml
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
```
The following is a sample request payload for a DynamoDB read item operation:

```yaml
{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1"
        }
    }
}
```

# Setup

**Create Lambda IAM Policy**

Create the execution role that gives your function permission to access AWS resources.

1. Open the IAM console **https://console.aws.amazon.com/iamv2**
2. In the navigation pane, choose **Policies**
3. Choose **Create policy** for custom policy with permission to DynamoDB and CloudWatch Logs
 
This custom policy has the permissions that the function needs to write data to DynamoDB and upload logs
```yaml
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
```    
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

![image](https://user-images.githubusercontent.com/91480603/214173895-0f91c26e-ed28-484b-a921-8fa5c207c0a9.png)

3. Select **Author from scratch**
4. **Basic information** Function name **LambdaFunctionOverHttps** Runtime **Python 3.7** Architecture **x86_64**
5. Permissions **Use an existing role** and select **lambda-apigateway-role**

![image](https://user-images.githubusercontent.com/91480603/214162587-281cfe86-75c4-4b9b-ab92-0e8ced4bdd6e.png)

6. Choose **Create function**

7. Replace **Code** **Code source** with the following code snippet and click **Deploy**

**Example Python Code**
```python
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
```
![image](https://user-images.githubusercontent.com/91480603/214162709-1a63cf2d-4987-44a8-9e19-2a36e92066fa.png)

# Test Lambda Function

We haven't created DynamoDB and the API yet, so we'll do a sample echo operation. The function should output whatever input we pass.

1. Choose **Test**
2. **Create new event** Event name **echotest**
3. Copy/paste JSON
```yaml
{
    "operation": "echo",
    "payload": {
        "somekey1": "somevalue1",
        "somekey2": "somevalue2"
    }
}
```
![image](https://user-images.githubusercontent.com/91480603/214163079-427d44da-7b43-4aed-acc7-a15f97d2372b.png)

4. Click **Save**
5. Click **Test**

![image](https://user-images.githubusercontent.com/91480603/214163190-e85c4f21-6a01-404c-9fdc-fea3a3c5429f.png)

Now we are ready to create DynamoDB and an API using Lambda as the backend

# Creating DynamoDB Table

1. Open the DynamoDB console **https://console.aws.amazon.com/dynamodbv2/home**
2. Choose **Create table**
3. Table name - **lambda-apigateway** 
4. Partition key - **id** - **String**
5. Choose **Create table**

![image](https://user-images.githubusercontent.com/91480603/214163247-34949982-ddc5-4695-b094-adaad7fb7dca.png)

# Create API

1.  Open the API Gateway console **https://console.aws.amazon.com/apigateway/main**
2.  APIs choose **Rest API** - **Build**

![image](https://user-images.githubusercontent.com/91480603/214163316-2cc8e5c1-30ba-4fb6-afbc-7e8b4c7b1bfb.png)

3.  Choose the protocol **REST**
4.  Create new API **New API**
5.  Settings API name **DynamoDBOperations**

![image](https://user-images.githubusercontent.com/91480603/214163404-9fce5a86-59e2-485a-8601-270ba126640d.png)

6.  Choose **Create API**
7.  Each API is collection of resources and methods that are integrated with backend HTTP endpoints, Lambda functions, or other AWS services. Typically, API resources are organized in a resource tree according to the application logic. At this time you only have the root resource, but let's add a resource next
Choose **Actions** and **Create Resource**

![image](https://user-images.githubusercontent.com/91480603/214163536-45c558b2-a0a4-4693-8ab4-210b6f4c67e0.png)

8.  Create Resource Name **DynamoDBManager** and Resouce Path will get auto-populated
9.  Select **Create Resource**

![image](https://user-images.githubusercontent.com/91480603/214163681-e497ff98-a016-42c4-a583-7f2244ce6504.png)

Create a POST Method for the API with dynamodbmanager resource selected
10.  Choose **Actions** and **Create Method**

![image](https://user-images.githubusercontent.com/91480603/214163725-ff95a958-eee4-4c72-8fd7-278db0c29832.png)

11.  Select **POST** from the drop down and click checkmark

![image](https://user-images.githubusercontent.com/91480603/214163767-1e7be4e6-3278-4278-8d40-759e9fa117d6.png)

The integration will come up automatically with **Lambda Function** option selected.
12. Select **LambdaFunctionOverHttps** function that we created earlier and select **Save**

![image](https://user-images.githubusercontent.com/91480603/214163851-e98ff2f6-6429-49ad-94c3-38111d119811.png)

13. Add Permission to Lambda Function - **OK**

![image](https://user-images.githubusercontent.com/91480603/214163900-ae224591-18f1-4425-b52f-7931584e14a2.png)

The API-Lambda integration is complete!

![image](https://user-images.githubusercontent.com/91480603/214163996-334f4064-a4ca-4d3f-b746-c3530231ed32.png)

# Deploy the API

1. Select **Actions**, select **Deploy API**

![image](https://user-images.githubusercontent.com/91480603/214164148-cb36973e-50ce-478e-a683-668281c2e31f.png)

2. Select Deployment Stage <New Stage> - Stage name **Prod**
3. Choose **Deploy**

![image](https://user-images.githubusercontent.com/91480603/214164237-63252062-9922-4a62-8fef-096f908c5867.png)

We are ready to run the solution. To invoke the API endpoint we need the endpoint URL
    
Navigate and expand the **Stages** tree and select POST to view the endpoint URL
    
![image](https://user-images.githubusercontent.com/91480603/214164300-6063245e-e4d6-4493-9bba-552ce06ed3d4.png)

# Running the solution
    
1. The Lambda function supports using the create operation to create an item in your DynamoDB table.
    
   To request this operation, use the following JSON:
```yaml
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234ABCD",
            "number": 5
        }
    }
}
```
    
 2. To execute the API from local machine, we are going to use Postman and/or Curl command.
    
To run from Postman, select **POST**, paste the API invoke URL
    
Under **Body** select **raw** and paste the JSON above
    
Choose **Send**
    
![image](https://user-images.githubusercontent.com/91480603/214174794-91295db9-8368-4254-8e57-2cfe82e7ac6e.png)

API should execute and return **HTTPStatusCode 200**
    
![image](https://user-images.githubusercontent.com/91480603/214174822-573e4e37-c0c3-4a13-9b97-0b939e998c91.png)

To run this from terminal using Curl

       $ curl -X POST -d "{\"operation\":\"create\",\"tableName\":\"lambda-apigateway\",\"payload\":{\"Item\":{\"id\":\"1\",\"name\":\"Bob\"}}}" https://ejnggdo47b.execute-api.us-east-1.amazonaws.com/Prod/dynamodbmanager
    
3. To validate that the item is indeed inserted into DynamoDB table, go to Dynamo console, select **lambda-apigateway** table, select **Explore items** tab, and the newly inserted item should be displayed.
    
![image](https://user-images.githubusercontent.com/91480603/214174959-afb96c5f-27b8-43c1-9ad9-e7d6f22d3064.png)

4. To get all the inserted items from the table, we can use the **list** operation of Lambda using the same API. Pass the following JSON to the API, and it will return all the items from the Dynamo table
```yaml   
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
}
```
![image](https://user-images.githubusercontent.com/91480603/214175037-e4e11b7e-fd9a-4295-a792-2e2a9d9644a7.png)

![image](https://user-images.githubusercontent.com/91480603/214175068-ddfff1f0-58c5-4806-b7e7-bf104c0a5e47.png)

We have successfully created a serverless API using API Gateway, Lambda, and DynamoDB!
    
# Cleanup
    
To delete the table, from DynamoDB console, select the table **lambda-apigateway** and click **Delete table**
    
![image](https://user-images.githubusercontent.com/91480603/214175104-c16ffda5-153c-44cd-b820-954a4591803e.png)

To delete the Lambda, from the Lambda console, select lambda **LambdaFunctionOverHttps** click **Actions** then click **Delete**
    
![image](https://user-images.githubusercontent.com/91480603/214175133-29656a1c-682a-4860-9414-0d4f47d3e541.png)

To delete the API, from the API Gateway console, under APIs, select **DynamoDBOperations** click **Actions** then **Delete**
    
![image](https://user-images.githubusercontent.com/91480603/214175162-2b62a8ad-64c6-43fd-b313-dce2ab8248fe.png)

    
