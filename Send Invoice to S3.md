# Document Ingestion System

## Project Overview
1. **User Interface:** UI for users to fill the invoice details.
2. **Integration:** Configure Connected System and Integration to transmit invoice data securely to AWS.
3. **Lambda and API Gateway:** Write a python lambda function to parse the incoming payload, store in S3 and draft a response to Appian. Configure API gateway with HTTP API, route, method and integration configuration.
4. **S3 bucket:** Create S3 bucket to store invoice details in CSV.
5. **Download CSV from S3:** Use Power Automate workflow to download Invoice file from S3 to local.

### Lambda function `Python function.py`
```python
import json
import boto3

def lambda_handler(event, context):
    try:
        # Extract the incoming body text safely
        body = event.get('body', '{}')
        if isinstance(body, str):
            payload = json.loads(body)
        else:
            payload = body
            
        # Get your individual invoice fields
        vendor = payload.get('vendorName')
        invoice_no = payload.get('invoiceNumber', 'MISSING_NO')
        amount = payload.get('totalAmount')
        
        if not vendor or not amount:
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'Missing critical information'})
            }
            
        # ─── NEW STEP: FORMAT AS A CLEAN CSV ROW ───
        # We combine our variables into a single line separated by commas.
        # This creates a spreadsheet-compatible text format natively.
        csv_row = f"{invoice_no},{vendor},{amount}\n"
        
        # In our next step, we will tell Lambda to write this 'csv_row' 
        # string directly into a cloud storage file.
        print(f"Formatted Spreadsheet Row: {csv_row}")

        # 2. Initialize the AWS Storage Client
        s3 = boto3.client('s3')
        bucket_name = 'invoices-from-appian' # Update this placeholder later
        file_name = 'pending_invoices.csv'
        
        # 3. Write/Append data to S3 cloud storage natively
        try:
            # Check if file already exists to read existing content
            existing_file = s3.get_object(Bucket=bucket_name, Key=file_name)
            current_content = existing_file['Body'].read().decode('utf-8')
            updated_content = current_content + csv_row
        except:
            # If file doesn't exist yet, start a fresh spreadsheet file with headers
            headers = "Invoice Number,Vendor Name,Total Amount\n"
            updated_content = headers + csv_row
            
        # 4. Save the file back up to your AWS S3 bucket
        s3.put_object(Bucket=bucket_name, Key=file_name, Body=updated_content)
        
        return {
            'statusCode': 201,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'status': 'SUCCESS', 'message': 'Data formatted for spreadsheet storage.'})
        }
        
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'system_error': str(e)})
        }
```

### Power Automate workflow to download CSV file from S3 to Local folder
<img width="1133" height="504" alt="image" src="https://github.com/user-attachments/assets/9f9ec6e5-1dfd-4541-993c-6b690eb0db60" />


