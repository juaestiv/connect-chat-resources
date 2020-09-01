# connect-chat-resources

# Process for a Basic Chat Implementation
These are the steps that we will follow
* Create Chat Entry Point Flow
* Create a Lambda integration for identification of the customer
* Test the Chat simulating parameters 
* Add a Lexbot
* Create a React Customer Page.

# Create a chat entry point flow
Prerequisites: 
* Create two "Hours of operation" tables and call them "Sales and Billing" and "Customer Service"
* Create 3 queues and name them "Billing", "Customer Service" and "Sales" 

## Create the flow
Create the following flow or import the attached resources.  

![Image of Flow](https://github.com/juaestiv/connect-chat-resources/blob/master/images/ChatEntryPointFlow.png)

You can download the json file here:
[Flow to import](resources/ChatEntryPoint.json.zip)

You can use the import feature to import this flow. Just Create a new flow and click on the arrow on the left hand side near of the top to drop down the Import Flow option. The set up a name and save and publish the flow. Take into account that you must tuo unzip de file first. 


## Test the Flow
Open the Amazon Connect Console and navigate to the dasboard. Yoy can show/hide the guide in an hyperlink located near of the top on left hand side. 
When the guide is shown you can locate a "Test Chat" link on the first step. 

Clicking over it you can simulate Customer and Agent conversations on the same screen. 
Click over the link "Test Settings" to select the flow to test. 


![Image of Test Chat](https://github.com/juaestiv/connect-chat-resources/blob/master/images/TestingChat.png)

# Create Lambda integration for customer identification. 
## Create a DynamoDB Table
Open the AWS Console and navigate to DynamoDB. Select "Tables" from left hand tree and click over "Create a Table".

Create a table with the following properties
  |  
------------ | -------------
Table Name | customerLookup
Primary Key | phoneNum

* Name of the table: customerLookup
* Primary Key: phoneNum

Create an index:
* Primary Key: emailAddress

Create an Item in text format:

```json
{
    "accountNum" : "74235003",
    "addressCity" : "Everett",
    "addressStreet" : "Maple st",
    "addressZip" : "98201",
    "csmrAccount" : "725221",
    "firstName" : "Cameron",
    "lastName" : "Evans",
    "phoneNum" : "+14252105892",
    "streetNum" : "12423",
    "emailAddress": "youremail@address.com"
}
```

## Create a lambda function
From AWS Console look for Lambda service. 
* Click Create a Function. 
* Select "Author from scratch"
* Give it the name customerLookup
* Select Node.js 12,x as runtime
* Keep the remaining defaults

Replace the code with the following code:

```js
var AWS = require("aws-sdk");
var docClient = new AWS.DynamoDB.DocumentClient();

exports.handler = (event, context, callback) => {
  var CallerID = event.Details.Parameters.CallerID;
  var Channel = event.Details.ContactData.Channel;
  var email = event.Details.ContactData.Attributes.email;
  var ReturnResult = {};
  console.log("Amazon Connect event: " + JSON.stringify(event));
  
  //Voice Channel. Looking by PhoneNumber
  if (Channel == 'VOICE') {
    var params = {
      TableName: 'customerLookup',
      KeyConditionExpression: "phoneNum = :varNumber",
      ExpressionAttributeValues: { ":varNumber": CallerID }
    };
    QueryDynamoDB(params);
    

  //Chat Channel. Looking by email address
  } else if (Channel == 'CHAT') {
        if (email) {
        var params = {
                    TableName: 'customerLookup',
                    KeyConditionExpression: "emailAddress = :varEmail",
                    IndexName: "emailAddress-index",
                    ExpressionAttributeValues: 
                    {
                        ":varEmail": email
                    }
                };
                QueryDynamoDB(params);        
        } else {
            console.log("No email");
            callback(null, buildResponse(false));      
            }
    
    
    
  
    
  }
  
function QueryDynamoDB(params)
    {
        docClient.query(params, function(err, data) {
            
            if (err) {
                console.error("Unable to read item. Error JSON:", JSON.stringify(err, null, 2));

            } else {
                console.log("GetItem succeeded:", JSON.stringify(data, null, 2));
                if(data.Items.length == 1)
                {
                    var recordFound = "True";
                    var lastName = data.Items[0].lastName;
                    var firstName = data.Items[0].firstName;
                    var emailAddress = data.Items[0].emailAddress;
                    var accountNum = data.Items[0].accountNum;
                    var addressCity = data.Items[0].addressCity;
                    var addressStreet = data.Items[0].addressStreet;
                    var addressZip = data.Items[0].addressZip;
                    var streetNum = data.Items[0].streetNum;
                    var phoneNum = data.Items[0].phoneNum;
                    callback(null, buildResponse(true, recordFound, lastName, firstName, emailAddress, accountNum, addressCity, addressStreet, addressZip, streetNum, phoneNum));
                    ReturnResult.EmployeeID = data.Items[0].EmployeeID;
                    ReturnResult.EmployeeName = data.Items[0].EmployeeName;
                    ReturnResult.EmployeePIN = data.Items[0].EmployeePIN;
                }
            }
            

        });
    }
  
 
  
};


 
function buildResponse(isSuccess, recordFound, lastName, firstName, emailAddress, accountNum, addressCity, addressStreet, addressZip, streetNum, phoneNum) {
  if (isSuccess) {
    return {
      recordFound: recordFound,
      lastName: lastName,
      firstName: firstName,
      emailAddress:emailAddress,
      accountNum: accountNum,
      addressCity: addressCity,
      addressStreet: addressStreet,
      addressZip: addressZip,
      streetNum: streetNum,
      phoneNum: phoneNum,
      lambdaResult: "Success"
    };
  }
  else {
    console.log("Lambda returned error to Connect");
    return { lambdaResult: "Error" };
  }
}



```

## Edit the lambda role
* Look for the tab Permissions
* Click on the hyperlink to the lambda role
* Add Inline Policy

Add inline policies to the lambda role for:

Service -> DynamoDB
Actions -> List and Read
Resources-> Add ARN to the index and to the table. 

arn:aws:dynamodb:us-east-1:<account_number>:table/customerLookup
arn:aws:dynamodb:us-east-1:<account_number>:table/customerLookup/index/emailAddress-index

![Image of Lambda Role](https://github.com/juaestiv/connect-chat-resources/blob/master/images/LambdaRole.png)

Come back to the lambda function and create a test event

## Create a Chat Test Event

```json
{
    "Details": {
        "ContactData": {
            "Attributes": {
                "email": "youremail@address.com"
            },
            "Channel": "CHAT",
            "ContactId": "ef26d146-xxx-4e1e-a481-87fc64ce1112",
            "CustomerEndpoint": null,
            "Description": null,
            "InitialContactId": "ef26d146-xxx-4e1e-a481-87fc64ce1112",
            "InitiationMethod": "API",
            "InstanceARN": "arn:aws:connect:us-east-1:<account_number>:instance/xxxxxxxx-xxxx-xxxxx-xxx-xxx",
            "LanguageCode": "en-US",
            "MediaStreams": {
                "Customer": {
                    "Audio": {}
                }
            },
            "Name": null,
            "PreviousContactId": "ef26d146-xxxx-4e1e-a481-87fc64ce1112",
            "Queue": null,
            "References": {},
            "SystemEndpoint": null
        },
        "Parameters": {}
    },
    "Name": "ContactFlowEvent"
}

```

## Create a Call Event
```json
{
    "Details": {
        "ContactData": {
            "Attributes": {},
            "Channel": "VOICE",
            "ContactId": "8693452c-7fd1-462b-adb6-fcf18996aabb",
            "CustomerEndpoint": {
                "Address": "+34674870071",
                "Type": "TELEPHONE_NUMBER"
            },
            "Description": null,
            "InitialContactId": "8693452c-7fd1-462b-adb6-fcf18996aabb",
            "InitiationMethod": "INBOUND",
            "InstanceARN": "arn:aws:connect:us-east-1:150934812383:instance/0879ee04-aaaa-4668-8ef5-5ba3f60d0ca3",
            "LanguageCode": "en-US",
            "MediaStreams": {
                "Customer": {
                    "Audio": null
                }
            },
            "Name": null,
            "PreviousContactId": "8693452c-7fd1-462b-adb6-fcf18996aabb",
            "Queue": null,
            "References": {},
            "SystemEndpoint": {
                "Address": "+34902018610",
                "Type": "TELEPHONE_NUMBER"
            }
        },
        "Parameters": {
            "CallerID": "+34674870071"
        }
    },
    "Name": "ContactFlowEvent"
}
```
You can execute this test event and check if returns the appropiate parameters


## Edit the Flow to search the customer information using email as identifier
TBD

# Creating a Lex bot to automate your answers. 
TBD

# Create a customer application 
npx create-react-app chat-website-test
cd chat-website-test/
npm install amazon-connect-chatjs




