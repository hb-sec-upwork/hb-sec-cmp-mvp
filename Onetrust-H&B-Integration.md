# Content Management MVP - Onetrack Integration

This document aims to explain the process to setup fetching "Marketing Subject Consent" data from OneTrack in H&B AWS

## Table of Contents

- [Pushing Consent Subjects from Ontrack to H&B](#Pushing-Consent-Subjects-from-Ontrack-to-H&B)
- [Pulling Consent Subjects from Ontrack into H&B](#Pulling-Consent-Subjects-from-Ontrack-into-H&B)

## Pushing Consent Subjects from Ontrack to H&B

The process followed to push subject data from Onetrust to H&B was as below 

### H&B AWS Infrastructure Setup

 **Create a DynamoDB table in H&B AWS**
  - This involves creating a DynamoDB table (e.g ontrust-subjects) in H&B AWS
  - The table should have a primary key called 'eventId' of type string (sort key not needed for now)

![image](https://github.com/abhicoderx/for_patrick/assets/54671816/5f4bc902-ffea-4e86-b35d-1023ccdfd9fc)

    
**Create a Lambda function in H&B AWS**
  - This involves creating a lambda function (e.g post-ontrust-subjects-lambda) to insert received event into the above DynamoDB table
  - This lambda will be invoked when the api gatway endoint (as stated below) is invoked.
  - It will insert the received event into to ontrust-subject DynamoDB table.

Source code for this Lmabda function is ias below

```
import os
import json
import boto3
from decimal import Decimal
from botocore.exceptions import ClientError

aws_region = 'eu-west-1'
table_name = 'onetrust_test'
dynamodb = boto3.resource('dynamodb', region_name=aws_region)
table = dynamodb.Table(table_name)

def decimal_default(obj):
    if isinstance(obj, Decimal):
        return int(obj)
    raise TypeError

def lambda_handler(event, context):
    print(f"Event Received - {json.dumps(event)}")
    response_text = event["body"]
    # Remove extra slashes and newlines
    response_text = response_text.replace('\\n', '').replace('\\\\', '\\').replace('\\', '')
    print(f"EventCorrected request - {response_text}")
    #response_text = json.loads(response_text)
    try:
        db_response = table.put_item(
            Item=json.loads(response_text)
        )
        print("PutItem succeeded:", db_response)
    
    except ClientError as e:
        print("Error putting item:", e)
        return {
            'statusCode': 200,
            'body': f"Error in request, got {e}"
        }            

    return {
                'statusCode': 200,
                'body': "Succeeded"
            }
```

    
- **Create an API Gateway Endpoint in H&B AWS**
  - This involves creating an api gateway endpoint in H&B AWS
  - The API Gateway endpoint must have a route called '/v1/api'
  - Create an API key to secure access to the api (invokers to pass this in the 'x-api-key' HTTP header
  - Create a Lambda Integration integration and plug the above lambda (post-ontrust-subjects-lambda) behind the integration
  - Create a Stage and Deploy the API .
  - Note the endpoint of the API Gateway to configure in Onetrust
  
![image](https://github.com/abhicoderx/for_patrick/assets/54671816/0dc6ccc7-b692-4ade-b793-907dba354560)


The endpoint currently available and in use in H&B AWS and configured in OneTrust is https://jekozmhnha.execute-api.us-east-1.amazonaws.com/dev/v1/api

### Onetrust Configuration

To configure the above API Gateway Integration in Onetrust, below is the process

- Go to 

### Testing

## Pulling Consent Subjects from Ontrack into H&B

[List the key features of your project.]

