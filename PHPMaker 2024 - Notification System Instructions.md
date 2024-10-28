#  Notification System Usage Guide

## Setup Instructions

### 1. PHPMaker Configuration

1. In PHPMaker, go to **Tools > Scripts and Stylesheet**
2. Add the MQTT notification script to **Script (Global)**:
```html
../_shared/mqtt-notification.js
```

3. Go to **Code (Server Events, Client Script...)**
4. In the **Page_foot** Tab, add:
```php
include(dirname(__DIR__, 2) . "/_shared/foot.php");
include(dirname(__DIR__, 2) . "/_shared/notification.php");
```

### 2. Sending Notifications

You can send notifications using either client-side JavaScript or server-side PHP.

#### Using JavaScript Fetch
```javascript
// Get JWT token from PHPMaker
const token = ew.API_JWT_TOKEN;

// System-wide notification
async function sendSystemNotification() {
    try {
        const response = await fetch('/UAC/api/notifications/send', {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${token}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                type: 'system',
                subject: 'System Maintenance',
                body: 'Scheduled maintenance tonight at 10 PM',
                link: '/maintenance',
                from_system: 'ADS'
            })
        });
        const result = await response.json();
        console.log(result);
    } catch (error) {
        console.error('Error:', error);
    }
}

// Personal notification
async function sendPersonalNotification(userId) {
    try {
        const response = await fetch('/UAC/api/notifications/send', {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${token}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                type: 'personal',
                target: userId,
                subject: 'Document Approved',
                body: 'Your document has been approved',
                link: '/documents/123',
                from_system: 'DMS'
            })
        });
        const result = await response.json();
        console.log(result);
    } catch (error) {
        console.error('Error:', error);
    }
}
```

#### Using PHP GuzzleHTTP
```php
use GuzzleHttp\Client;

function sendNotification() {
    $client = new Client();
    $token = GetJwtToken(); // Get JWT token from PHPMaker

    try {
        $response = $client->post('http://your-site/UAC/api/notifications/send', [
            'headers' => [
                'Authorization' => 'Bearer ' . $token,
                'Content-Type' => 'application/json'
            ],
            'json' => [
                'type' => 'userLevel',
                'target' => '1000', // user_level_id
                'subject' => 'New Feature',
                'body' => 'New features available',
                'link' => '/features',
                'from_system' => 'LMS'
            ]
        ]);
        
        $result = json_decode($response->getBody(), true);
        return $result;
    } catch (\Exception $e) {
        return ['error' => $e->getMessage()];
    }
}
```

## Notification Types and Examples

### 1. System-wide Notification
```json
{
    "type": "system",
    "subject": "System Maintenance",
    "body": "The system will be down for maintenance on Sunday at 2 AM.",
    "link": "/maintenance-schedule",
    "from_system": "ADS"
}
```

### 2. Personal Notification
```json
{
    "type": "personal",
    "target": "123",
    "subject": "Document Approved",
    "body": "Your document #ABC123 has been approved.",
    "link": "/documents/ABC123",
    "from_system": "DMS"
}
```

### 3. User Level Notification
```json
{
    "type": "userLevel",
    "target": "1000",
    "subject": "New Feature Available",
    "body": "Archive search feature is now available for administrators.",
    "link": "/features/archive-search",
    "from_system": "AMS"
}
```

## System Codes
- ADS: Administrative System
- UAC: User Access Control Management
- AMS: Archives Management System
- ASM: Asset Management System
- LMS: Library Management System
- PMS: Project Management System
- MMS: Museum Management System
- DMS: Document Management System
- PRESERBA: Preservation and Renewal System
- SMS: Survey Management System
- HRS: HR System

## Testing with cURL

### System Notification
```bash
curl -X POST http://your-site/UAC/api/notifications/send \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "system",
    "subject": "System Update",
    "body": "Important system update scheduled.",
    "link": "/updates",
    "from_system": "ADS"
  }'
```

### Personal Notification
```bash
curl -X POST http://your-site/UAC/api/notifications/send \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "personal",
    "target": "123",
    "subject": "Profile Updated",
    "body": "Your profile has been updated successfully.",
    "link": "/profile",
    "from_system": "HRS"
  }'
```

### User Level Notification
```bash
curl -X POST http://your-site/UAC/api/notifications/send \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "userLevel",
    "target": "1000",
    "subject": "New Access Level",
    "body": "Your access level has been updated.",
    "link": "/access-levels",
    "from_system": "UAC"
  }'
```

## Troubleshooting

1. **Authentication Issues**
   - Verify JWT token is valid
   - Check token expiration
   - Ensure proper headers are set

2. **Missing Fields**
   - Required fields: type, subject, body
   - target required for personal and userLevel notifications

3. **Invalid System Codes**
   - Use only approved system codes from the list above

4. **Common Error Codes**
   - 400: Missing or invalid fields
   - 401: Authentication failed
   - 500: Server error

For additional support, contact the system administrator or refer to the technical documentation.
