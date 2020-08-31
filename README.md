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
TBD

## Test the Flow
Open the Amazon Connect Console and navigate to the dasboard. Yoy can show/hide the guide in an hyperlink located near of the top on left hand side. 
When the guide is shown you can locate a "Test Chat" link on the first step. 

Clicking over it you can simulate Customer and Agent conversations on the same screen. 
Click over the link "Test Settings" to select the flow to test. 

# Create Lambda integration for customer identification. 
## Create a DynamoDB Table
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

