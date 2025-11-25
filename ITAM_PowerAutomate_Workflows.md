# Power Automate Workflows - Implementation Guide

## 1. Asset Assignment Workflow

**Trigger**: Record creation on Asset Assignment entity

**Configuration Details**:

```
Trigger: When a row is added to Asset Assignment table

Step 1: Get Asset Details
Action: Get a row
Table: Assets
Row ID: AssetID (from trigger)
Retrieve columns: AssetName, Status, Owner, Department, CurrentValue

Step 2: Validate Asset Status
Action: Condition
Condition: Status equals "Active"
If True: Continue to Step 3
If False: Compose error message and exit

Step 3: Get User Details
Action: Get a row
Table: Contact/User table
Row ID: AssignedToUser (from trigger)
Retrieve columns: Email, Name, Department, Manager

Step 4: Check for Valid Onboarding
Action: List rows (filtered)
Table: Employee Onboarding table
Filter: UserID = AssignedToUser AND Status = "Complete"
If count = 0: Send warning email to IT Manager, continue anyway

Step 5: Send Assignment Notification Email
Action: Send an email (V2)
To: body('Get User Details')?['email']
Subject: "New Asset Assignment: @{body('Get Asset Details')?['AssetName']}"
Body: HTML formatted email with:
- Asset details (name, serial number, model)
- Handover instructions
- Warranty information
- Support contact details
- Calendar invitation link

Step 6: Schedule Reminder (if assignment is Temporary/Loan)
Action: Condition
Condition: AssignmentType = "Loan" OR "Temporary"
If True:
  Action: Schedule a cloud flow
  Delay: Calculate days until UnassignedDate minus 14 days
  Action: Send reminder email to AssignedToUser and Manager
  Subject: "Asset Return Reminder: @{AssetName} due in 14 days"

Step 7: Update Audit Log
Action: Create a row
Table: AuditLog
Fields:
- EntityType: "Asset Assignment"
- EntityID: AssignmentID
- ActionPerformed: "Asset assigned"
- ChangedBy: User().Email
- ChangeDate: utcNow()
- AfterValue: JSON object with assignment details
- Reason: AssignmentNotes

Step 8: Update Asset Status
Action: Update a row
Table: Asset
Row ID: AssetID
Fields:
- CurrentOwner: AssignedToUser
- LastAssignmentDate: utcNow()
- AssignmentStatus: "Assigned"
```

**Error Handling**:
- Add Try-Catch error handling around email steps
- If email fails, create Task record for manual follow-up
- Log all errors to separate Error Log table for monitoring

**Advanced Logic**:
- If asset is being reassigned (previous owner exists), send "Asset Unassigned" notification to previous owner
- Check department budget for asset transfer costs
- Validate assignment against CompliancePolicy restrictions (e.g., no restricted assets in certain departments)

---

## 2. Warranty Expiration Monitoring Workflow

**Trigger**: Scheduled cloud flow (Daily at 6 AM)

**Configuration Details**:

```
Step 1: List All Active Warranties
Action: List rows
Table: Warranty table
Filter: 
  EndDate >= utcNow() AND 
  EndDate <= addDays(utcNow(), 90) AND 
  ReminderSent = false

Step 2: For Each Warranty (Apply to each)
Action: Apply to each
Input: value (from Step 1)

Inside loop:

Step 2.1: Get Asset Details
Action: Get a row
Table: Asset
Row ID: AssetID (from warranty loop)
Retrieve: AssetName, Owner, Department

Step 2.2: Get Warranty Provider Details
Action: Get a row
Table: Warranty Provider (Vendor)
Row ID: WarrantyProvider
Retrieve: VendorName, ContactEmail, PhoneNumber

Step 2.3: Calculate Days Until Expiry
Action: Compose
Expression: 
dateDiff(utcNow(), items('Apply_to_each')?['EndDate'], 'D')

Step 2.4: Determine Urgency Level
Action: Condition
If daysDiff < 30: Priority = "Critical"
If daysDiff 30-60: Priority = "High"
If daysDiff 60-90: Priority = "Medium"

Step 2.5: Send Renewal Reminder Email
Action: Send an email (V2)
To: body('Get Asset Details')?['Owner.Email']
CC: Manager email (if available)
Subject: 
  Expression: concat('URGENT: Warranty Expiring in ', daysDiff, ' days - ', AssetName)

Body: HTML with:
- Warranty details (type, start date, end date)
- Renewal options and pricing
- Warranty provider contact information
- Direct renewal link (if available)
- Risk assessment of not renewing
- Approval workflow link

Step 2.6: Create Task for Procurement
Action: Create a row
Table: Task/Issue
Fields:
- Title: concat('Renew Warranty - ', AssetName)
- Description: Warranty details and renewal recommendations
- DueDate: addDays(items('Apply_to_each')?['EndDate'], -7)
- AssignedTo: Procurement Manager
- Priority: (from Step 2.4)
- LinkedWarranty: Current warranty record ID

Step 2.7: Check Asset Lifecycle Alignment
Action: Condition
If AssetAge > 5 years AND EndDate < 365 days away:
  Send additional email to Finance recommending asset refresh vs warranty renewal
  Create cost comparison analysis task

Step 2.8: Mark Reminder as Sent
Action: Update a row
Table: Warranty
Row ID: Current warranty ID
Fields:
- ReminderSent: true
- ReminderDate: utcNow()

Step 2.9: Create Audit Entry
Action: Create a row
Table: AuditLog
Fields:
- EntityType: "Warranty Reminder"
- EntityID: Warranty ID
- ActionPerformed: "Expiration reminder sent"
- ChangedBy: "System"
- ChangeDate: utcNow()

Step 3: Summary Report (After loop)
Action: Create HTML table with all processed warranties
Action: Send summary email to IT Manager
Include: Count of reminders sent, breakdown by priority, total at-risk asset value
```

**Advanced Logic**:
- If warranty is critical (expiring within 7 days) and reminder not yet sent, send SMS alert to asset owner
- Integrate with procurement system to check if renewal PO already exists
- Calculate warranty cost per month to aid renewal decisions
- Flag warranties with high claim rates for provider evaluation

**Error Handling**:
- Wrap email sending in Try-Catch; if fails, retry with exponential backoff
- Log any warranties that couldn't be retrieved due to missing owner/provider info
- Send daily summary of processing errors to IT Admin

---

## 3. Invoice Processing with AI Extraction Workflow

**Trigger**: When an invoice is created OR when invoice document is uploaded to SharePoint

**Configuration Details**:

```
Trigger: When a row is created (Invoice table)

Step 1: Get Invoice Details
Action: Get a row
Table: Invoice
Row ID: InvoiceID (from trigger)
Retrieve: InvoiceNumber, InvoiceDate, VendorID, RawDocumentPath

Step 2: Retrieve Invoice Document
Action: Get file content using path
Path: RawDocumentPath (from SharePoint or Blob Storage)

Step 3: Call Azure Form Recognizer
Action: HTTP request (or use pre-built connector if available)
Method: POST
URI: https://{region}.api.cognitive.microsoft.com/formrecognizer/v3.1-preview.1/prebuilt-invoice/analyze

Headers:
- Ocp-Apim-Subscription-Key: @{variables('FormRecognizerKey')}
- Content-Type: application/octet-stream

Body: body('Get file content')

Expected Response Structure:
{
  "analyzeResult": {
    "documents": [{
      "fields": {
        "InvoiceNumber": {"value": "INV-2024-001", "confidence": 0.95},
        "InvoiceDate": {"value": "2024-01-15", "confidence": 0.98},
        "VendorName": {"value": "TechCorp Inc", "confidence": 0.92},
        "Items": {
          "values": [
            {
              "Description": {"value": "Lenovo ThinkPad X1 Carbon"},
              "Quantity": {"value": 5},
              "UnitPrice": {"value": 1500},
              "Amount": {"value": 7500}
            }
          ]
        },
        "Total": {"value": 7500, "confidence": 0.99}
      }
    }]
  }
}

Step 4: Parse JSON Response
Action: Parse JSON
Schema: (auto-detect from Step 3 response)

Step 5: Validate Confidence Scores
Action: Condition - Check confidence thresholds
If all scores > 95%: Mark as HIGH_CONFIDENCE
If scores 85-95%: Mark as MEDIUM_CONFIDENCE
If scores < 85%: Mark as LOW_CONFIDENCE, flag for manual review

Step 6: Create/Update Invoice Fields
Action: Condition
If HIGH_CONFIDENCE:
  Update Invoice record:
  - InvoiceNumber: parsed value
  - InvoiceDate: parsed value
  - VendorID: lookup vendor by parsed VendorName
  - TotalAmount: parsed total
  - ProcessingStatus: "Extracted"
  - ConfidenceScore: average of all confidence scores

If MEDIUM_CONFIDENCE:
  Create Task for Finance:
  - Title: "Manual Review Required - Invoice @{InvoiceNumber}"
  - Description: List fields with confidence scores
  - AssignedTo: Finance Manager
  - Due: Today + 1 day
  
  Send email to Finance Manager with extracted data and options to:
  - Approve extraction
  - Correct specific fields
  - Reject extraction

If LOW_CONFIDENCE:
  Create Issue for Finance/Procurement
  - Priority: High
  - Description: "Low confidence extraction, manual processing required"
  - Attach original document for reference

Step 7: Auto-Create/Link Assets from Line Items
Action: Apply to each
Input: Line Items array from parsed response

For each line item:
  
Step 7.1: Fuzzy Match Item Description to Asset Catalog
Action: List rows (filtered)
Table: Asset table
Filter: AssetType matches item description (using CONTAINS logic)
Order by: Similarity score (descending)

Step 7.2: If Match Found (confidence > 80%):
  Create AssetInvoiceLink record:
  - InvoiceID: Current invoice
  - AssetID: Matched asset
  - LineItemDescription: Item description from invoice
  - Quantity: Parsed quantity
  - UnitPrice: Parsed unit price
  - MatchConfidence: Similarity score
  
Step 7.3: If No Match Found (confidence < 80%) OR No match exists:
  Create new Asset record:
  - AssetName: Item description
  - AssetType: AI categorization (call Azure OpenAI if available)
  - Manufacturer: Extract from description or vendor data
  - Model: Extract from description
  - PurchaseDate: Invoice date
  - OriginalCost: Unit price
  - Status: "Received"
  - SourceInvoiceID: Current invoice
  - Flag for: Manual review and categorization
  
  Create Task for IT Admin:
  - Title: "Classify New Asset from Invoice"
  - Description: "Item @{Description} from @{VendorName} needs asset type verification"

Step 8: Validate Against Procurement Policies
Action: Get Compliance Policy (filter by AssetType)
For each item:
  
  Check: Is vendor in approved vendor list?
  If NO: Send alert to Procurement Manager
  
  Check: Does item comply with approved asset types for department?
  If NO: Create Issue - "Non-compliant purchase detected"
  
  Check: Does total amount exceed budget limit for asset type?
  If YES: Send approval request to Finance Director
  
  Calculate: Ensure line items sum to total amount (within 1% tolerance)
  If variance > 1%: Flag for manual review

Step 9: Route to Approval Workflow
Action: Condition
If ProcessingStatus = "Extracted" AND TotalAmount > $10,000:
  Create Approval task
  Assigned to: Finance Director
  With: Invoice details, extracted data, extracted assets list
  Deadline: 3 business days
  
If ProcessingStatus = "Extracted" AND TotalAmount <= $10,000:
  Auto-approve
  Send notification to Procurement team
  
If ProcessingStatus = "Requires Review":
  Create Task for Finance Manager
  Deadline: Today + 2 days

Step 10: Send Confirmation Email
Action: Send email to Procurement/Finance team
Subject: "Invoice @{InvoiceNumber} Processed - @{ProcessingStatus}"
Body: 
- Summary of extraction results
- List of extracted assets
- Confidence score
- Any discrepancies or flags
- Action required (if any)
- Link to Invoice record for manual review

Step 11: Create Audit Entry
Action: Create AuditLog record
- EntityType: "Invoice Processing"
- EntityID: InvoiceID
- ActionPerformed: "Invoice extracted via AI"
- ConfidenceScore: Average confidence
- ExtractedAssetCount: Count of assets created/linked
- ChangeDate: utcNow()

Step 12: Error Handling
Try-Catch around entire workflow:
  If Form Recognizer call fails:
    Retry with exponential backoff (3 attempts)
    If still fails: Set ProcessingStatus = "ExtractionFailed"
    Create Issue for DevOps team
    Send alert email
```

**Integration with Blob Storage**:
- Original invoice PDF stored in Azure Blob Storage with structure: invoices/{year}/{month}/{InvoiceID}.pdf
- Access control via Azure AD
- Retention policy: Keep for 7 years for audit compliance

**Advanced Features**:
- ML model retraining: Collect human corrections and retrain Form Recognizer monthly
- Confidence score trending: Monitor if confidence scores declining over time (indicates model drift)
- Cost tracking: Update Budget tracking entity with extracted amounts in real-time

---

## 4. Maintenance Schedule Automation Workflow

**Trigger**: When a MaintenanceSchedule row is created

**Configuration Details**:

```
Trigger: When a row is added to MaintenanceSchedule table

Step 1: Get Schedule Details
Action: Get a row
Table: MaintenanceSchedule
Row ID: ScheduleID
Retrieve all fields

Step 2: Calculate Days Until Maintenance
Action: Compose
Expression: dateDiff(utcNow(), ScheduledDate, 'D')
Store in variable: DaysUntilMaintenance

Step 3: Get Asset Details
Action: Get a row
Table: Asset
Row ID: AssetID (from schedule)
Retrieve: AssetName, Owner, Location, MaintenanceHistory

Step 4: Get Maintenance Provider Details
Action: Get a row
Table: Vendor/MaintenanceProvider
Row ID: MaintenanceProvider
Retrieve: VendorName, ContactEmail, PhoneNumber, SLAResponse

Step 5: Check SLA Response Time
Action: Condition
If DaysUntilMaintenance < 7:
  Priority = "Urgent"
  SLARequired = true
  Send immediate notification to MaintenanceProvider

Else if DaysUntilMaintenance < 14:
  Priority = "High"
  
Else:
  Priority = "Standard"

Step 6: Send Notification to Maintenance Provider
Action: Send email (V2) + Teams message
To Provider Email
Subject: "@{Priority} - Scheduled Maintenance: @{AssetName}"
Include: 
- Asset details and location
- Maintenance type and requirements
- Contact for asset owner
- SLA response time (if applicable)
- Calendar invitation

Also post to Teams channel #MaintenanceBridge with @mentions

Step 7: Schedule Pre-Maintenance Reminders
Action: Schedule cloud flow
Repeat: 3 times (1 week before, 1 day before, 2 hours before)

For each reminder:
  Action: Send email to Asset Owner and Manager
  Subject: "Maintenance Reminder: @{AssetName}"
  Include: Maintenance details, expected downtime, temporary replacement (if any)

Step 8: Create Checklist Task
Action: Create a row
Table: Task/Checklist
Fields:
- Title: "Maintenance Checklist - @{AssetName}"
- AssignedTo: MaintenanceProvider
- DueDate: ScheduledDate
- Description: HTML with pre-configured checklist items:
  - Pre-maintenance backup status
  - Current asset condition assessment
  - Required spare parts availability
  - Post-maintenance testing checklist
  - Documentation to update

Link to MaintenanceSchedule record for easy access

Step 9: Send Notification to Asset Owner
Action: Send email
To: Asset Owner
Subject: "Upcoming Maintenance: @{AssetName} on @{ScheduledDate}"
Body:
- Maintenance details
- Expected impact/downtime
- Actions asset owner should take (backup, save work, etc.)
- Contact for questions

Step 10: Create Audit Entry
Action: Create AuditLog
- EntityType: "MaintenanceSchedule"
- EntityID: ScheduleID
- ActionPerformed: "Maintenance scheduled"
- ChangeDate: utcNow()

Step 11: On Maintenance Completion (Separate scheduled flow triggered on status change)
Trigger: When MaintenanceSchedule Status = "Completed"

Step 11.1: Get Completion Details
Action: Get the updated MaintenanceSchedule record

Step 11.2: Request Technician Upload
Action: Send email to MaintenanceProvider
Subject: "Please Submit Maintenance Report: @{AssetName}"
Include: Upload link to form for:
- Work performed
- Issues found
- Spare parts used
- Time spent
- Next maintenance recommendation
- Signature/certification

Step 11.3: Update Asset Record
Action: Update Asset
Fields:
- LastMaintenanceDate: CompletedDate
- MaintenanceStatus: "Compliant"
- NextMaintenanceDueDate: 
  Expression: addDays(CompletedDate, MaintenanceIntervalDays)
  [MaintenanceIntervalDays retrieved from Asset definition]

Step 11.4: Send Completion Notification
Action: Send email to Asset Owner
Subject: "Maintenance Completed: @{AssetName}"
Body:
- Completion date/time
- Summary of work performed
- Any issues found
- Recommendations for future maintenance
- Contact for follow-up questions

Step 11.5: Archive Old Records
Action: Scheduled flow (Monthly)
List all MaintenanceSchedule records with:
- Status = "Completed"
- CompletedDate < 3 years ago

For each old record:
  Archive to Azure SQL database
  Delete from active Dataverse table (optional, for performance)

```

**Error Handling**:
- If MaintenanceProvider email delivery fails, create Task for IT Manager to contact provider manually
- If scheduled maintenance exceeds estimated duration by >50%, trigger escalation to IT Director
- Monitor for missed maintenance (ScheduledDate passed, Status still "Planned") and send alerts

---

## 5. Workflow Monitoring and Optimization

**Flow Run Analytics**:
- Create Power Automate analytics dashboard tracking:
  - Flow success/failure rates
  - Average execution time
  - Cost per run (in Power Automate units)
  - Peak execution times
  
**Optimization Techniques**:
- Use batch operations instead of apply-to-each where possible
- Implement parallel branching for independent operations
- Cache frequently accessed data using variables
- Schedule resource-intensive flows during off-peak hours
- Use Filter array instead of List rows with complex filters

**Monitoring Alerts**:
- Flow fails > 5% of runs: Alert DevOps
- Flow execution time > 2x average: Investigate performance
- Error message pattern detection: Proactive issue resolution

---

## Deployment and Version Control

All workflows should be version controlled in GitHub with the following structure:
```
/power-automate-flows/
  /asset-assignment/
    - flow-definition.json
    - flow-configuration.md
    - test-cases.xlsx
  /warranty-monitoring/
  /invoice-processing/
  /maintenance-scheduling/
  /compliance-audit/
  /shared-resources/
    - shared-connections.json
    - environment-variables.json
```

Use Power Automate cloud flows in Canvas or automated triggers, with clear naming conventions:
- Format: [Trigger Type] - [Primary Entity] - [Action]
- Example: "Automated - Invoice - Process with AI"

Implement change management with approval gates before deploying to Production environment.
