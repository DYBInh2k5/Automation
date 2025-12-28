# Microsoft Power Automate - Low-Code Automation Platform

## Tổng quan về Power Automate

Microsoft Power Automate (trước đây là Microsoft Flow) là nền tảng automation low-code/no-code, cho phép tạo workflows tự động giữa các ứng dụng và dịch vụ Microsoft và bên thứ ba.

## Các loại Flow

### 1. Cloud Flows
- **Automated flows**: Trigger bởi events
- **Instant flows**: Trigger thủ công
- **Scheduled flows**: Chạy theo lịch

### 2. Desktop Flows (RPA)
- **Attended flows**: Cần sự tương tác của người dùng
- **Unattended flows**: Chạy tự động hoàn toàn

### 3. Business Process Flows
- Guided workflows cho business processes
- Multi-stage approval processes

## Connectors phổ biến

### Microsoft Services
- **SharePoint**: Document management
- **Outlook**: Email automation
- **Teams**: Collaboration workflows
- **OneDrive**: File operations
- **Excel**: Spreadsheet automation
- **Dynamics 365**: CRM operations
- **Azure**: Cloud services integration

### Third-party Services
- **Salesforce**: CRM integration
- **Twitter**: Social media automation
- **Dropbox**: File storage
- **Slack**: Team communication
- **Google Services**: Gmail, Drive, Sheets
- **ServiceNow**: IT service management

## Ví dụ Flow cơ bản

### 1. Email Approval Workflow
```json
{
  "definition": {
    "triggers": {
      "When_a_new_email_arrives": {
        "type": "ApiConnection",
        "inputs": {
          "host": {
            "connectionName": "shared_outlook",
            "operationId": "OnNewEmail"
          },
          "parameters": {
            "folderPath": "Inbox",
            "subjectFilter": "Approval Request"
          }
        }
      }
    },
    "actions": {
      "Start_an_approval": {
        "type": "ApiConnection",
        "inputs": {
          "host": {
            "connectionName": "shared_approvals",
            "operationId": "StartApproval"
          },
          "parameters": {
            "approvalType": "Basic",
            "title": "@triggerBody()?['Subject']",
            "assignedTo": "manager@company.com",
            "details": "@triggerBody()?['Body']"
          }
        }
      },
      "Condition": {
        "type": "If",
        "expression": {
          "equals": [
            "@body('Start_an_approval')?['outcome']",
            "Approve"
          ]
        },
        "actions": {
          "Send_approval_email": {
            "type": "ApiConnection",
            "inputs": {
              "host": {
                "connectionName": "shared_outlook",
                "operationId": "SendEmail"
              },
              "parameters": {
                "emailMessage": {
                  "To": "@triggerBody()?['From']",
                  "Subject": "Request Approved",
                  "Body": "Your request has been approved."
                }
              }
            }
          }
        },
        "else": {
          "actions": {
            "Send_rejection_email": {
              "type": "ApiConnection",
              "inputs": {
                "host": {
                  "connectionName": "shared_outlook",
                  "operationId": "SendEmail"
                },
                "parameters": {
                  "emailMessage": {
                    "To": "@triggerBody()?['From']",
                    "Subject": "Request Rejected",
                    "Body": "Your request has been rejected."
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### 2. SharePoint to Teams Notification
```json
{
  "definition": {
    "triggers": {
      "When_an_item_is_created": {
        "type": "ApiConnection",
        "inputs": {
          "host": {
            "connectionName": "shared_sharepointonline",
            "operationId": "GetOnNewItems"
          },
          "parameters": {
            "dataset": "https://company.sharepoint.com/sites/projects",
            "table": "Tasks"
          }
        }
      }
    },
    "actions": {
      "Post_message_in_a_chat_or_channel": {
        "type": "ApiConnection",
        "inputs": {
          "host": {
            "connectionName": "shared_teams",
            "operationId": "PostMessageToConversation"
          },
          "parameters": {
            "poster": "Flow bot",
            "location": "Channel",
            "teamsId": "team-id",
            "channelId": "channel-id",
            "rootMessage": {
              "body": {
                "content": "New task created: @{triggerBody()?['Title']} assigned to @{triggerBody()?['AssignedTo']?['DisplayName']}"
              }
            }
          }
        }
      }
    }
  }
}
```

## Power Automate Desktop (RPA)

### Cài đặt và Setup
```powershell
# Download từ Microsoft Store hoặc
# https://powerautomate.microsoft.com/desktop/

# Kiểm tra version
Get-WmiObject -Class Win32_Product | Where-Object {$_.Name -like "*Power Automate*"}
```

### Basic Desktop Flow Actions

#### 1. Web Automation
```
# Launch browser
Web.LaunchBrowser Url: "https://example.com" Browser: Chrome

# Fill text field
Web.PopulateTextField Browser: %Browser% Selector: "#username" Text: "user@company.com"

# Click button
Web.ClickElement Browser: %Browser% Selector: "#login-button"

# Extract data
Web.ExtractData Browser: %Browser% Selector: ".data-table" ExtractedData: %ExtractedData%
```

#### 2. Excel Automation
```
# Open Excel file
Excel.LaunchExcel Visible: True Instance: %ExcelInstance%
Excel.OpenWorkbook Instance: %ExcelInstance% DocumentPath: "C:\\Data\\report.xlsx"

# Read data
Excel.ReadFromExcel Instance: %ExcelInstance% StartColumn: 1 StartRow: 1 EndColumn: 5 EndRow: 100 ReadAsText: False ExcelData: %ExcelData%

# Write data
Excel.WriteToExcel Instance: %ExcelInstance% Column: 6 Row: %CurrentRow% ValueToWrite: %CalculatedValue%

# Save and close
Excel.SaveExcel Instance: %ExcelInstance%
Excel.CloseExcel Instance: %ExcelInstance%
```

#### 3. File Operations
```
# Copy files
File.Copy SourceFilePath: "C:\\Source\\*.pdf" DestinationFolder: "C:\\Destination\\"

# Rename files
File.Rename FilePath: "C:\\Files\\old_name.txt" NewName: "new_name.txt"

# Create folder
Folder.Create FolderPath: "C:\\Reports\\%DateTime.Now.ToString('yyyy-MM-dd')%"
```

### Advanced Desktop Flow Features

#### Variables và Expressions
```
# Set variable
SET NewVariable TO "Hello World"

# DateTime operations
SET CurrentDate TO %DateTime.Now%
SET FormattedDate TO %DateTime.Now.ToString('yyyy-MM-dd')%

# String operations
SET UpperText TO %OriginalText.ToUpper()%
SET SubString TO %OriginalText.Substring(0, 10)%

# Mathematical operations
SET Result TO %Number1 + Number2%
SET Percentage TO %Value / Total * 100%
```

#### Loops và Conditions
```
# For each loop
FOR EACH Item IN %ListVariable%
    # Actions here
    Display.ShowMessage Message: %Item%
END

# While loop
WHILE %Counter% < 10
    SET Counter TO %Counter + 1%
    # Actions here
END

# If condition
IF %Variable% = "Expected Value" THEN
    # True actions
ELSE
    # False actions
END
```

## Custom Connectors

### Creating Custom Connector
```json
{
  "swagger": "2.0",
  "info": {
    "title": "Custom API Connector",
    "description": "Custom connector for internal API",
    "version": "1.0"
  },
  "host": "api.company.com",
  "basePath": "/v1",
  "schemes": ["https"],
  "consumes": ["application/json"],
  "produces": ["application/json"],
  "paths": {
    "/users": {
      "get": {
        "summary": "Get Users",
        "description": "Retrieves list of users",
        "operationId": "GetUsers",
        "parameters": [
          {
            "name": "limit",
            "in": "query",
            "type": "integer",
            "description": "Number of users to return"
          }
        ],
        "responses": {
          "200": {
            "description": "Success",
            "schema": {
              "type": "array",
              "items": {
                "$ref": "#/definitions/User"
              }
            }
          }
        }
      }
    }
  },
  "definitions": {
    "User": {
      "type": "object",
      "properties": {
        "id": {"type": "integer"},
        "name": {"type": "string"},
        "email": {"type": "string"}
      }
    }
  }
}
```

### Authentication Configuration
```json
{
  "authType": "oauth2",
  "oauth2Settings": {
    "identityProvider": "oauth2generic",
    "clientId": "your-client-id",
    "scopes": ["read", "write"],
    "redirectMode": "Global",
    "redirectUrl": "https://global.consent.azure-apim.net/redirect",
    "authorizationUrl": "https://api.company.com/oauth/authorize",
    "tokenUrl": "https://api.company.com/oauth/token",
    "refreshUrl": "https://api.company.com/oauth/refresh"
  }
}
```

## Power Platform Integration

### Power Apps Integration
```javascript
// Trigger flow from Power Apps
PowerAutomate.Run(
    "FlowName",
    {
        inputParameter1: TextInput1.Text,
        inputParameter2: Dropdown1.Selected.Value
    }
)

// Handle flow response
If(
    PowerAutomate.Run("FlowName", {param: "value"}).result = "success",
    Navigate(SuccessScreen),
    Navigate(ErrorScreen)
)
```

### Power BI Integration
```json
{
  "triggers": {
    "Recurrence": {
      "type": "Recurrence",
      "recurrence": {
        "frequency": "Day",
        "interval": 1,
        "schedule": {
          "hours": ["6"]
        }
      }
    }
  },
  "actions": {
    "Refresh_dataset": {
      "type": "ApiConnection",
      "inputs": {
        "host": {
          "connectionName": "shared_powerbi",
          "operationId": "RefreshDataset"
        },
        "parameters": {
          "workspace": "workspace-id",
          "dataset": "dataset-id"
        }
      }
    }
  }
}
```

## Error Handling và Monitoring

### Configure Run After Settings
```json
{
  "actions": {
    "Try_action": {
      "type": "ApiConnection",
      "inputs": {
        "host": {
          "connectionName": "shared_connector",
          "operationId": "Operation"
        }
      }
    },
    "Handle_error": {
      "type": "Compose",
      "inputs": "Error occurred: @{result('Try_action')['error']['message']}",
      "runAfter": {
        "Try_action": ["Failed", "TimedOut"]
      }
    },
    "Send_error_notification": {
      "type": "ApiConnection",
      "inputs": {
        "host": {
          "connectionName": "shared_outlook",
          "operationId": "SendEmail"
        },
        "parameters": {
          "emailMessage": {
            "To": "admin@company.com",
            "Subject": "Flow Error",
            "Body": "@outputs('Handle_error')"
          }
        }
      },
      "runAfter": {
        "Handle_error": ["Succeeded"]
      }
    }
  }
}
```

### Retry Policies
```json
{
  "retryPolicy": {
    "type": "exponential",
    "count": 3,
    "interval": "PT10S",
    "maximumInterval": "PT1M"
  }
}
```

## Performance Optimization

### Parallel Branches
```json
{
  "actions": {
    "Parallel_branch": {
      "type": "Parallel",
      "branches": {
        "Branch_1": {
          "actions": {
            "Action_1": {
              "type": "ApiConnection"
            }
          }
        },
        "Branch_2": {
          "actions": {
            "Action_2": {
              "type": "ApiConnection"
            }
          }
        }
      }
    }
  }
}
```

### Pagination
```json
{
  "actions": {
    "Get_items": {
      "type": "ApiConnection",
      "inputs": {
        "host": {
          "connectionName": "shared_sharepointonline",
          "operationId": "GetItems"
        },
        "parameters": {
          "dataset": "site-url",
          "table": "list-name"
        }
      },
      "runtimeConfiguration": {
        "paginationPolicy": {
          "minimumItemCount": 5000
        }
      }
    }
  }
}
```

## Security Best Practices

### Data Loss Prevention (DLP)
```json
{
  "displayName": "Corporate DLP Policy",
  "defaultConnectorsClassification": "General",
  "connectorGroups": [
    {
      "classification": "Confidential",
      "connectors": [
        {
          "id": "/providers/Microsoft.PowerApps/apis/shared_sharepointonline"
        },
        {
          "id": "/providers/Microsoft.PowerApps/apis/shared_sql"
        }
      ]
    },
    {
      "classification": "General",
      "connectors": [
        {
          "id": "/providers/Microsoft.PowerApps/apis/shared_twitter"
        }
      ]
    }
  ]
}
```

### Environment Variables
```json
{
  "environmentVariables": {
    "API_BASE_URL": {
      "value": "https://api.company.com",
      "type": "string"
    },
    "DATABASE_CONNECTION": {
      "value": "connection-string",
      "type": "secret"
    }
  }
}
```

## Deployment và ALM

### Solution Export
```powershell
# Export solution
pac solution export --path "solution.zip" --name "MySolution" --managed

# Import solution
pac solution import --path "solution.zip" --force-overwrite
```

### CI/CD Pipeline
```yaml
# Azure DevOps Pipeline
trigger:
  branches:
    include:
    - main

pool:
  vmImage: 'windows-latest'

steps:
- task: PowerPlatformToolInstaller@0
  displayName: 'Install Power Platform Tools'

- task: PowerPlatformExportSolution@0
  displayName: 'Export Solution'
  inputs:
    authenticationType: 'PowerPlatformSPN'
    PowerPlatformSPN: '$(PowerPlatformConnection)'
    SolutionName: '$(SolutionName)'
    SolutionOutputFile: '$(Build.ArtifactStagingDirectory)\$(SolutionName).zip'

- task: PowerPlatformImportSolution@0
  displayName: 'Import Solution'
  inputs:
    authenticationType: 'PowerPlatformSPN'
    PowerPlatformSPN: '$(TargetEnvironmentConnection)'
    SolutionInputFile: '$(Build.ArtifactStagingDirectory)\$(SolutionName).zip'
```

Power Automate cung cấp một nền tảng automation mạnh mẽ với khả năng tích hợp sâu vào hệ sinh thái Microsoft và nhiều dịch vụ bên thứ ba.