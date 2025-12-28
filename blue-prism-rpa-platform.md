# Blue Prism - Enterprise RPA Platform

## Tổng quan về Blue Prism

Blue Prism là nền tảng RPA enterprise-grade, được thiết kế cho các tổ chức lớn với yêu cầu cao về bảo mật, governance và scalability. Blue Prism sử dụng kiến trúc server-based và cung cấp khả năng quản lý tập trung.

## Kiến trúc Blue Prism

### Core Components
- **Application Server**: Central management và execution
- **Database Server**: Lưu trữ processes, schedules và audit logs
- **Interactive Client**: Development environment
- **Runtime Resource**: Execution machines
- **Control Room**: Web-based management interface

### Security Model
- **Role-based Access Control (RBAC)**
- **Process-level Security**
- **Credential Management**
- **Audit Trail**

## Development Environment

### Process Studio
Blue Prism sử dụng visual process designer với flowchart-based approach.

#### Basic Process Structure
```
Main Page
├── Start
├── Initialize
├── Get Work Data
├── Process Item
├── Update Results
└── End
```

### Object Studio
Tạo reusable business objects để tương tác với applications.

#### Application Modeller
```xml
<ApplicationModel>
  <Application Name="Calculator" 
               Path="calc.exe"
               ProcessName="Calculator">
    
    <Elements>
      <Element Name="Button_1"
               Type="Button"
               Match="1"
               Attributes="text:1;class:Button">
        <Identification>
          <Attribute Name="Text" Value="1" />
          <Attribute Name="Class Name" Value="Button" />
        </Identification>
      </Element>
      
      <Element Name="Button_Plus"
               Type="Button" 
               Match="+"
               Attributes="text:+;class:Button">
        <Identification>
          <Attribute Name="Text" Value="+" />
          <Attribute Name="Class Name" Value="Button" />
        </Identification>
      </Element>
    </Elements>
  </Application>
</ApplicationModel>
```

## Process Development

### Basic Actions

#### Navigation Actions
```vb
' Launch Application
Application Manager::Launch Application
  Application Name: "Calculator"
  Wait for Application: True
  Timeout: 30

' Attach to Application  
Application Manager::Attach
  Application Name: "Calculator"
  Timeout: 10

' Navigate to Element
Navigate::Navigate
  Application Name: "Calculator"
  Element Name: "Button_1"
  Action: "Click"
```

#### Data Manipulation
```vb
' Read from Collection
Utility - Collection Manipulation::Get Collection Field
  Collection Name: "WorkData"
  Field Name: "CustomerID" 
  Row: 1
  Output: CustomerID

' Write to Collection
Utility - Collection Manipulation::Add Row
  Collection Name: "Results"
  Fields: 
    - CustomerID: [CustomerID]
    - Status: "Processed"
    - Timestamp: [Now()]

' Loop through Collection
Loop Start::Loop Start - Collection
  Collection Name: "WorkData"
  Loop Collection Name: "CurrentRow"

' Loop End
Loop End::Loop End
```

#### File Operations
```vb
' Read Excel File
MS Excel VBO::Open Workbook
  File Path: "C:\Data\input.xlsx"
  Output: Workbook Handle

MS Excel VBO::Get Worksheet as Collection
  Workbook Handle: [Workbook Handle]
  Worksheet Name: "Sheet1"
  Output: ExcelData

' Write Excel File
MS Excel VBO::Create Instance
  Output: Excel Handle

MS Excel VBO::Create Workbook
  Excel Handle: [Excel Handle]
  Output: Workbook Handle

MS Excel VBO::Write Collection
  Excel Handle: [Excel Handle]
  Workbook Handle: [Workbook Handle]
  Collection: [ProcessedData]
  Worksheet Name: "Results"
```

### Advanced Process Patterns

#### Exception Handling
```vb
' Try Block
Exception::Try
  Exception Type: "System Exception"
  
  ' Main process logic here
  Navigate::Navigate
    Application Name: "WebApp"
    Element Name: "SubmitButton"
    Action: "Click"
    
' Catch Block  
Exception::Catch
  Exception Type: "System Exception"
  
  ' Error handling logic
  Utility - Strings::Concatenate
    Text 1: "Error occurred: "
    Text 2: [Exception.Message]
    Output: ErrorMessage
    
  ' Log error
  Utility - File Management::Write Text File
    File Path: "C:\Logs\errors.log"
    Text: [ErrorMessage]
    Append: True

' Finally Block
Exception::Finally
  ' Cleanup logic
  Application Manager::Terminate
    Application Name: "WebApp"
```

#### Retry Logic
```vb
' Initialize retry counter
Calculation::Calculate
  Expression: "0"
  Output: RetryCount

' Retry Loop
Loop Start::Loop Start - Count
  Start Number: 1
  End Number: 3
  
  Exception::Try
    Exception Type: "System Exception"
    
    ' Action that might fail
    Navigate::Navigate
      Application Name: "WebApp"
      Element Name: "DataElement"
      Action: "Get Text"
      Output: ExtractedText
      
    ' If successful, exit loop
    Control::Resume
    
  Exception::Catch
    Exception Type: "System Exception"
    
    ' Increment retry count
    Calculation::Calculate
      Expression: "[RetryCount] + 1"
      Output: RetryCount
      
    ' Wait before retry
    Utility - General::Wait
      Duration: "00:00:05"

Loop End::Loop End
```

## Business Objects Development

### Web Automation Object
```vb
' Launch Browser Action
Action: Launch Browser
Inputs:
  - URL (Text): The website URL to navigate to
  - Browser Type (Text): Chrome/IE/Firefox
  
Code Stage:
  Dim browser As Object
  Select Case Browser_Type
    Case "Chrome"
      browser = CreateObject("Chrome.Application")
    Case "IE" 
      browser = CreateObject("InternetExplorer.Application")
  End Select
  
  browser.Navigate(URL)
  browser.Visible = True
  
Outputs:
  - Browser Handle (Number): Reference to browser instance

' Extract Table Data Action  
Action: Extract Table Data
Inputs:
  - Browser Handle (Number): Browser instance reference
  - Table Element (Text): CSS selector for table
  
Code Stage:
  Dim doc As Object
  Dim table As Object
  Dim rows As Object
  Dim collection As New Collection
  
  Set doc = browser.Document
  Set table = doc.querySelector(Table_Element)
  Set rows = table.getElementsByTagName("tr")
  
  For i = 1 To rows.Length - 1
    Dim cells As Object
    Set cells = rows(i).getElementsByTagName("td")
    
    Dim row As New Collection
    For j = 0 To cells.Length - 1
      row.Add cells(j).innerText
    Next j
    collection.Add row
  Next i
  
Outputs:
  - Table Data (Collection): Extracted table data
```

### Database Object
```vb
' Connect to Database Action
Action: Connect Database
Inputs:
  - Connection String (Text): Database connection string
  - Timeout (Number): Connection timeout in seconds
  
Code Stage:
  Dim conn As Object
  Set conn = CreateObject("ADODB.Connection")
  
  conn.ConnectionTimeout = Timeout
  conn.Open Connection_String
  
Outputs:
  - Connection Handle (Number): Database connection reference

' Execute Query Action
Action: Execute Query  
Inputs:
  - Connection Handle (Number): Database connection
  - SQL Query (Text): SQL statement to execute
  
Code Stage:
  Dim rs As Object
  Dim collection As New Collection
  
  Set rs = conn.Execute(SQL_Query)
  
  ' Convert recordset to collection
  Do While Not rs.EOF
    Dim row As New Collection
    For i = 0 To rs.Fields.Count - 1
      row.Add rs.Fields(i).Value, rs.Fields(i).Name
    Next i
    collection.Add row
    rs.MoveNext
  Loop
  
  rs.Close
  
Outputs:
  - Query Results (Collection): Query result data
```

## Work Queue Management

### Queue Configuration
```xml
<WorkQueue Name="InvoiceProcessing">
  <Fields>
    <Field Name="InvoiceNumber" Type="Text" Key="True" />
    <Field Name="CustomerName" Type="Text" />
    <Field Name="Amount" Type="Number" />
    <Field Name="DueDate" Type="Date" />
    <Field Name="Priority" Type="Text" />
  </Fields>
  
  <Settings>
    <MaxRetries>3</MaxRetries>
    <RetryInterval>300</RetryInterval>
    <ItemTimeout>1800</ItemTimeout>
  </Settings>
</WorkQueue>
```

### Queue Operations
```vb
' Add Item to Queue
Work Queues::Add To Queue
  Queue Name: "InvoiceProcessing"
  Item Data:
    - InvoiceNumber: [InvoiceNum]
    - CustomerName: [Customer]
    - Amount: [InvoiceAmount]
    - DueDate: [DueDate]
    - Priority: "High"

' Get Next Item from Queue
Work Queues::Get Next Item
  Queue Name: "InvoiceProcessing"
  Output: QueueItem

' Mark Item Complete
Work Queues::Mark Completed
  Queue Item ID: [QueueItem.ID]
  
' Mark Item Exception
Work Queues::Mark Exception
  Queue Item ID: [QueueItem.ID]
  Exception Type: "Business Exception"
  Exception Reason: "Invalid invoice format"
```

## Scheduling và Resource Management

### Schedule Configuration
```xml
<Schedule Name="DailyInvoiceProcessing">
  <Trigger Type="Time">
    <Time>08:00:00</Time>
    <Days>Monday,Tuesday,Wednesday,Thursday,Friday</Days>
  </Trigger>
  
  <Process Name="ProcessInvoices" />
  
  <Resources>
    <Resource Name="Robot1" />
    <Resource Name="Robot2" />
  </Resources>
  
  <Parameters>
    <Parameter Name="ProcessDate" Value="{Today}" />
    <Parameter Name="BatchSize" Value="50" />
  </Parameters>
</Schedule>
```

### Resource Pool Management
```xml
<ResourcePool Name="FinanceRobots">
  <Resources>
    <Resource Name="FinanceBot01" 
              Machine="FINANCE-VM-01"
              Status="Available" />
    <Resource Name="FinanceBot02"
              Machine="FINANCE-VM-02" 
              Status="Available" />
  </Resources>
  
  <LoadBalancing Type="RoundRobin" />
  <MaxConcurrentProcesses>2</MaxConcurrentProcesses>
</ResourcePool>
```

## Security và Compliance

### Credential Management
```vb
' Store Credential
Credentials::Set Credential
  Credential Name: "SAP_Login"
  Username: "sap_user"
  Password: [SecurePassword]
  
' Retrieve Credential
Credentials::Get Credential
  Credential Name: "SAP_Login"
  Output Username: SAPUser
  Output Password: SAPPassword

' Use Credential in Process
Navigate::Navigate
  Application Name: "SAP"
  Element Name: "UsernameField"
  Action: "Set Text"
  Text: [SAPUser]
  
Navigate::Navigate
  Application Name: "SAP"
  Element Name: "PasswordField"  
  Action: "Set Text"
  Text: [SAPPassword]
```

### Audit Trail
```vb
' Log Process Start
Utility - General::Log Line
  Text: "Process started for invoice: " & [InvoiceNumber]
  Log Type: "Information"

' Log Business Decision
Decision::Decision
  Expression: "[Amount] > 10000"
  
  True Path:
    Utility - General::Log Line
      Text: "High value invoice requires approval: " & [InvoiceNumber]
      Log Type: "Warning"
      
  False Path:
    Utility - General::Log Line
      Text: "Standard processing for invoice: " & [InvoiceNumber]
      Log Type: "Information"
```

## Performance Monitoring

### Process Analytics
```sql
-- Process Performance Query
SELECT 
  p.ProcessName,
  AVG(s.Duration) as AvgDuration,
  COUNT(*) as ExecutionCount,
  SUM(CASE WHEN s.Status = 'Completed' THEN 1 ELSE 0 END) as SuccessCount,
  (SUM(CASE WHEN s.Status = 'Completed' THEN 1 ELSE 0 END) * 100.0 / COUNT(*)) as SuccessRate
FROM BPASession s
JOIN BPAProcess p ON s.ProcessId = p.ProcessId
WHERE s.StartDateTime >= DATEADD(day, -30, GETDATE())
GROUP BY p.ProcessName
ORDER BY ExecutionCount DESC
```

### Resource Utilization
```sql
-- Resource Utilization Report
SELECT 
  r.ResourceName,
  COUNT(s.SessionId) as TotalSessions,
  AVG(s.Duration) as AvgSessionDuration,
  SUM(s.Duration) as TotalRuntime,
  MAX(s.StartDateTime) as LastExecution
FROM BPAResource r
LEFT JOIN BPASession s ON r.ResourceId = s.ResourceId
WHERE s.StartDateTime >= DATEADD(day, -7, GETDATE())
GROUP BY r.ResourceName, r.ResourceId
ORDER BY TotalRuntime DESC
```

## Integration Capabilities

### Web Services Integration
```vb
' SOAP Web Service Call
Web Services::Call Web Service
  WSDL URL: "http://api.company.com/service?wsdl"
  Method Name: "GetCustomerInfo"
  Parameters:
    - customerId: [CustomerID]
  Output: CustomerData

' REST API Call
HTTP::Send Request
  Method: "POST"
  URL: "https://api.company.com/orders"
  Headers:
    - Content-Type: "application/json"
    - Authorization: "Bearer " & [APIToken]
  Body: "{""customerId"": """ & [CustomerID] & """, ""amount"": " & [Amount] & "}"
  Output: APIResponse
```

### Email Integration
```vb
' Send Email with Attachment
Email - SMTP::Send Email
  SMTP Server: "smtp.company.com"
  Port: 587
  Username: [EmailUser]
  Password: [EmailPassword]
  From: "automation@company.com"
  To: [RecipientEmail]
  Subject: "Invoice Processing Complete"
  Body: "Invoice " & [InvoiceNumber] & " has been processed successfully."
  Attachments: "C:\Reports\invoice_" & [InvoiceNumber] & ".pdf"

' Read Email with IMAP
Email - IMAP::Get Emails
  IMAP Server: "imap.company.com"
  Port: 993
  Username: [EmailUser]
  Password: [EmailPassword]
  Folder: "INBOX"
  Filter: "UNSEEN"
  Output: EmailCollection
```

## Deployment Strategies

### Environment Management
```xml
<Environments>
  <Environment Name="Development">
    <DatabaseServer>DEV-DB-01</DatabaseServer>
    <ApplicationServer>DEV-APP-01</ApplicationServer>
    <Credentials>
      <Credential Name="TestDB" Value="dev_connection_string" />
    </Credentials>
  </Environment>
  
  <Environment Name="Production">
    <DatabaseServer>PROD-DB-01</DatabaseServer>
    <ApplicationServer>PROD-APP-01</ApplicationServer>
    <Credentials>
      <Credential Name="ProdDB" Value="prod_connection_string" />
    </Credentials>
  </Environment>
</Environments>
```

### Release Management
```vb
' Export Process
Process::Export Process
  Process Name: "InvoiceProcessing"
  Export Path: "C:\Releases\InvoiceProcessing_v2.1.bprelease"
  Include Dependencies: True

' Import Process  
Process::Import Process
  Import Path: "C:\Releases\InvoiceProcessing_v2.1.bprelease"
  Target Environment: "Production"
  Overwrite Existing: True
```

## Best Practices

### Error Handling Strategy
```vb
' Global Exception Handler
Exception::Try
  Exception Type: "System Exception"
  
  ' Main process flow
  
Exception::Catch
  Exception Type: "System Exception"
  
  ' Capture screenshot
  Utility - Environment::Take Screenshot
    File Path: "C:\Logs\Screenshots\Error_" & [Now()] & ".png"
    
  ' Log detailed error
  Utility - File Management::Write Text File
    File Path: "C:\Logs\ProcessErrors.log"
    Text: [Now()] & " - " & [Exception.Message] & " - " & [Exception.Detail]
    Append: True
    
  ' Send alert email
  Email - SMTP::Send Email
    To: "support@company.com"
    Subject: "Process Exception Alert"
    Body: "Process failed with error: " & [Exception.Message]
```

### Performance Optimization
```vb
' Efficient Element Identification
Navigate::Navigate
  Application Name: "WebApp"
  Element Name: "SubmitButton"
  Action: "Click"
  Wait Before: 0
  Wait After: 1
  Timeout: 5

' Batch Processing
Loop Start::Loop Start - Collection
  Collection Name: "WorkItems"
  Loop Collection Name: "CurrentItem"
  
  ' Process item
  
  ' Commit every 10 items
  Decision::Decision
    Expression: "[Loop Counter] Mod 10 = 0"
    
    True Path:
      ' Commit transaction
      Database::Commit Transaction
        Connection Handle: [DBConnection]

Loop End::Loop End
```

Blue Prism cung cấp một nền tảng RPA enterprise mạnh mẽ với focus vào security, governance và scalability, phù hợp cho các tổ chức lớn có yêu cầu cao về compliance và audit.