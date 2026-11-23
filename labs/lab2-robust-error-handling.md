Lab 2 – Robust iFlow with Error Handling (L2_Customer_Orders_Robust)
Lab 2 turns the Lab 1 happy-path iFlow into one that handles bad input, missing data and unexpected failures the way a production integration should. Built and tested on SAP Integration Suite (trial, US10), November 2026.
Endpoint: `POST /http/premach/customer-orders-v2`
Security: OAuth 2.0 client credentials (service key on a Process Integration Runtime instance, role `ESBMessaging.send`)
Backend: Northwind OData V2 demo service (`Customers`, `Orders`)
How the flow is built
```
HTTPS -> JSON to XML -> Content Modifier 1 (property customerId = //customerId)
          |
       Router 1 --"Missing ID" (customerId = '')--> Set 400 -> End 1
          | default "Valid"
       Request Reply 2 -> OData: Customers?$filter=CustomerID eq '...'
          |
       Read Customer (property customerName = //CompanyName)
          |
       Router 2 --"Not Found" (customerName = '')--> Set 404 -> End 2
          | default "Found"
       Request Reply 1 -> OData: Orders?$filter=CustomerID eq '...'
          |
       Groovy Script 1 (orders to JSON, adds customerName) -> End (200)

Exception Subprocess 1:
       Error Start -> Build Error Response (Groovy) -> End 3 (500 + mplId)
```
What each part added
Part	Pattern	What was built	Test result
A	Input validation	Router 1 with a Non-XML condition; Set 400 sets `CamelHttpResponseCode` = 400, `Content-Type` and a constant JSON body	Empty customerId returned 400 in 44 ms, before any backend call
B	Chained lookup with not-found handling	Request Reply 2 checks the customer exists; an empty `customerName` property is the not-found signal; Router 2 sends it to Set 404 with an expression body that includes the ID	`test` returned 404; `ALFKI` returned 200 with "Alfreds Futterkiste"
C	Central exception handling	Exception Subprocess with a Groovy script that reads `CamelExceptionCaught` and `SAP_MessageProcessingLogID`, logs ErrorDetail to the message log, and returns a safe JSON 500 with the mplId	`hello` (not JSON) returned 500 with the mplId; Message ID in the monitor matched; ErrorDetail present under Custom Headers
Exception handler script (Part C)
```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import groovy.json.JsonOutput

def Message processData(Message message) {
    def ex     = message.getProperty('CamelExceptionCaught')
    def mplId  = message.getProperty('SAP_MessageProcessingLogID')
    def detail = ex ? ex.getMessage() : 'Unknown error'

    def messageLog = messageLogFactory.getMessageLog(message)
    if (messageLog != null) {
        messageLog.addCustomHeaderProperty('ErrorDetail', detail.take(250))
    }

    def result = [status : 'error',
                  message: 'Unexpected error while processing the request. Contact support with the reference ID.',
                  mplId  : mplId]
    message.setBody(JsonOutput.toJson(result))
    message.setHeader('Content-Type', 'application/json')
    message.setHeader('CamelHttpResponseCode', 500)
    return message
}
```
Concepts
Exchange properties vs headers: properties hold internal state (customerId, customerName); headers go back to the caller (`CamelHttpResponseCode`, `Content-Type`).
Router default route: catches everything that doesn't match a condition. Each branch needs its own End event.
Empty value as a signal: an XPath that finds nothing returns an empty string, which a router can test.
End Message vs Error End: ending the subprocess with End Message shows the message as Completed and the caller gets the clean response. Error End marks it Failed, which triggers standard alerts but returns a generic error. In production, give the caller a clean response and make the failure visible to operations.
OData filter values: text goes in quotes (`'ALFKI'`); numbers don't (`CategoryID eq 1`).
Older Groovy runtime: use `new XmlSlurper()` without `import groovy.xml.XmlSlurper`, and read the body as `java.io.Reader` for streaming.
Problems hit and how they were diagnosed
Symptom	Cause	Lesson
Deploy errors; all iFlows stuck "Stopping"	Trial platform outage	Deploy a known-good flow to rule out your own code; keep an exported backup
Token request returned 401 `invalid_client`	Postman held credentials from an earlier key	Re-copy clientid and clientsecret whenever a key is recreated
404 on the request	Called `/customer-orders` instead of `-v2`	404 means the wrong path or the flow isn't deployed
ID search in the monitor finds nothing	Trial tenant search issue	Fall back to filtering by time and artifact
Design principle
Every iFlow gets three error paths: a router returns a 4xx for bad input, a lookup returns a 404 when the business record doesn't exist, and an exception subprocess handles anything unexpected by returning a reference ID and logging the detail. Callers never see a stack trace, and support can trace any failure.
