# UiPath - Robotic Process Automation Platform

## Tổng quan về UiPath

UiPath là nền tảng RPA (Robotic Process Automation) hàng đầu, cho phép tự động hóa các quy trình kinh doanh thông qua việc mô phỏng hành động của con người trên giao diện người dùng.

## Kiến trúc UiPath

### Core Components
- **UiPath Studio**: Development environment
- **UiPath Robot**: Execution agent
- **UiPath Orchestrator**: Management và monitoring platform
- **UiPath Assistant**: Desktop application cho attended automation

### Robot Types
- **Attended Robot**: Làm việc cùng với người dùng
- **Unattended Robot**: Chạy tự động không cần giám sát
- **Development Robot**: Dành cho phát triển và testing

## UiPath Studio

### Project Types
- **Process**: Standard automation workflows
- **Library**: Reusable components
- **Orchestrator Process**: Cloud-ready processes
- **Background Process**: Unattended automation
- **Robotic Enterprise Framework (REFramework)**: Enterprise template

### Basic Activities

#### UI Automation
```csharp
// Click activity
Click "Button1"

// Type Into activity
TypeInto "TextBox1", "Hello World"

// Get Text activity
GetText "Label1", out string labelText

// Select Item activity
SelectItem "ComboBox1", "Option 1"
```

#### Data Manipulation
```csharp
// Read Excel
ReadRange "Sheet1", "A1:C10", out DataTable dt

// Write Excel
WriteRange dt, "Sheet2", "A1"

// Filter DataTable
FilterDataTable dt, "Age > 25", out DataTable filteredDt

// Join DataTable
JoinDataTable dt1, dt2, "ID", "ID", out DataTable joinedDt
```

#### File Operations
```csharp
// Copy File
CopyFile "C:\Source\file.txt", "C:\Destination\file.txt"

// Move File
MoveFile "C:\Source\file.txt", "C:\Archive\file.txt"

// Delete File
DeleteFile "C:\Temp\file.txt"

// Create Directory
CreateDirectory "C:\NewFolder"
```

### Advanced Activities

#### Web Automation
```csharp
// Open Browser
OpenBrowser "https://example.com", BrowserType.Chrome, out Browser browser

// Navigate To
NavigateTo browser, "https://example.com/login"

// Extract Structured Data
ExtractData browser, "table", out DataTable webData

// Download File
DownloadFile browser, "https://example.com/file.pdf", "C:\Downloads\"
```

#### Email Automation
```csharp
// Get Outlook Mail Messages
GetOutlookMailMessages "Inbox", 10, out List<MailMessage> emails

// Send Outlook Mail Message
SendOutlookMailMessage "recipient@company.com", "Subject", "Body", attachments

// Get IMAP Mail Messages
GetIMAPMailMessages "imap.gmail.com", 993, "username", "password", "INBOX", out List<MailMessage> imapEmails
```

#### Database Operations
```csharp
// Execute Query
ExecuteQuery "SELECT * FROM Users WHERE Active = 1", connectionString, out DataTable result

// Execute Non Query
ExecuteNonQuery "UPDATE Users SET LastLogin = GETDATE() WHERE ID = 1", connectionString

// Insert Data Table
InsertDataTable dt, "Users", connectionString
```

## Workflow Development

### Sequence Workflow
```xml
<Sequence DisplayName="Main Sequence">
  <Sequence.Variables>
    <Variable x:TypeArguments="x:String" Name="userName" />
    <Variable x:TypeArguments="x:Int32" Name="userAge" />
  </Sequence.Variables>
  
  <ui:TypeInto Text="admin" Target="{x:Null}" DisplayName="Type Username">
    <ui:TypeInto.Target>
      <ui:Target ClippingRegion="{x:Null}" Element="{x:Null}" Id="username_field" />
    </ui:TypeInto.Target>
  </ui:TypeInto>
  
  <ui:Click ClickType="CLICK_SINGLE" DisplayName="Click Login">
    <ui:Click.Target>
      <ui:Target ClippingRegion="{x:Null}" Element="{x:Null}" Id="login_button" />
    </ui:Click.Target>
  </ui:Click>
</Sequence>
```

### Flowchart Workflow
```xml
<Flowchart DisplayName="Decision Flowchart">
  <FlowDecision x:Name="CheckAge" DisplayName="Age > 18?" Condition="[userAge > 18]">
    <FlowDecision.True>
      <FlowStep x:Name="ProcessAdult">
        <ui:MessageBox Caption="Info" Text="Processing adult user" />
      </FlowStep>
    </FlowDecision.True>
    <FlowDecision.False>
      <FlowStep x:Name="ProcessMinor">
        <ui:MessageBox Caption="Info" Text="Processing minor user" />
      </FlowStep>
    </FlowDecision.False>
  </FlowDecision>
</Flowchart>
```

### State Machine Workflow
```xml
<StateMachine DisplayName="Order Processing State Machine">
  <State DisplayName="Initial State" x:Name="InitialState">
    <State.Entry>
      <ui:LogMessage Level="Info" Message="Starting order processing" />
    </State.Entry>
    <State.Transitions>
      <Transition DisplayName="To Validation" To="ValidationState">
        <Transition.Condition>
          <InArgument x:TypeArguments="x:Boolean">[orderReceived]</InArgument>
        </Transition.Condition>
      </Transition>
    </State.Transitions>
  </State>
  
  <State DisplayName="Validation State" x:Name="ValidationState">
    <State.Entry>
      <ui:LogMessage Level="Info" Message="Validating order" />
    </State.Entry>
  </State>
</StateMachine>
```

## REFramework Implementation

### Main.xaml Structure
```xml
<Sequence DisplayName="Main">
  <TryCatch DisplayName="Main Try Catch">
    <TryCatch.Try>
      <Sequence DisplayName="Main Sequence">
        <!-- Initialization -->
        <ui:InvokeWorkflowFile WorkflowFileName="Framework\InitiAllSettings.xaml" />
        
        <!-- Get Transaction Data -->
        <ui:InvokeWorkflowFile WorkflowFileName="Framework\GetTransactionData.xaml" />
        
        <!-- Process Transaction -->
        <ui:InvokeWorkflowFile WorkflowFileName="Framework\Process.xaml" />
        
        <!-- End Process -->
        <ui:InvokeWorkflowFile WorkflowFileName="Framework\EndProcess.xaml" />
      </Sequence>
    </TryCatch.Try>
    <TryCatch.Catches>
      <Catch x:TypeArguments="s:Exception">
        <ActivityAction x:TypeArguments="s:Exception">
          <ActivityAction.Argument>
            <DelegateInArgument x:TypeArguments="s:Exception" Name="exception" />
          </ActivityAction.Argument>
          <ui:LogMessage Level="Error" Message="[exception.Message]" />
        </ActivityAction>
      </Catch>
    </TryCatch.Catches>
  </TryCatch>
</Sequence>
```

### Config.xlsx Structure
| Name | Value | Description |
|------|-------|-------------|
| LogLevel | Information | Logging level |
| MaxRetryNumber | 3 | Maximum retry attempts |
| RetryNumberGetTransactionData | 1 | Retry for data retrieval |
| RetryNumberSetTransactionStatus | 1 | Retry for status update |

### Exception Handling
```csharp
// Business Rule Exception
throw new BusinessRuleException("Invalid customer data")

// Application Exception  
throw new ApplicationException("System unavailable")

// System Exception (handled automatically by framework)
```

## UiPath Orchestrator

### Queue Management
```csharp
// Add Queue Item
AddQueueItem "ProcessQueue", new Dictionary<string, object> {
    {"CustomerID", "12345"},
    {"OrderAmount", 1000.50},
    {"Priority", "High"}
}

// Get Transaction Item
GetTransactionItem "ProcessQueue", out QueueItem transactionItem

// Set Transaction Status
SetTransactionStatus transactionItem, TransactionStatus.Successful, "Processed successfully"
```

### Asset Management
```csharp
// Get Credential Asset
GetCredential "DatabaseCredentials", out string username, out SecureString password

// Get Text Asset
GetAsset "APIEndpoint", out string apiUrl

// Get Bool Asset
GetAsset "EnableLogging", out bool loggingEnabled
```

### Process Scheduling
```json
{
  "Name": "Daily Report Generation",
  "ProcessKey": "ReportProcess",
  "StartProcessCron": "0 8 * * MON-FRI",
  "TimeZoneId": "UTC",
  "RuntimeType": "Unattended",
  "InputArguments": {
    "ReportDate": "#{DateTime.Now.ToString('yyyy-MM-dd')}",
    "EmailRecipients": "reports@company.com"
  }
}
```

## Custom Activities Development

### Activity Structure
```csharp
[Designer(typeof(CustomActivityDesigner))]
public sealed class CustomActivity : CodeActivity<string>
{
    [Category("Input")]
    [RequiredArgument]
    public InArgument<string> InputText { get; set; }
    
    [Category("Options")]
    public InArgument<bool> CaseSensitive { get; set; }
    
    protected override string Execute(CodeActivityContext context)
    {
        string input = InputText.Get(context);
        bool caseSensitive = CaseSensitive.Get(context);
        
        // Custom logic here
        string result = caseSensitive ? input.ToUpper() : input.ToLower();
        
        return result;
    }
}
```

### Activity Designer
```xml
<sap:ActivityDesigner x:Class="CustomActivities.CustomActivityDesigner"
                      xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                      xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
                      xmlns:sap="clr-namespace:System.Activities.Presentation;assembly=System.Activities.Presentation">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <TextBlock Grid.Row="0" Text="Custom Activity" FontWeight="Bold"/>
        <sap:WorkflowItemPresenter Grid.Row="1" Item="{Binding Path=ModelItem.Body}"/>
    </Grid>
</sap:ActivityDesigner>
```

## Testing và Debugging

### Unit Testing
```csharp
[TestMethod]
public void TestCustomActivity()
{
    // Arrange
    var activity = new CustomActivity
    {
        InputText = "Test Input",
        CaseSensitive = true
    };
    
    // Act
    var result = WorkflowInvoker.Invoke(activity);
    
    // Assert
    Assert.AreEqual("TEST INPUT", result);
}
```

### Mock Testing
```csharp
// Mock external dependencies
var mockService = new Mock<IExternalService>();
mockService.Setup(x => x.GetData(It.IsAny<string>()))
           .Returns("Mocked Data");

// Inject mock into workflow
var workflow = new TestWorkflow();
workflow.ExternalService = mockService.Object;

var result = WorkflowInvoker.Invoke(workflow);
```

## Performance Optimization

### Selector Optimization
```xml
<!-- Bad selector -->
<webctrl tag='INPUT' type='text' />

<!-- Good selector -->
<webctrl tag='INPUT' type='text' id='username' />

<!-- Dynamic selector -->
<webctrl tag='INPUT' type='text' aaname='{{variableName}}' />
```

### Memory Management
```csharp
// Dispose large objects
using (var dataTable = new DataTable())
{
    // Use dataTable
    ReadRange("Sheet1", "A1:Z1000", out dataTable);
    // dataTable is automatically disposed
}

// Clear variables when not needed
dataTable = Nothing
largeString = String.Empty
```

### Parallel Processing
```xml
<Parallel DisplayName="Parallel Processing">
  <Sequence DisplayName="Branch 1">
    <ui:InvokeWorkflowFile WorkflowFileName="ProcessBatch1.xaml" />
  </Sequence>
  <Sequence DisplayName="Branch 2">
    <ui:InvokeWorkflowFile WorkflowFileName="ProcessBatch2.xaml" />
  </Sequence>
</Parallel>
```

## Security Best Practices

### Credential Management
```csharp
// Use Orchestrator Assets for credentials
GetCredential "DatabaseCredentials", out string dbUser, out SecureString dbPassword

// Encrypt sensitive data
var encryptedData = Encrypt("sensitive data", "encryption key")

// Use Windows Credential Manager
GetSecureCredential "MyApp_Credentials", out NetworkCredential credentials
```

### Audit Logging
```csharp
// Log all critical actions
LogMessage LogLevel.Info, $"Processing transaction {transactionId}"
LogMessage LogLevel.Warn, $"Retry attempt {retryCount} for transaction {transactionId}"
LogMessage LogLevel.Error, $"Failed to process transaction {transactionId}: {exception.Message}"

// Screenshot on errors
TakeScreenshot $"Error_{DateTime.Now:yyyyMMdd_HHmmss}.png"
```

## Deployment Strategies

### Package Creation
```xml
<!-- project.json -->
{
  "name": "ProcessAutomation",
  "description": "Automated invoice processing",
  "main": "Main.xaml",
  "dependencies": {
    "UiPath.Excel.Activities": "[2.11.4]",
    "UiPath.Mail.Activities": "[1.12.3]",
    "UiPath.System.Activities": "[21.10.4]"
  },
  "webSearches": [],
  "entryPoints": [
    {
      "filePath": "Main.xaml",
      "uniqueId": "main-entry-point",
      "input": [],
      "output": []
    }
  ],
  "isTemplate": false,
  "projectVersion": "1.0.0"
}
```

### CI/CD Pipeline
```yaml
# Azure DevOps Pipeline for UiPath
trigger:
  branches:
    include:
    - main

pool:
  vmImage: 'windows-latest'

steps:
- task: UiPathPack@3
  inputs:
    versionType: 'AutoVersion'
    projectJsonPath: '$(System.DefaultWorkingDirectory)'
    outputPath: '$(Build.ArtifactStagingDirectory)'

- task: UiPathDeploy@3
  inputs:
    orchestratorConnection: 'UiPath-Orchestrator'
    packagesPath: '$(Build.ArtifactStagingDirectory)'
    folderName: 'Production'

- task: UiPathRunTests@3
  inputs:
    orchestratorConnection: 'UiPath-Orchestrator'
    testTarget: 'TestSet'
    testSet: 'RegressionTests'
```

## Monitoring và Analytics

### Process Mining Integration
```csharp
// Log process events for mining
LogProcessEvent "ProcessStart", new Dictionary<string, object> {
    {"CaseId", caseId},
    {"Timestamp", DateTime.Now},
    {"User", Environment.UserName}
}

LogProcessEvent "ActivityComplete", new Dictionary<string, object> {
    {"CaseId", caseId},
    {"Activity", "DataValidation"},
    {"Duration", activityDuration},
    {"Status", "Success"}
}
```

### Custom Dashboards
```json
{
  "dashboard": {
    "name": "Process Performance",
    "widgets": [
      {
        "type": "chart",
        "title": "Daily Transaction Volume",
        "query": "SELECT DATE(StartTime), COUNT(*) FROM Jobs WHERE ProcessName = 'InvoiceProcessing' GROUP BY DATE(StartTime)"
      },
      {
        "type": "gauge",
        "title": "Success Rate",
        "query": "SELECT (SUM(CASE WHEN State = 'Successful' THEN 1 ELSE 0 END) * 100.0 / COUNT(*)) FROM Jobs WHERE ProcessName = 'InvoiceProcessing'"
      }
    ]
  }
}
```

UiPath cung cấp một nền tảng RPA toàn diện với khả năng mở rộng cao, phù hợp cho cả automation đơn giản và các quy trình enterprise phức tạp.