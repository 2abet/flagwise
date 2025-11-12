# Detection Rules

Detection rules are the core of FlagWise's security monitoring. They analyze LLM traffic in real-time to identify risky patterns, unauthorized usage, and potential security threats.

## Rule Types

### Keyword Rules
Exact string matching for sensitive terms.

```json
{
  "name": "Sensitive Keywords",
  "rule_type": "keyword",
  "pattern": "password|secret|api_key|token",
  "severity": "high",
  "points": 50
}
```

### Regex Rules
Pattern matching using regular expressions.

```json
{
  "name": "Credit Card Detection",
  "rule_type": "regex",
  "pattern": "\\b(?:\\d{4}[-\\s]?){3}\\d{4}\\b",
  "severity": "critical",
  "points": 75
}
```

### Model Restriction Rules
Block specific AI models or providers.

```json
{
  "name": "Unauthorized Models",
  "rule_type": "model_restriction",
  "pattern": "gpt-4|claude-3",
  "severity": "medium",
  "points": 30
}
```

### Custom Scoring Rules
Advanced logic for complex threat detection.

```json
{
  "name": "High Token Usage",
  "rule_type": "custom_scoring",
  "pattern": "tokens_total > 2000",
  "severity": "low",
  "points": 25
}
```

## Rule Configuration

### Basic Settings
- **Name**: Descriptive rule identifier
- **Description**: Detailed explanation of the rule's purpose
- **Category**: `data_privacy`, `security`, or `compliance`
- **Pattern**: The detection pattern (keyword, regex, etc.)
- **Severity**: `critical`, `high`, `medium`, or `low`
- **Points**: Risk score contribution (0-100)

### Advanced Settings
- **Priority**: Execution order (0-1000, higher = earlier)
- **Stop on Match**: Halt processing after this rule triggers
- **Combination Logic**: `AND` or `OR` for multi-pattern rules
- **Active Status**: Enable/disable rule without deletion

## Default Rules

FlagWise includes pre-configured rules for common security patterns:

| Rule Name | Type | Pattern | Points | Description |
|-----------|------|---------|--------|-------------|
| Critical Keywords | keyword | password\|secret\|api_key | 50 | Sensitive authentication data |
| Email Addresses | regex | `\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z\|a-z]{2,}\b` | 30 | Personal identifiable information |
| Credit Cards | regex | `\b(?:\d{4}[-\s]?){3}\d{4}\b` | 60 | Financial data patterns |
| IP Addresses | regex | `\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b` | 20 | Network infrastructure data |
| Phone Numbers | regex | `\b\d{3}-\d{3}-\d{4}\b` | 15 | Personal contact information |
| High Token Usage | custom_scoring | tokens_total > 2000 | 25 | Resource consumption limits |
| Restricted Models | model_restriction | unauthorized_model | 20 | Policy enforcement |

## Managing Rules

### Creating Rules

1. Navigate to **Detection Rules** in the sidebar
2. Click **Add Rule**
3. Configure rule settings:
   - Enter name and description
   - Select category and rule type
   - Define the detection pattern
   - Set severity and point values
4. Test the rule with sample data
5. Save and activate

### Editing Rules

1. Find the rule in the Detection Rules list
2. Click the edit icon
3. Modify settings as needed
4. Test changes before saving
5. Update the rule

### Bulk Operations

Select multiple rules to:
- **Enable/Disable**: Toggle active status
- **Delete**: Remove rules permanently
- **Export**: Download rule configurations
- **Import**: Upload rule templates

## Rule Testing

### Pattern Validation
Test your patterns before deployment:

```bash
# Test regex pattern
curl -X POST http://localhost:8000/api/rules/test \
  -H "Content-Type: application/json" \
  -d '{
    "pattern": "\\b(?:\\d{4}[-\\s]?){3}\\d{4}\\b",
    "test_string": "My card number is 1234-5678-9012-3456"
  }'
```

### Sample Data Testing
Use the built-in test interface:
1. Create or edit a rule
2. Click **Test Rule**
3. Enter sample prompt text
4. View matching results and score calculation

## Risk Scoring

### Point System
- Each rule contributes 0-100 points
- Total risk score is sum of triggered rules
- Scores above 70 typically indicate high risk
- Flagged requests require manual review

### Severity Mapping
- **Critical**: 60-100 points (immediate attention)
- **High**: 40-59 points (priority review)
- **Medium**: 20-39 points (routine monitoring)
- **Low**: 1-19 points (informational)

### Combination Logic
Rules can be combined using AND/OR logic:
- **AND**: All patterns must match
- **OR**: Any pattern triggers the rule
- **Priority**: Higher priority rules execute first
- **Stop on Match**: Prevents further rule processing

## Best Practices

### Rule Design
- Start with broad patterns, refine based on results
- Use descriptive names and detailed descriptions
- Test thoroughly before activating
- Monitor false positive rates

### Performance Optimization
- Place high-priority rules first
- Use "Stop on Match" for definitive violations
- Avoid overly complex regex patterns
- Regular cleanup of unused rules

### Maintenance
- Review rule effectiveness monthly
- Update patterns based on new threats
- Archive outdated rules instead of deleting
- Document rule changes and rationale

## Troubleshooting

### High False Positives
- Review and refine rule patterns
- Adjust point values to reduce sensitivity
- Add exclusion patterns for legitimate use cases
- Consider rule priority and combination logic

### Missing Detections
- Verify rule is active and properly configured
- Check pattern syntax and test with known examples
- Ensure rule priority allows execution
- Review logs for processing errors

### Performance Issues
- Optimize complex regex patterns
- Reduce number of active rules
- Use priority settings effectively
- Monitor system resource usage

## API Reference

### List Rules
```bash
GET /api/detection-rules
```

### Create Rule
```bash
POST /api/detection-rules
Content-Type: application/json

{
  "name": "New Rule",
  "category": "security",
  "rule_type": "keyword",
  "pattern": "sensitive_term",
  "severity": "high",
  "points": 50
}
```

### Update Rule
```bash
PUT /api/detection-rules/{rule_id}
```

### Delete Rule
```bash
DELETE /api/detection-rules/{rule_id}
```