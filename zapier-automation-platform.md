# Zapier - Automation Platform

## Tổng quan về Zapier

Zapier là nền tảng automation dựa trên cloud, cho phép kết nối hơn 5000+ ứng dụng web để tự động hóa các tác vụ mà không cần coding.

## Khái niệm cơ bản

### Zaps
- **Zap**: Một workflow automation
- **Trigger**: Sự kiện khởi động workflow
- **Action**: Hành động được thực hiện
- **Step**: Mỗi action trong workflow

### Trigger Types
- **Instant Triggers**: Real-time webhooks
- **Polling Triggers**: Định kỳ kiểm tra thay đổi
- **Schedule Triggers**: Chạy theo lịch

## Cấu trúc Zap

### Basic Zap Structure
```
Trigger App → Filter (Optional) → Action App
```

### Multi-Step Zap
```
Trigger → Action 1 → Action 2 → Action 3
```

### Advanced Zap with Logic
```
Trigger → Filter → Path A → Action A
                → Path B → Action B
```

## Ví dụ Zap phổ biến

### 1. Email to Slack
```yaml
Trigger: Gmail - New Email
Filter: Subject contains "urgent"
Action: Slack - Send Channel Message
  Channel: #alerts
  Message: "Urgent email from {{trigger.from}}: {{trigger.subject}}"
```

### 2. Form to CRM
```yaml
Trigger: Google Forms - New Response
Action 1: Google Sheets - Create Row
Action 2: HubSpot - Create Contact
  Email: {{trigger.email}}
  Name: {{trigger.name}}
  Source: "Web Form"
```

### 3. Social Media Automation
```yaml
Trigger: RSS - New Item in Feed
Filter: Category equals "Tech News"
Action 1: Twitter - Create Tweet
  Tweet: "{{trigger.title}} {{trigger.link}}"
Action 2: LinkedIn - Share Update
  Content: "Interesting read: {{trigger.title}}"
```

## Zapier CLI và Development

### Cài đặt Zapier CLI
```bash
# Cài đặt CLI
npm install -g zapier-platform-cli

# Đăng nhập
zapier login

# Tạo app mới
zapier init my-app --template=minimal
cd my-app
npm install
```

### App Structure
```
my-app/
├── index.js          # Entry point
├── package.json      # Dependencies
├── triggers/         # Trigger definitions
├── creates/          # Create action definitions
├── searches/         # Search action definitions
├── authentication.js # Auth configuration
└── test/            # Test files
```

### Basic Trigger Example
```javascript
// triggers/new_contact.js
const perform = async (z, bundle) => {
  const response = await z.request({
    url: 'https://api.example.com/contacts',
    params: {
      created_after: bundle.meta.timestamp || '2023-01-01'
    }
  });
  
  return response.data;
};

module.exports = {
  key: 'new_contact',
  noun: 'Contact',
  display: {
    label: 'New Contact',
    description: 'Triggers when a new contact is created.'
  },
  operation: {
    perform,
    sample: {
      id: 1,
      name: 'John Doe',
      email: 'john@example.com',
      created_at: '2023-01-01T00:00:00Z'
    }
  }
};
```

### Basic Create Action
```javascript
// creates/create_contact.js
const perform = async (z, bundle) => {
  const response = await z.request({
    method: 'POST',
    url: 'https://api.example.com/contacts',
    body: {
      name: bundle.inputData.name,
      email: bundle.inputData.email,
      phone: bundle.inputData.phone
    }
  });
  
  return response.data;
};

module.exports = {
  key: 'create_contact',
  noun: 'Contact',
  display: {
    label: 'Create Contact',
    description: 'Creates a new contact.'
  },
  operation: {
    inputFields: [
      {
        key: 'name',
        label: 'Name',
        type: 'string',
        required: true
      },
      {
        key: 'email',
        label: 'Email',
        type: 'string',
        required: true
      },
      {
        key: 'phone',
        label: 'Phone',
        type: 'string',
        required: false
      }
    ],
    perform,
    sample: {
      id: 1,
      name: 'John Doe',
      email: 'john@example.com'
    }
  }
};
```

### Authentication Configuration
```javascript
// authentication.js
module.exports = {
  type: 'oauth2',
  oauth2Config: {
    authorizeUrl: 'https://api.example.com/oauth/authorize',
    getAccessToken: {
      url: 'https://api.example.com/oauth/token',
      method: 'POST',
      body: {
        grant_type: 'authorization_code',
        client_id: '{{process.env.CLIENT_ID}}',
        client_secret: '{{process.env.CLIENT_SECRET}}',
        code: '{{bundle.inputData.code}}',
        redirect_uri: '{{bundle.inputData.redirect_uri}}'
      }
    },
    refreshAccessToken: {
      url: 'https://api.example.com/oauth/refresh',
      method: 'POST',
      body: {
        grant_type: 'refresh_token',
        refresh_token: '{{bundle.authData.refresh_token}}'
      }
    },
    scope: 'read,write'
  },
  test: {
    url: 'https://api.example.com/me'
  }
};
```

## Zapier Webhooks

### Incoming Webhooks
```javascript
// Catch Hook Trigger
const perform = (z, bundle) => {
  return [bundle.cleanedRequest];
};

module.exports = {
  key: 'catch_hook',
  noun: 'Webhook',
  display: {
    label: 'Catch Hook',
    description: 'Catches webhooks from external services.'
  },
  operation: {
    type: 'hook',
    performSubscribe: async (z, bundle) => {
      const response = await z.request({
        method: 'POST',
        url: 'https://api.example.com/webhooks',
        body: {
          url: bundle.targetUrl,
          events: ['contact.created', 'contact.updated']
        }
      });
      return response.data;
    },
    performUnsubscribe: async (z, bundle) => {
      await z.request({
        method: 'DELETE',
        url: `https://api.example.com/webhooks/${bundle.subscribeData.id}`
      });
    },
    perform
  }
};
```

### Outgoing Webhooks
```javascript
// Send webhook action
const perform = async (z, bundle) => {
  const response = await z.request({
    method: 'POST',
    url: bundle.inputData.webhook_url,
    headers: {
      'Content-Type': 'application/json'
    },
    body: {
      event: bundle.inputData.event,
      data: bundle.inputData.data,
      timestamp: new Date().toISOString()
    }
  });
  
  return { status: response.status };
};
```

## Testing và Debugging

### Unit Tests
```javascript
// test/triggers/new_contact.test.js
const zapier = require('zapier-platform-core');
const App = require('../../index');
const appTester = zapier.createAppTester(App);

describe('New Contact Trigger', () => {
  test('should fetch contacts', async () => {
    const bundle = {
      authData: {
        access_token: 'test_token'
      }
    };
    
    const results = await appTester(App.triggers.new_contact.operation.perform, bundle);
    expect(results).toBeDefined();
    expect(results.length).toBeGreaterThan(0);
  });
});
```

### Integration Tests
```javascript
// test/integration.test.js
describe('Integration Tests', () => {
  test('full workflow', async () => {
    // Test trigger
    const triggerResults = await appTester(App.triggers.new_contact.operation.perform, triggerBundle);
    
    // Test create action
    const createBundle = {
      inputData: triggerResults[0]
    };
    const createResults = await appTester(App.creates.create_contact.operation.perform, createBundle);
    
    expect(createResults.id).toBeDefined();
  });
});
```

## Advanced Features

### Code Steps
```javascript
// Custom JavaScript code in Zapier
const inputData = inputData || {};
const name = inputData.full_name || '';
const nameParts = name.split(' ');

return {
  first_name: nameParts[0] || '',
  last_name: nameParts.slice(1).join(' ') || '',
  initials: nameParts.map(part => part.charAt(0)).join('').toUpperCase()
};
```

### Formatter Functions
```javascript
// Date formatting
const moment = require('moment');
const formatted_date = moment(inputData.date).format('YYYY-MM-DD');

// Text manipulation
const cleaned_text = inputData.text
  .toLowerCase()
  .replace(/[^a-z0-9]/g, '_')
  .replace(/_+/g, '_')
  .replace(/^_|_$/g, '');

return { formatted_date, cleaned_text };
```

### Lookup Tables
```javascript
// Status mapping
const statusMap = {
  'new': 'pending',
  'in_progress': 'active',
  'completed': 'done',
  'cancelled': 'inactive'
};

const mapped_status = statusMap[inputData.status] || 'unknown';
return { mapped_status };
```

## Error Handling

### Retry Logic
```javascript
const perform = async (z, bundle) => {
  try {
    const response = await z.request({
      url: 'https://api.example.com/data',
      method: 'GET'
    });
    return response.data;
  } catch (error) {
    if (error.status === 429) {
      // Rate limited - throw HaltedError to retry
      throw new z.errors.HaltedError('Rate limited, will retry');
    }
    if (error.status >= 500) {
      // Server error - throw error to retry
      throw error;
    }
    // Client error - don't retry
    return [];
  }
};
```

### Custom Error Messages
```javascript
const perform = async (z, bundle) => {
  const response = await z.request(options);
  
  if (response.status === 404) {
    throw new z.errors.HaltedError('Resource not found. Please check your configuration.');
  }
  
  if (!response.data || response.data.length === 0) {
    throw new z.errors.HaltedError('No data available. This is expected and the Zap will continue normally.');
  }
  
  return response.data;
};
```

## Performance Optimization

### Pagination
```javascript
const perform = async (z, bundle) => {
  let allResults = [];
  let page = 1;
  let hasMore = true;
  
  while (hasMore && allResults.length < 100) {
    const response = await z.request({
      url: 'https://api.example.com/data',
      params: {
        page: page,
        limit: 50
      }
    });
    
    allResults = allResults.concat(response.data);
    hasMore = response.data.length === 50;
    page++;
  }
  
  return allResults;
};
```

### Caching
```javascript
const perform = async (z, bundle) => {
  const cacheKey = `user_${bundle.authData.user_id}`;
  let userData = z.cache.get(cacheKey);
  
  if (!userData) {
    const response = await z.request({
      url: 'https://api.example.com/user'
    });
    userData = response.data;
    z.cache.set(cacheKey, userData, 300); // Cache for 5 minutes
  }
  
  return userData;
};
```

## Deployment và Monitoring

### Deploy App
```bash
# Build và deploy
zapier build
zapier upload

# Promote version
zapier promote 1.0.0

# Monitor logs
zapier logs
```

### Environment Variables
```bash
# Set environment variables
zapier env:set CLIENT_ID=your_client_id
zapier env:set CLIENT_SECRET=your_client_secret

# List environment variables
zapier env
```

Zapier cung cấp một nền tảng mạnh mẽ cho automation với khả năng tích hợp rộng rãi và dễ sử dụng cho cả người dùng thông thường và developers.