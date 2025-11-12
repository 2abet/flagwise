# Alert Setup

FlagWise provides real-time alerting capabilities to notify security teams when risky LLM activity is detected. Configure alerts to integrate with your existing incident response workflows.

## Alert Types

### Detection Rule Alerts
Triggered when specific detection rules are matched:
- High-risk prompts containing sensitive data
- Unauthorized model usage
- Suspicious activity patterns
- Policy violations

### Threshold Alerts
Triggered when metrics exceed defined limits:
- Request volume spikes
- High average risk scores
- Unusual geographic activity
- Token consumption limits

### System Alerts
Triggered by system events:
- Service health issues
- Database connectivity problems
- Processing errors
- Configuration changes

## Notification Channels

### Slack Integration
Send alerts to Slack channels for team collaboration.

#### Setup
1. Create a Slack webhook URL in your workspace
2. Navigate to **Settings → Alerts**
3. Click **Add Notification Channel**
4. Select **Slack** and enter webhook URL
5. Test the connection

#### Configuration
```json
{
  "channel_type": "slack",
  "webhook_url": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
  "channel": "#security-alerts",
  "username": "FlagWise",
  "icon_emoji": ":warning:"
}
```

**Note**: For user mentions, use Slack user IDs in the format `<@USER_ID>` (e.g., `<@U1234567890>`) rather than @username.

### Email Notifications
*Coming Soon* - Email integration for alert delivery.

### Webhook Integration
Send alerts to custom endpoints for integration with SIEM systems.

```json
{
  "channel_type": "webhook",
  "url": "https://your-siem.company.com/api/alerts",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer YOUR_API_TOKEN",
    "Content-Type": "application/json"
  }
}
```

## Alert Rules

### Creating Alert Rules

1. Navigate to **Settings → Alerts**
2. Click **Add Alert Rule**
3. Configure rule settings:
   - **Name**: Descriptive identifier
   - **Type**: Detection rule or threshold-based
   - **Severity**: Critical, high, medium, or low
   - **Conditions**: Trigger criteria
   - **Notifications**: Which channels to notify

### Detection Rule Alerts

Link alerts to specific detection rules:

```json
{
  "name": "Critical Data Exposure",
  "rule_type": "detection_rule",
  "severity": "critical",
  "detection_rule_ids": ["rule-uuid-1", "rule-uuid-2"],
  "notifications": {
    "slack": {
      "enabled": true,
      "channel": "#security-critical"
    }
  }
}
```

### Threshold Alerts

Monitor system metrics and activity levels:

```json
{
  "name": "High Risk Activity",
  "rule_type": "threshold",
  "severity": "high",
  "threshold_config": {
    "metric": "avg_risk_score",
    "operator": "greater_than",
    "value": 70,
    "time_window": "5m",
    "min_requests": 10
  }
}
```

## Alert Management

### Alert Dashboard
View and manage active alerts:
- **New**: Recently triggered alerts requiring attention
- **Acknowledged**: Alerts being investigated
- **Resolved**: Completed investigations

### Alert Actions
- **Acknowledge**: Mark alert as being investigated
- **Resolve**: Close alert after addressing the issue
- **Archive**: Remove from active view (keeps history)
- **Bulk Operations**: Handle multiple alerts simultaneously

### Alert Details
Each alert includes:
- Trigger timestamp and conditions
- Related LLM requests and sessions
- Risk assessment and context
- Investigation notes and actions taken
- Resolution status and timeline

### Alert Lifecycle and Retention
- **New alerts**: Retained indefinitely until acknowledged/resolved
- **Resolved alerts**: Automatically cleaned up after 3 months
- **Archive function**: Removes from active view but preserves history
- **Manual cleanup**: Bulk delete resolved alerts older than specified date

## Configuration Examples

### High-Risk Prompt Alert
```json
{
  "name": "Sensitive Data Detected",
  "rule_type": "detection_rule",
  "severity": "critical",
  "detection_rule_ids": ["credit-card-rule", "ssn-rule"],
  "notifications": {
    "slack": {
      "enabled": true,
      "channel": "#security-alerts",
      "mention_users": ["<@U1234567890>", "<@U0987654321>"]
    }
  }
}
```

### Volume Spike Alert
```json
{
  "name": "Request Volume Spike",
  "rule_type": "threshold",
  "severity": "medium",
  "threshold_config": {
    "metric": "request_count",
    "operator": "greater_than",
    "value": 1000,
    "time_window": "1h",
    "comparison": "previous_hour"
  }
}
```

### Model Restriction Alert
```json
{
  "name": "Unauthorized Model Usage",
  "rule_type": "detection_rule",
  "severity": "high",
  "detection_rule_ids": ["model-restriction-rule"],
  "notifications": {
    "webhook": {
      "enabled": true,
      "url": "https://siem.company.com/api/alerts"
    }
  }
}
```

## Alert Tuning

### Reducing False Positives
- Adjust detection rule sensitivity
- Implement alert suppression rules
- Use time-based conditions
- Add context-aware filtering

### Alert Fatigue Prevention
- **Priority thresholds**: Critical (>80 risk), High (60-79), Medium (40-59)
- **Grouping rules**: Combine alerts from same IP within 5-minute window
- **Suppression**: Silence duplicate alerts for 1 hour after first occurrence
- **Escalation**: Auto-escalate unacknowledged critical alerts after 15 minutes

### Performance Optimization
- Limit alert frequency per rule
- Use efficient threshold calculations
- Batch notifications when possible
- Monitor alert processing latency

## Integration Patterns

### SIEM Integration
Forward alerts to Security Information and Event Management systems:

```bash
# Example webhook payload
{
  "timestamp": "2024-01-15T10:30:00Z",
  "severity": "critical",
  "source": "flagwise",
  "event_type": "llm_security_alert",
  "details": {
    "rule_name": "Credit Card Detection",
    "risk_score": 75,
    "src_ip": "192.168.1.100",
    "model": "gpt-4",
    "request_id": "req-uuid-123"
  }
}
```

### Incident Response Integration
Connect with ticketing systems:
- Automatically create tickets for critical alerts
- Include relevant context and investigation links
- Update ticket status based on alert resolution

### Monitoring Integration
Send metrics to monitoring platforms:
- Alert frequency and response times
- False positive rates
- System health indicators
- User activity patterns

## Troubleshooting

### Alerts Not Triggering
- Verify alert rule is active
- Check detection rule configuration
- Confirm notification channels are working
- Review alert processing logs

### Missing Notifications
- Test webhook/Slack connectivity
- Verify authentication credentials
- Check rate limiting and quotas
- Review notification channel settings

### Performance Issues
- Monitor alert processing latency
- Optimize threshold calculations
- Reduce notification frequency
- Check system resource usage

## API Reference

### List Alerts
```bash
GET /api/alerts?status=new&severity=critical
```

### Create Alert Rule
```bash
POST /api/alert-rules
Content-Type: application/json

{
  "name": "New Alert Rule",
  "rule_type": "threshold",
  "severity": "high",
  "threshold_config": {
    "metric": "risk_score",
    "operator": "greater_than",
    "value": 80
  }
}
```

### Acknowledge Alert
```bash
PUT /api/alerts/{alert_id}
Content-Type: application/json

{
  "status": "acknowledged",
  "acknowledged_by": "security_analyst"
}
```

### Test Notification Channel
**Note**: Requires Admin authentication to prevent unauthorized testing of arbitrary endpoints.

```bash
POST /api/notifications/test
Content-Type: application/json
Authorization: Bearer ADMIN_TOKEN

{
  "channel_type": "slack",
  "webhook_url": "https://hooks.slack.com/...",
  "test_message": "FlagWise alert test"
}
```