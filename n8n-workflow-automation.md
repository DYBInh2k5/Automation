# n8n - Workflow Automation Platform

## Tổng quan về n8n

n8n là một công cụ automation workflow mã nguồn mở, cho phép kết nối các ứng dụng và dịch vụ khác nhau để tự động hóa các quy trình công việc.

## Đặc điểm chính

### 1. Visual Workflow Builder
- Giao diện kéo thả trực quan
- Node-based workflow design
- Real-time execution preview
- Debug mode với step-by-step execution

### 2. Tích hợp rộng rãi
- Hơn 400+ integrations sẵn có
- REST API support
- Webhook triggers
- Custom nodes development

### 3. Deployment Options
- **Self-hosted**: Triển khai trên server riêng
- **n8n Cloud**: Managed service
- **Docker**: Container deployment
- **npm**: Local installation

## Cài đặt và Triển khai

### Docker Deployment
```bash
# Chạy n8n với Docker
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n

# Docker Compose
version: '3.8'
services:
  n8n:
    image: n8nio/n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=password
    volumes:
      - ~/.n8n:/home/node/.n8n
```

### npm Installation
```bash
# Cài đặt global
npm install n8n -g

# Chạy n8n
n8n start

# Hoặc với npx
npx n8n
```

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: n8n
spec:
  replicas: 1
  selector:
    matchLabels:
      app: n8n
  template:
    metadata:
      labels:
        app: n8n
    spec:
      containers:
      - name: n8n
        image: n8nio/n8n
        ports:
        - containerPort: 5678
        env:
        - name: N8N_HOST
          value: "0.0.0.0"
        - name: N8N_PORT
          value: "5678"
```

## Các Node phổ biến

### Trigger Nodes
- **Webhook**: HTTP requests
- **Cron**: Scheduled execution
- **Manual Trigger**: Manual execution
- **File Trigger**: File system changes
- **Email Trigger**: IMAP email monitoring

### Core Nodes
- **HTTP Request**: API calls
- **Set**: Data manipulation
- **IF**: Conditional logic
- **Switch**: Multiple conditions
- **Merge**: Combine data streams
- **Split In Batches**: Process large datasets

### Service Integrations
- **Gmail**: Email automation
- **Slack**: Team communication
- **Google Sheets**: Spreadsheet operations
- **Notion**: Database operations
- **Airtable**: Database management
- **Telegram**: Bot automation

## Ví dụ Workflow

### 1. Email to Slack Notification
```json
{
  "nodes": [
    {
      "name": "Gmail Trigger",
      "type": "n8n-nodes-base.gmailTrigger",
      "parameters": {
        "pollTimes": {
          "item": [
            {
              "mode": "everyMinute"
            }
          ]
        }
      }
    },
    {
      "name": "Slack",
      "type": "n8n-nodes-base.slack",
      "parameters": {
        "operation": "postMessage",
        "channel": "#general",
        "text": "New email: {{$node['Gmail Trigger'].json['subject']}}"
      }
    }
  ],
  "connections": {
    "Gmail Trigger": {
      "main": [
        [
          {
            "node": "Slack",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

### 2. Data Sync Workflow
```json
{
  "nodes": [
    {
      "name": "Cron Trigger",
      "type": "n8n-nodes-base.cron",
      "parameters": {
        "triggerTimes": {
          "item": [
            {
              "hour": 9,
              "minute": 0
            }
          ]
        }
      }
    },
    {
      "name": "Google Sheets",
      "type": "n8n-nodes-base.googleSheets",
      "parameters": {
        "operation": "read",
        "sheetId": "your-sheet-id",
        "range": "A:Z"
      }
    },
    {
      "name": "Airtable",
      "type": "n8n-nodes-base.airtable",
      "parameters": {
        "operation": "create",
        "application": "your-base-id",
        "table": "your-table-name"
      }
    }
  ]
}
```

## Best Practices

### 1. Workflow Design
- Sử dụng descriptive node names
- Thêm notes cho complex logic
- Implement error handling
- Test với sample data

### 2. Performance Optimization
- Sử dụng Split In Batches cho large datasets
- Implement proper filtering
- Cache expensive operations
- Monitor execution times

### 3. Security
- Sử dụng credentials store
- Enable authentication
- Secure webhook endpoints
- Regular backup workflows

### 4. Monitoring
- Set up execution logging
- Monitor failed executions
- Implement alerting
- Track performance metrics

## Environment Configuration

### Environment Variables
```bash
# Basic Configuration
N8N_HOST=0.0.0.0
N8N_PORT=5678
N8N_PROTOCOL=https
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=secure_password

# Database Configuration
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=localhost
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n_user
DB_POSTGRESDB_PASSWORD=n8n_password

# Execution Configuration
EXECUTIONS_PROCESS=main
EXECUTIONS_MODE=regular
EXECUTIONS_TIMEOUT=3600
EXECUTIONS_MAX_TIMEOUT=3600

# Webhook Configuration
WEBHOOK_URL=https://your-domain.com/
N8N_PAYLOAD_SIZE_MAX=16
```

## Custom Nodes Development

### Node Structure
```typescript
import {
  IExecuteFunctions,
  INodeExecutionData,
  INodeType,
  INodeTypeDescription,
} from 'n8n-workflow';

export class CustomNode implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'Custom Node',
    name: 'customNode',
    group: ['transform'],
    version: 1,
    description: 'Custom node description',
    defaults: {
      name: 'Custom Node',
    },
    inputs: ['main'],
    outputs: ['main'],
    properties: [
      {
        displayName: 'Parameter',
        name: 'parameter',
        type: 'string',
        default: '',
        description: 'Parameter description',
      },
    ],
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];

    for (let i = 0; i < items.length; i++) {
      const parameter = this.getNodeParameter('parameter', i) as string;
      
      // Custom logic here
      const newItem: INodeExecutionData = {
        json: {
          ...items[i].json,
          processed: true,
          parameter,
        },
      };

      returnData.push(newItem);
    }

    return [returnData];
  }
}
```

## Troubleshooting

### Common Issues
1. **Memory Issues**: Increase Node.js memory limit
2. **Timeout Errors**: Adjust execution timeout
3. **Database Connections**: Check connection strings
4. **Webhook Issues**: Verify URL accessibility

### Debug Mode
```bash
# Enable debug logging
N8N_LOG_LEVEL=debug n8n start

# Specific module debugging
DEBUG=n8n* n8n start
```

## Tích hợp với CI/CD

### GitHub Actions
```yaml
name: Deploy n8n Workflows
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Deploy to n8n
        run: |
          curl -X POST "$N8N_WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -d @workflows/workflow.json
```

## Monitoring và Logging

### Prometheus Metrics
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'n8n'
    static_configs:
      - targets: ['localhost:5678']
    metrics_path: '/metrics'
```

### Log Configuration
```json
{
  "level": "info",
  "output": "console",
  "file": {
    "location": "logs/n8n.log",
    "maxsize": "10m",
    "maxFiles": 5
  }
}
```

n8n là một công cụ mạnh mẽ cho workflow automation với khả năng mở rộng cao và cộng đồng phát triển tích cực.