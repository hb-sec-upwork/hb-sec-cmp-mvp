# Content Management MVP - OneTrust Integration

This document aims to explain the process for fetching "Marketing Subject Consent" data from OneTrust in H&B AWS

## Table of Contents

- [Pushing Consent Subjects from OneTrust to H&B](#pushing-consent-subjects-from-onetrust-to-hb)
- [Pulling Consent Subjects from OneTrust into H&B](#pulling-consent-subjects-from-onetrust-into-hb)


# Pushing Consent Subjects from OneTrust to H&B

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
  - Create an API Key to access the API
  - Note the endpoint of the API Gateway and the API Key to configure in Onetrust
  
![image](https://github.com/abhicoderx/for_patrick/assets/54671816/0dc6ccc7-b692-4ade-b793-907dba354560)


The endpoint currently available and in use in H&B AWS and configured in OneTrust is https://jekozmhnha.execute-api.us-east-1.amazonaws.com/dev/v1/api

### Onetrust Configuration


Below is the configuration needed in Onetrust to create the API Gateway Integration
- Go to 'Integrations'  in the Onetrust console
  ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/2ab0715a-d43d-46f8-8022-8c4c3ed6ba07)

**Create Credentials**

- Create Credentials of Type API Gateway [From the 'Credentials' link in the menu]
- Put the API Gateway URL as created above in the HostName
- Keep Protocol as HTTPS
- Add a Header called 'x-api-key' and put the value as the API Key of the api gateway created.
- Save the Credentials.
- Test the Credentials by clicking on the 'Test Credentials' button. It should be Connected if all goes well
  ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/51b42ad7-f017-49e4-bbb2-1844e8011fa8)

 **Create Connection**
- Create a connection [From the 'Connections' link in the menu]
- Chose a Name and System as 'API Gateway'
- In the Workflow Builder
- Create A Trigger
  - Add a Onetrust Trigger by navigating to Consent -> Chose needed notificatin on H&B AWS
  - 
    ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/af7c82c0-fce3-4733-aa45-afac11c4a8a2)

    ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/bbb227ff-c6d7-441f-ae4d-7d47a5fa99a6)

- Create An Action
  - Add an Action of Type POST Request
  - In Credentials section use the API Gateway credentials created above
  - For Body, use below sample (this is customizable and this is the Event body that will be received by H&B, relevant fields may be included)

```
{
  "eventId": \"${(event.eventId)!}\",
  "name" : \"${(event.payload.collectionPointIntegrationEventPayload.name)!}\",
  "eventType": \"${(event.eventType)!}\",
  "eventTime": \"${(event.eventTime)!}\",
  "id": \"${(event.payload.collectionPointIntegrationEventPayload.id)!}\",
  "consentType": \"${(event.payload.collectionPointIntegrationEventPayload.consentType)!}\",
  "name": \"${(event.payload.collectionPointIntegrationEventPayload.name)!}\",
  "subjectIdentifier": \"${(event.payload.collectionPointIntegrationEventPayload.subjectIdentifier)!}\",
  "identifierDataElementGuid": \"${(event.payload.collectionPointIntegrationEventPayload.identifierDataElement.guid)!}\",
  "email": \"${(event.userDetails.email)!}\"
}
```
  - Save the Connection
  - Activate the Connection
![image](https://github.com/abhicoderx/for_patrick/assets/54671816/bd26f69b-9afe-41a2-baf0-d5a9b6664d34)

This finishes the Integration creation in OneTrust

### Testing

Once created, to test the Integration, below needs to be done
- In the Connection created above, Click "Test Workflow"
  ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/c1142d02-599a-4c30-ae38-66c421da411e)

- Click on the 'Create new Event Data' button
- Add an Event, Save it and Click Test
- If all goes well, the Test should succeed
  ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/8aa0b26b-3721-4c49-aa4d-68722b2c4baa)

- Below is a sample Event Data to use

```
{
  "eventId" : "fbb3b643-40f0-468d-84df-c140f8fb3d9e",
  "eventType" : "3040",
  "payload" : {
    "collectionPointPublishedTime" : "2021-12-06T19:28:12.87789",
    "collectionPointIntegrationEventPayload" : {
      "id" : "6621676c-b676-4c27-93a8-ea073e1935dc",
      "version" : 17,
      "name" : "RP API Test",
      "status" : "ACTIVE",
      "collectionPointType" : "API",
      "consentType" : "CONDITIONALTRIGGER",
      "description" : "RP API Test",
      "subjectIdentifier" : "Email",
      "identifierDataElement" : {
        "guid" : "21e0de87-d6ef-4b39-a54b-6d877698e3f6",
        "numberOfLanguages" : 0,
        "canEdit" : false,
        "dataElementFields" : {
          "dataElementType" : "USER_INPUT",
          "displayAs" : "NONE",
          "dataElementOptions" : "string_value"
        },
        "label" : "Email",
        "isIdentifier" : false,
        "collectionPoints" : 0,
        "languages" : "string_value"
      },
      "receiptCount" : 0,
      "organizationId" : "bc6f7c41-51c2-423e-82ba-443a9783a648",
      "doubleOptIn" : false,
      "noConsentTransactions" : false,
      "language" : "en-us",
      "hostedSDK" : false,
      "showWarning" : true,
      "sendConsentEmail" : false,
      "warningReasons" : [ "NO_ACTIVITY", "NO_ACTIVITY", "NO_ACTIVITY", "NO_ACTIVITY", "NO_ACTIVITY" ],
      "consentIntegration" : true,
      "enableNewConsentIntegration" : true,
      "isAuthenticationRequired" : false,
      "createdBy" : "6EDA74E0-60D9-4B2A-935E-DD86F21C97D3",
      "lastModifiedBy" : "6eda74e0-60d9-4b2a-935e-dd86f21c97d3",
      "reconfirmActivePurpose" : false,
      "publishedBy" : "6eda74e0-60d9-4b2a-935e-dd86f21c97d3",
      "dataElements" : "string_value",
      "jwtToken" : {
        "token" : "string_value"
      },
      "noticesWithVersions" : "string_value",
      "purposes" : [ {
        "id" : "f674ccd5-c7e5-4c7d-bfa6-818b4a397872",
        "label" : "RP Test Purpose",
        "description" : "RP Test Purpose",
        "status" : "ACTIVE",
        "version" : 3,
        "consentLifeSpan" : 0,
        "purposeType" : "STANDARD",
        "createdBy" : "6EDA74E0-60D9-4B2A-935E-DD86F21C97D3",
        "createdDate" : "2021-11-18T17:09:36.693Z",
        "lastModifiedDate" : "2021-12-06T08:44:30.297Z",
        "publishedBy" : "6EDA74E0-60D9-4B2A-935E-DD86F21C97D3",
        "publishedDate" : "2021-12-06T08:44:30.250Z",
        "customPreferences" : [ {
          "id" : "8a513c05-8d08-4b57-b622-3c4cb53897d7",
          "name" : "cp 013019 2",
          "displayAs" : "BUTTONS",
          "customPreferenceOptions" : [ {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          } ],
          "languages" : [ {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          } ]
        }, {
          "id" : "8a513c05-8d08-4b57-b622-3c4cb53897d7",
          "name" : "cp 013019 2",
          "displayAs" : "BUTTONS",
          "customPreferenceOptions" : [ {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          } ],
          "languages" : [ {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          } ]
        }, {
          "id" : "8a513c05-8d08-4b57-b622-3c4cb53897d7",
          "name" : "cp 013019 2",
          "displayAs" : "BUTTONS",
          "customPreferenceOptions" : [ {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          } ],
          "languages" : [ {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          } ]
        }, {
          "id" : "8a513c05-8d08-4b57-b622-3c4cb53897d7",
          "name" : "cp 013019 2",
          "displayAs" : "BUTTONS",
          "customPreferenceOptions" : [ {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          } ],
          "languages" : [ {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          } ]
        }, {
          "id" : "8a513c05-8d08-4b57-b622-3c4cb53897d7",
          "name" : "cp 013019 2",
          "displayAs" : "BUTTONS",
          "customPreferenceOptions" : [ {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          } ],
          "languages" : [ {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          } ]
        }, {
          "id" : "8a513c05-8d08-4b57-b622-3c4cb53897d7",
          "name" : "cp 013019 2",
          "displayAs" : "BUTTONS",
          "customPreferenceOptions" : [ {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          } ],
          "languages" : [ {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          } ]
        } ],
        "topics" : [ {
          "id" : "ea2f6534-d5f5-4028-b285-ad85b3c45f10",
          "name" : "!!!!!Topic_3325_1",
          "integrationKey" : "RP-Test-Purpose_!!!!!Topic_3325_1",
          "languages" : "string_value",
          "canDelete" : true
        }, {
          "id" : "ea2f6534-d5f5-4028-b285-ad85b3c45f10",
          "name" : "!!!!!Topic_3325_1",
          "integrationKey" : "RP-Test-Purpose_!!!!!Topic_3325_1",
          "languages" : "string_value",
          "canDelete" : true
        }, {
          "id" : "ea2f6534-d5f5-4028-b285-ad85b3c45f10",
          "name" : "!!!!!Topic_3325_1",
          "integrationKey" : "RP-Test-Purpose_!!!!!Topic_3325_1",
          "languages" : "string_value",
          "canDelete" : true
        }, {
          "id" : "ea2f6534-d5f5-4028-b285-ad85b3c45f10",
          "name" : "!!!!!Topic_3325_1",
          "integrationKey" : "RP-Test-Purpose_!!!!!Topic_3325_1",
          "languages" : "string_value",
          "canDelete" : true
        }, {
          "id" : "ea2f6534-d5f5-4028-b285-ad85b3c45f10",
          "name" : "!!!!!Topic_3325_1",
          "integrationKey" : "RP-Test-Purpose_!!!!!Topic_3325_1",
          "languages" : "string_value",
          "canDelete" : true
        } ],
        "organizations" : [ "9048ec86-df69-4191-868a-cb31e11eb15d", "9048ec86-df69-4191-868a-cb31e11eb15d", "9048ec86-df69-4191-868a-cb31e11eb15d", "9048ec86-df69-4191-868a-cb31e11eb15d", "9048ec86-df69-4191-868a-cb31e11eb15d" ],
        "languages" : [ {
          "name" : "RP Test Purpose",
          "description" : "RP Test Purpose",
          "language" : "en-us",
          "default" : true
        }, {
          "name" : "RP Test Purpose",
          "description" : "RP Test Purpose",
          "language" : "en-us",
          "default" : true
        } ],
        "attributes" : { },
        "receiptInclusionAttributes" : { }
      }, {
        "id" : "f674ccd5-c7e5-4c7d-bfa6-818b4a397872",
        "label" : "RP Test Purpose",
        "description" : "RP Test Purpose",
        "status" : "ACTIVE",
        "version" : 3,
        "consentLifeSpan" : 0,
        "purposeType" : "STANDARD",
        "createdBy" : "6EDA74E0-60D9-4B2A-935E-DD86F21C97D3",
        "publishedBy" : "6EDA74E0-60D9-4B2A-935E-DD86F21C97D3",
        "customPreferences" : [ {
          "id" : "8a513c05-8d08-4b57-b622-3c4cb53897d7",
          "name" : "cp 013019 2",
          "displayAs" : "BUTTONS",
          "customPreferenceOptions" : [ {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          } ],
          "languages" : [ {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          } ]
        }, {
          "id" : "8a513c05-8d08-4b57-b622-3c4cb53897d7",
          "name" : "cp 013019 2",
          "displayAs" : "BUTTONS",
          "customPreferenceOptions" : [ {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          } ],
          "languages" : [ {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          } ]
        }, {
          "id" : "8a513c05-8d08-4b57-b622-3c4cb53897d7",
          "name" : "cp 013019 2",
          "displayAs" : "BUTTONS",
          "customPreferenceOptions" : [ {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          }, {
            "id" : "3b4165d1-af46-486b-b9b0-8cc924d42be6",
            "label" : "car",
            "order" : 0,
            "isDefault" : false
          } ],
          "languages" : [ {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          }, {
            "name" : "cp 013019 2",
            "description" : "test",
            "language" : "en-us",
            "default" : true,
            "options" : "string_value"
          } ]
        } ],
        "topics" : [ {
          "id" : "ea2f6534-d5f5-4028-b285-ad85b3c45f10",
          "name" : "!!!!!Topic_3325_1",
          "integrationKey" : "RP-Test-Purpose_!!!!!Topic_3325_1",
          "languages" : "string_value",
          "canDelete" : true
        }, {
          "id" : "ea2f6534-d5f5-4028-b285-ad85b3c45f10",
          "name" : "!!!!!Topic_3325_1",
          "integrationKey" : "RP-Test-Purpose_!!!!!Topic_3325_1",
          "languages" : "string_value",
          "canDelete" : true
        }, {
          "id" : "ea2f6534-d5f5-4028-b285-ad85b3c45f10",
          "name" : "!!!!!Topic_3325_1",
          "integrationKey" : "RP-Test-Purpose_!!!!!Topic_3325_1",
          "languages" : "string_value",
          "canDelete" : true
        }, {
          "id" : "ea2f6534-d5f5-4028-b285-ad85b3c45f10",
          "name" : "!!!!!Topic_3325_1",
          "integrationKey" : "RP-Test-Purpose_!!!!!Topic_3325_1",
          "languages" : "string_value",
          "canDelete" : true
        } ],
        "organizations" : [ "9048ec86-df69-4191-868a-cb31e11eb15d", "9048ec86-df69-4191-868a-cb31e11eb15d", "9048ec86-df69-4191-868a-cb31e11eb15d" ],
        "languages" : [ {
          "name" : "RP Test Purpose",
          "description" : "RP Test Purpose",
          "language" : "en-us",
          "default" : true
        }, {
          "name" : "RP Test Purpose",
          "description" : "RP Test Purpose",
          "language" : "en-us",
          "default" : true
        }, {
          "name" : "RP Test Purpose",
          "description" : "RP Test Purpose",
          "language" : "en-us",
          "default" : true
        }, {
          "name" : "RP Test Purpose",
          "description" : "RP Test Purpose",
          "language" : "en-us",
          "default" : true
        }, {
          "name" : "RP Test Purpose",
          "description" : "RP Test Purpose",
          "language" : "en-us",
          "default" : true
        } ],
        "attributes" : { },
        "receiptInclusionAttributes" : { }
      } ],
      "canCreateNewVersion" : false,
      "newSdkIntegrationEnabled" : false,
      "languages" : [ "en-us", "en-us", "en-us", "en-us", "en-us" ],
      "preferenceCenter" : { },
      "attributes" : {
        "sp_test4" : [ "textt publish v17", "textt publish v17" ]
      }
    }
  },
  "integrationId" : null,
  "workflowId" : "8d07373e-79cc-4bf3-b0df-43461b993ffe",
  "messageKey" : "6621676c-b676-4c27-93a8-ea073e1935dc:2021-12-06T19:28:12.877890",
  "messageSequenceNumber" : 1638818892916,
  "deDuplicationStrategy" : "PROCESS_ALL",
  "reprocessCount" : null,
  "isBulk" : false,
  "headers" : null,
  "parentWorkflowId" : null,
  "throttlingDelayMillis" : null,
  "blobStoreLocation" : "18967/8d07373e-79cc-4bf3-b0df-43461b993ffe/fbb3b643-40f0-468d-84df-c140f8fb3d9e",
  "blobStoreFileSize" : 4808,
  "isCompressed" : false,
  "referenceMessage" : false,
  "isManualProcess" : null,
  "validationErrors" : null
}
```

  
### Validation

To validate that the Event pushed above is now iwith H&B,
- Go to the ontrust-subjects DynamoDB table
- Run the query to Check items
- A new item (with the Event pushed in the test above) should appear in the table as below
  ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/e9a813a8-f5ad-4c9f-b13b-7c524dfc797d)

  

# Pulling Consent Subjects from OneTrust into H&B

The process followed to pull subject data periodically from Onetrust to H&B was as below 

### H&B AWS Infrastructure Setup

 **Create a DynamoDB table in H&B AWS**
  - This involves creating a DynamoDB table (e.g ontrust-subjects) in H&B AWS
  - The table should have a primary key called 'eventId' of type string (sort key not needed for now)

![image](https://github.com/abhicoderx/for_patrick/assets/54671816/5f4bc902-ffea-4e86-b35d-1023ccdfd9fc)

    
**Create a Lambda function in H&B AWS**
  - This involves creating a lambda function (e.g post-ontrust-subjects-pull-lambda) to make a call to OneTrust SDK to fetch and insert subjects data into the above DynamoDB table
  - This lambda will be invoked periodically with an EventBridge rule.
  - It will insert the received event into to ontrust-subject DynamoDB table. (The process to create ONETRUST_API_KEY environment variable used in the lambda is discussed in the Onetrust-Configuration section below)

Source code for this Lmabda function is ias below

```
import os
import requests
import json
import boto3
from decimal import Decimal
from botocore.exceptions import ClientError

aws_region = 'us-east-1'
table_name = 'onetrust_test'
dynamodb = boto3.resource('dynamodb', region_name=aws_region)
table = dynamodb.Table(table_name)

def decimal_default(obj):
    if isinstance(obj, Decimal):
        return int(obj)
    raise TypeError

def lambda_handler(event, context):
    # Define the API URL
    api_url = 'https://trial.onetrust.com/api/consentmanager/v1/datasubjects/profiles'

    # Retrieve the API key from environment variables
    api_key = os.environ.get('ONETRUST_API_KEY')

    # Check if the API key is available
    if not api_key:
        print("Error: ONETRUST_API_KEY not found in environment variables.")
        return {
            'statusCode': 500,
            'body': 'Error: ONETRUST_API_KEY not found in environment variables.'
        }

    # Define headers, including the Authorization header with the token
    headers = {
        'accept': 'application/json',
        'Authorization': f'Bearer {api_key}'
    }

    try:
        # Make the GET request
        response = requests.get(api_url, headers=headers)
        # Check if the request was successful (status code 200)
        if response.status_code == 200:
            # Print the response content
            response_text = response.text
            print(response_text)
            response_text = json.loads(response_text)
            for item in response_text["content"]:
                item["eventId"] = item["Id"]
                del item["Id"]
                try:
                    db_response = table.put_item(
                        Item=json.loads(json.dumps(item, default=decimal_default))
                    )
                    print("PutItem succeeded:", db_response)
                except ClientError as e:
                    print("Error putting item:", e)
            return {
                'statusCode': 200,
                'body': response.json()
            }            
        else:
            # Print an error message if the request was not successful
            print(f"Error: {response.status_code} - {response.text}")
            return {
                'statusCode': response.status_code,
                'body': f"Error: {response.status_code} - {response.text}"
            }
    except Exception as e:
        # Print an error message if an exception occurs
        print(f"Exception: {str(e)}")
        return {
            'statusCode': 500,
            'body': f"Exception: {str(e)}"
        }


```

**Create an EventBridge Rule in H&B AWS to invoke the above lambda function periodically**
    
![image](https://github.com/abhicoderx/for_patrick/assets/54671816/a7c31fc8-f663-4dd7-a825-558c482c8431)

![image](https://github.com/abhicoderx/for_patrick/assets/54671816/5796218b-5446-4b7c-8fcb-75283c7e30f1)


### Onetrust Configuration


Below is the configuration needed in Onetrust to create the API Key to use in the Lambda function to fetch subject data.
- Navigate to Settings (Global, Top Right on Console) -> Access Management -> Client Credentials
 ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/f4923db1-15c0-4fff-b985-9da6e7042b92)

- Click on API Keys -> Add an API Key -> Provide Lifetime -> Click Next -> Add Permissions Needed by the API Key (We need 'Consent') and Click Create
  ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/9cde14f8-0623-4782-b56a-1de86c50d119)

- Note the API Key generated.
- Put it in the post-ontrust-subjects-pull-lambda lambda function with the KEY ONETRUST_API_KEY
- Redeploy the lambda function

### Testing

- Enter new marketing consent data in the webform at https://hb-sec-upwork.github.io/hb-sec-cmp-mvp/
  ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/890ead51-4dda-43fb-ac9b-4e424c088972)

  ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/9158bb8a-af6b-4954-b9b5-561be73241ca)


- The new entry should be avaialable in the DynamoDB table after the period the eventbridge of scheculed to run (5 minutes currently)

  ![image](https://github.com/abhicoderx/for_patrick/assets/54671816/ead3ddf8-a59a-44d4-a17c-c02ec3a70d1e)
