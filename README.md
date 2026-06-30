# Multi-Vendor Hybrid Cloud API Pipeline

A functional, serverless microservice pipeline connecting an enterprise low-code platform (Appian) with public cloud infrastructure (AWS Serverless) via secure REST API channels.

## Architecture Overview

```text
[Appian Cloud Sandbox] ──(Outbound HTTPS POST)──> [AWS API Gateway] ──> [AWS Lambda (Python)] ──(Inbound Write)──> [Appian Data Fabric / DB]
```

1. **Outbound Trigger**: Appian fires a structured JSON data payload over HTTPS.
2. **API Routing Gateway**: Amazon API Gateway catches the public web request and safely forwards the payload.
3. **Serverless Compute Engine**: AWS Lambda running a Python 3.12 microservice parses the transaction.
4. **Data Fabric Target**: The script maps and forwards the sanitized payload to write permanently to a backend relational table.

---

## Code Artifacts

### 1. AWS Lambda Microservice Script (`lambda_function.py`)
```python
import json
import urllib.request

def lambda_handler(event, context):
    print("Received Appian event: " + json.dumps(event))
    
    # Extract data securely from the API Gateway wrapper
    if 'body' in event and isinstance(event['body'], str):
        payload = json.loads(event['body'])
    else:
        payload = event
        
    # Map fields to match targeted data structures
    appian_data = {
        "CANDIDATE_NAME": payload.get("CANDIDATE_NAME", "Testing"),
        "TARGET_ROLE": payload.get("TARGET_ROLE", "Cloud Engineer"),
        "LEARNING_STATUS": payload.get("LEARNING_STATUS", "Learning")
    }
    
    # Sanitized URL placeholder for public repository safety
    url = "https://<YOUR-APPIAN-ENVIRONMENT-SUBDOMAIN>://"
    
    req = urllib.request.Request(
        url = url,
        data = json.dumps(appian_data).encode('utf-8'),
        headers = {'Content-Type': 'application/json'},
        method = 'POST'
    )
    
    try:
        with urllib.request.urlopen(req) as response:
            res_body = response.read().decode('utf-8')
            print("Target System Response: " + res_body)
            return {
                'statusCode': 200,
                'body': json.dumps({
                    'message': 'Success! AWS Lambda forwarded data.',
                    'reply': json.loads(res_body)
                })
            }
    except Exception as e:
        print("Routing Error: " + str(e))
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Failed to route data to target database.'})
        }
```

### 2. Appian Data Fabric Web API Layer (`web_api_expression.txt`)
```appian
a!localVariables(
  local!json: a!fromJson(http!request.body),
  local!value: 'recordType!HS AWS Lambda'(
    'recordType!HS AWS Lambda.fields.CANDIDATE_NAME': index(local!json, "CANDIDATE_NAME", null),
    'recordType!HS AWS Lambda.fields.TARGET_ROLE': index(local!json, "TARGET_ROLE", null),
    'recordType!HS AWS Lambda.fields.STATUS': index(local!json, "LEARNING_STATUS", null)
  ),
  if(
    a!isNullOrEmpty(local!json),
    null,
    a!writeRecords(
      records: local!value,
      onSuccess: a!httpResponse(
        statusCode: 200,
        headers: { a!httpHeader(name: "Content-Type", value: "application/json") },
        body: a!toJson(fv!recordsUpdated)
      ),
      onError: a!httpResponse(
        statusCode: 500,
        headers: { a!httpHeader(name: "Content-Type", value: "application/json") },
        body: a!toJson(a!map(message: "Write request has failed", error: fv!error))
      )
    )
  )
)
```

---

## Real-World Engineering & Troubleshooting Insights

Building this hybrid lab exposed several critical architecture constraints and cross-platform boundaries:

* **Runtime Data Passing Optimization**: Successfully debugged an AWS Python runtime payload extraction error by isolating how Amazon API Gateway packages proxy requests, shifting from raw text manipulation to a structured dictionary parameter check (`if 'body' in event`).
* **Modern Low-Code Architecture Pivots**: Navigated platform constraints in the modern Appian Community Edition environment by bypassing legacy Custom Data Types (CDTs) and Data Store Entities. Architected the integration using the modern Appian Data Fabric abstraction layer, deploying an advanced `a!writeRecords` Web API blueprint to map incoming AWS JSON streams directly to physical relational database tables.
* **Multi-Tenant Sandbox Constraints**: Uncovered a structural security architecture boundary within the free Appian Community Edition framework. The environment disables standard basic inbound authentication handshakes and service account token generation (`Bearer` headers) on incoming internet payloads. The complete data cycle evaluates cleanly up to this platform-enforced sandbox firewall.
