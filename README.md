Create Custom  objects and Fields (if not present)
Check if CareBenefitVerifyRequest__c and CoverageBenefit__c exist. If not:

-------------------------Step-1--------------------------------------------

Create CareBenefitVerifyRequest__c object with fields:
Field Label

Care_Representative_Name__c
Name
CreatedById
Date_of_Birth__c
Diagnosis_Code__c
External_Request_ID__c
Gender__c
Insurance__c(Member Plan)
LastModifiedById
OwnerId
Patient__c(account)
Patient_FirstName__c
Patient_LastName__c
Procedure_Code__c	
Provider__c (Account)
Provider_FirstName__c
Provider_LastName__c
Service_Date__c
Service_Type__c
Status__c
Status_Reason__c

 
Create CoverageBenefit__c object with:

Field Label
CareBenefitVerifyRequest__c
Coverage_Status__c
Coverage_Type__c
Name
CreatedById
LastModifiedById
Linked_Requests__c(CareBenefitVerifyRequest)
OwnerId
Status__c
Status_Reason__c

----------------------Step-2--------------------------------------------

Create Named Credentials(In classic version)

Go to Setup → Named Credentials

Click New Named Credential

Label: CareBenefitAPI

URL: https://infinitusmockbvendpoint-rji9z4.5sc6y6-2.usa-e2.cloudhub.io

Identity Type: Named Principal

Authentication Protocol: Password Authentication

Username: test_user

Password: test_password

Allow Callouts: ✅

Save.

----------------------------step-3-------------------------------------------
Apex Code Development

✅ 4. Create Apex Wrapper Class

📌 Developer Console → File → New → Apex Class → CareBenefitRequestWrapper

public class CareBenefitRequestWrapper {
    
    public class Patient {
        
        public String firstName;
        public String lastName;
        public Date dateOfBirth;
        public String gender;
    }

    public class Insurance {
        public String providerName;
        public String policyNumber;
        public String groupNumber;
        public String subscriberId;
    }

    public class Provider {
        public String npi;
        public String firstName;
        public String lastName;
    }

    public class Service {
        public String serviceType;
        public Date serviceDate;
        public String diagnosisCode;
        public String procedureCode;	
    }

    public Patient patient;
    public Insurance insurance;
    public Provider provider;
    public Service service;
    public string referenceId;
}

--------------------------------------------------------------------------------------------------------

Create Apex Class for Sending Request

📌 Developer Console → New → CareBenefitService

public with sharing class CareBenefitService {
    public static void sendRequest(CareBenefitVerifyRequest__c req) {
        Http http = new Http();
        HttpRequest httpReq = new HttpRequest();
        httpReq.setEndpoint('callout:CareBenefitAPI/benefit-verification-request');
        httpReq.setMethod('POST');
        httpReq.setHeader('Content-Type', 'application/json');

        CareBenefitRequestWrapper payload = new CareBenefitRequestWrapper();
        payload.referenceId = req.External_Request_ID__c;

        payload.patient = new CareBenefitRequestWrapper.Patient();
        payload.patient.firstName = req.Patient_FirstName__c;
        payload.patient.lastName = req.Patient_LastName__c;
        payload.patient.dateOfBirth = req.Date_of_Birth__c;
        payload.patient.gender = req.Gender__c;

        payload.insurance = new CareBenefitRequestWrapper.Insurance();
        payload.insurance.providerName = req.Provider__c;
       // payload.insurance.policyNumber = req.Policy_Number__c;
       // payload.insurance.groupNumber = req.Group_Number__c;
        //payload.insurance.subscriberId = req.Subscriber_ID__c;

        payload.provider = new CareBenefitRequestWrapper.Provider();
        //payload.provider.npi = req.NPI__c;
        payload.provider.firstName = req.Provider_FirstName__c;
        payload.provider.lastName = req.Provider_LastName__c;

        payload.service = new CareBenefitRequestWrapper.Service();
        payload.service.serviceType = req.Service_Type__c;
        payload.service.serviceDate = req.Service_Date__c;
        payload.service.diagnosisCode = req.Diagnosis_Code__c;
        payload.service.procedureCode = req.Procedure_Code__c;

        httpReq.setBody(JSON.serialize(payload));

        try {
            HttpResponse res = http.send(httpReq);
            if (res.getStatusCode() == 200) {
                Map<String, Object> resMap = (Map<String, Object>) JSON.deserializeUntyped(res.getBody());
                req.Status__c = (String) resMap.get('status');
                req.Status_Reason__c = (String) resMap.get('statusReason');
                update req;
            } else {
                System.debug('Callout Failed: ' + res.getBody());
            }
        } catch (Exception e) {
            System.debug('Callout Exception: ' + e.getMessage());
        }
    }
}
------------------------------------------------------------------------------------------------------------
 Create Trigger to assign it to queaue Request

trigger CBVRTrigger on CareBenefitVerifyRequest__c (before update) {
    for (CareBenefitVerifyRequest__c cbvr : Trigger.new) {
        Set<Id> userIds = new Set<Id>();
    for (CareBenefitVerifyRequest__c req : Trigger.new) {
        CareBenefitVerifyRequest__c old = Trigger.oldMap.get(req.Id);

        // Detect if ownership has changed from Queue to User
        if (req.OwnerId != old.OwnerId) {
            userIds.add(req.OwnerId);
        }
    }

    // Get User names
    Map<Id, User> userMap = new Map<Id, User>(
        [SELECT Id, Name FROM User WHERE Id IN :userIds]
    );

    for (CareBenefitVerifyRequest__c req : Trigger.new) {
        CareBenefitVerifyRequest__c old = Trigger.oldMap.get(req.Id);
        if (req.OwnerId != old.OwnerId && userMap.containsKey(req.OwnerId)) {
            req.Care_Representative_Name__c = userMap.get(req.OwnerId).Name;

            // ✅ Optional: Kick off external API call
            // CareBenefitService.sendRequest(req); (if logic should happen here)
        }
    }
        
}
}

----------------------------------------------------------------------------------------------------

 Create Trigger to Auto-send Request

📌 Developer Console → New Trigger → CBVRTrigger1

trigger CBVRTrigger1 on CareBenefitVerifyRequest__c (after insert) {
     for (CareBenefitVerifyRequest__c cbvr : Trigger.new) {
        CareBenefitService.sendRequest(cbvr);
    }
}

----------------------------------------------------------------------------------------------------
 REST API Endpoint to Receive Callback

 Create REST Resource Class
📌 Developer Console → New → CareBenefitVerificationAPI


@RestResource(urlMapping='/care-benefit-verification-results')
global with sharing class CareBenefitVerificationAPI {

    @HttpPost
    global static void receiveCallback() {
        RestRequest req = RestContext.request;
        String body = req.requestBody.toString();

        // Log body for debug
        System.debug('Received Callback: ' + body);

        // Parse JSON
        Map<String, Object> payload = (Map<String, Object>) JSON.deserializeUntyped(body);
        String externalId = (String) payload.get('externalId');
        String status = (String) payload.get('status');
        String statusReason = (String) payload.get('statusReason');

        if (String.isBlank(externalId)) {
            throw new CalloutException('Missing externalId in callback payload');
        }

        // Query the CBVR record by external ID
        List<CareBenefitVerifyRequest__c> cbvrList = [
            SELECT Id, Status__c, Status_Reason__c
            FROM CareBenefitVerifyRequest__c
            WHERE External_Request_ID__c = :externalId
            LIMIT 1
        ];

        if (cbvrList.isEmpty()) {
            throw new CalloutException('No matching CBVR record found for externalId: ' + externalId);
        }

        CareBenefitVerifyRequest__c cbvr = cbvrList[0];

        // Update the status and reason
        cbvr.Status__c = status;
        cbvr.Status_Reason__c = statusReason;

        // Create the linked CoverageBenefit__c record
        CoverageBenefit__c cb = new CoverageBenefit__c(
            CareBenefitVerifyRequest__c = cbvr.Id,
            Status__c = status,
            Status_Reason__c = statusReason
        );

        try {
            update cbvr;
            insert cb;
            System.debug('CBVR and CoverageBenefit__c updated/inserted successfully.');
        } catch (Exception e) {
            System.debug('Error in processing callback: ' + e.getMessage());
            throw new CalloutException('Callback processing failed.');
        }
    }
}


-----------------------------------------------------------------------------------------------------------
test class
@isTest
private class CareBenefitVerificationAPITest {
    @isTest static void testReceiveCallback() {
        // Insert a dummy CareBenefitVerifyRequest__c
        CareBenefitVerifyRequest__c req = new CareBenefitVerifyRequest__c(
            External_Request_ID__c = 'EXTERNAL123',
            Status__c = 'Acknowledged'
        );
        insert req;

        // Build a simulated callback
        String callbackJSON = '{"status": "Acknowledged", "statusReason": "Verified"}';

        // Simulate REST context
        RestRequest restReq = new RestRequest();
        restReq.requestBody = Blob.valueOf(callbackJSON);
        RestContext.request = restReq;
        RestContext.response = new RestResponse();

        // Invoke REST API logic directly
        Test.startTest();
        CareBenefitVerificationAPI.receiveCallback();
        Test.stopTest();

        // Check if coverage benefit was created
        List<CoverageBenefit__c> benefits = [SELECT Id, Status__c FROM CoverageBenefit__c];
        System.assert(!benefits.isEmpty(), 'Coverage Benefit should be created');
        System.assertEquals('Acknowledged', benefits[0].Status__c);
    }
}

----------------------------------------STEP-4--------------------------------------------------------
Use Workbench (Salesforce tool)
Steps:
Open: https://workbench.developerforce.com

Login with your Developer Org

Go to: Utilities → REST Explorer

Select POST method

Enter URL:

/services/apexrest/care-benefit-verification-results

Enter Request Body:

{
  "externalId": "12345",
  "status": "Acknowledged",
  "statusReason": "Approved by Payer"
}

-------------------------------------------------------------------------------------------------------------
