# Database Schema

FlagWise uses PostgreSQL 15+ for data persistence and analytics. This document describes the database structure, relationships, and optimization strategies.

## Core Tables

### llm_requests
Primary table storing all intercepted LLM API calls.

```sql
CREATE TABLE llm_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    src_ip TEXT NOT NULL,
    provider TEXT NOT NULL,
    model TEXT NOT NULL,
    endpoint TEXT,
    method TEXT DEFAULT 'POST',
    prompt TEXT NOT NULL,
    response TEXT,
    headers JSONB,
    tokens_prompt INTEGER,
    tokens_response INTEGER,
    tokens_total INTEGER,
    duration_ms INTEGER,
    status_code INTEGER,
    risk_score INTEGER DEFAULT 0,
    is_flagged BOOLEAN DEFAULT FALSE,
    flag_reason TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

**Key Fields:**
- `id`: UUID primary key for security
- `timestamp`: When the request occurred
- `src_ip`: Internal IP address of requestor
- `provider`: openai, anthropic, etc.
- `model`: gpt-4, claude-3, etc.
- `prompt`: User prompt (encrypted in production)
- `response`: LLM response (encrypted in production)
- `risk_score`: 0-100 calculated risk score
- `is_flagged`: Whether request triggered detection rules
- `flag_reason`: Comma-separated list of triggered rule names

### detection_rules
Configurable rules for flagging risky requests.

```sql
CREATE TABLE detection_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    category TEXT NOT NULL CHECK (category IN ('data_privacy', 'security', 'compliance')),
    rule_type TEXT NOT NULL CHECK (rule_type IN ('keyword', 'regex', 'model_restriction', 'custom_scoring')),
    pattern TEXT NOT NULL,
    severity TEXT NOT NULL CHECK (severity IN ('critical', 'high', 'medium', 'low')),
    points INTEGER NOT NULL CHECK (points >= 0 AND points <= 100),
    priority INTEGER DEFAULT 0,
    stop_on_match BOOLEAN DEFAULT FALSE,
    combination_logic TEXT DEFAULT 'AND' CHECK (combination_logic IN ('AND', 'OR')),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### alerts
Tracks notifications sent for flagged requests.

```sql
CREATE TABLE alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title TEXT NOT NULL,
    description TEXT,
    severity TEXT NOT NULL CHECK (severity IN ('critical', 'high', 'medium', 'low')),
    alert_type TEXT NOT NULL,
    status TEXT DEFAULT 'new' CHECK (status IN ('new', 'acknowledged', 'resolved')),
    source_type TEXT NOT NULL CHECK (source_type IN ('detection_rule', 'threshold', 'system')),
    source_id UUID,
    related_request_id UUID REFERENCES llm_requests(id),
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    acknowledged_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    acknowledged_by TEXT,
    resolved_by TEXT
);
```

### user_sessions
Groups requests by IP address and time windows for session analysis.

```sql
CREATE TABLE user_sessions (
    id TEXT PRIMARY KEY, -- Generated from src_ip + start_time
    src_ip TEXT NOT NULL,
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,
    duration_minutes INTEGER NOT NULL,
    request_count INTEGER NOT NULL,
    avg_risk_score FLOAT NOT NULL,
    flagged_count INTEGER DEFAULT 0,
    geographic_info TEXT,
    user_agent TEXT,
    top_providers TEXT[],
    top_models TEXT[],
    risk_level TEXT CHECK (risk_level IN ('critical', 'high', 'medium', 'low')),
    unusual_patterns TEXT[],
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Security Tables

### users
User accounts and authentication.

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username TEXT NOT NULL UNIQUE,
    hashed_password TEXT NOT NULL,
    first_name TEXT DEFAULT '',
    last_name TEXT DEFAULT '',
    role TEXT NOT NULL CHECK (role IN ('admin', 'read_only')),
    is_active BOOLEAN DEFAULT TRUE,
    last_login TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### alert_rules
Configuration for automated alert generation.

```sql
CREATE TABLE alert_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    rule_type TEXT NOT NULL CHECK (rule_type IN ('threshold', 'detection_rule')),
    is_active BOOLEAN DEFAULT TRUE,
    severity TEXT NOT NULL CHECK (severity IN ('critical', 'high', 'medium', 'low')),
    threshold_config JSONB,
    detection_rule_ids UUID[],
    notifications JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### system_settings
Application configuration and feature flags.

```sql
CREATE TABLE system_settings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key TEXT NOT NULL UNIQUE,
    value TEXT NOT NULL,
    description TEXT,
    category TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Indexes and Performance

### Primary Indexes
```sql
-- Time-based queries (dashboard views)
CREATE INDEX idx_llm_requests_timestamp ON llm_requests(timestamp DESC);
CREATE INDEX idx_llm_requests_created_at ON llm_requests(created_at DESC);

-- IP-based filtering (user tracking)
CREATE INDEX idx_llm_requests_src_ip ON llm_requests(src_ip);
CREATE INDEX idx_user_sessions_src_ip ON user_sessions(src_ip);

-- Provider/model filtering (analytics)
CREATE INDEX idx_llm_requests_provider ON llm_requests(provider);
CREATE INDEX idx_llm_requests_model ON llm_requests(model);
CREATE INDEX idx_llm_requests_provider_model ON llm_requests(provider, model);

-- Flagged requests (security review)
CREATE INDEX idx_llm_requests_flagged ON llm_requests(is_flagged) WHERE is_flagged = TRUE;
CREATE INDEX idx_llm_requests_risk_score ON llm_requests(risk_score DESC);

-- Alert management
CREATE INDEX idx_alerts_status ON alerts(status);
CREATE INDEX idx_alerts_severity ON alerts(severity);
CREATE INDEX idx_alerts_created_at ON alerts(created_at DESC);
```

### Composite Indexes
```sql
-- Dashboard queries
CREATE INDEX idx_llm_requests_timestamp_flagged ON llm_requests(timestamp DESC, is_flagged);
CREATE INDEX idx_llm_requests_timestamp_provider ON llm_requests(timestamp DESC, provider);

-- Session analysis
CREATE INDEX idx_user_sessions_time_range ON user_sessions(start_time, end_time);
CREATE INDEX idx_user_sessions_risk_level ON user_sessions(risk_level);
```

### Partial Indexes
```sql
-- High-risk requests only
CREATE INDEX idx_llm_requests_high_risk ON llm_requests(timestamp DESC) 
WHERE risk_score >= 70;

-- Active rules only
CREATE INDEX idx_detection_rules_active ON detection_rules(priority DESC) 
WHERE is_active = TRUE;
```

## Data Relationships

### Entity Relationships
```
llm_requests (1) ←→ (0..n) alerts
detection_rules (1) ←→ (0..n) alert_rules
users (1) ←→ (0..n) alerts (acknowledged_by, resolved_by)
user_sessions (1) ←→ (0..n) llm_requests (via src_ip + time range)
```

### Foreign Key Constraints
```sql
ALTER TABLE alerts ADD CONSTRAINT fk_alerts_request 
FOREIGN KEY (related_request_id) REFERENCES llm_requests(id);

ALTER TABLE alert_rules ADD CONSTRAINT fk_alert_rules_detection_rules
FOREIGN KEY (detection_rule_ids) REFERENCES detection_rules(id);
```

## Data Retention and Cleanup

### Automatic Cleanup Function
```sql
CREATE OR REPLACE FUNCTION cleanup_old_data()
RETURNS void AS $$
BEGIN
    -- Remove requests older than 6 months
    DELETE FROM llm_requests 
    WHERE created_at < NOW() - INTERVAL '6 months';
    
    -- Remove resolved alerts older than 3 months
    DELETE FROM alerts 
    WHERE status = 'resolved' 
    AND resolved_at < NOW() - INTERVAL '3 months';
    
    -- Remove old sessions
    DELETE FROM user_sessions 
    WHERE end_time < NOW() - INTERVAL '6 months';
END;
$$ LANGUAGE plpgsql;
```

### Scheduled Cleanup
```sql
-- Run cleanup monthly
SELECT cron.schedule('cleanup-old-data', '0 2 1 * *', 'SELECT cleanup_old_data();');
```

## Security Features

### Row-Level Security (Future)
```sql
-- Enable RLS for sensitive tables
ALTER TABLE llm_requests ENABLE ROW LEVEL SECURITY;

-- Admin users see all data
CREATE POLICY admin_all_access ON llm_requests
FOR ALL TO admin_role USING (true);

-- Read-only users see limited data
CREATE POLICY readonly_limited_access ON llm_requests
FOR SELECT TO readonly_role 
USING (created_at > NOW() - INTERVAL '30 days');
```

### Column-Level Encryption (Future)
```sql
-- Encrypt sensitive columns
ALTER TABLE llm_requests 
ADD COLUMN prompt_encrypted BYTEA,
ADD COLUMN response_encrypted BYTEA;
```

## Backup and Recovery

### Backup Strategy
```bash
# Daily full backup
pg_dump -h localhost -U flagwise_user -d flagwise > backup_$(date +%Y%m%d).sql

# Continuous WAL archiving
archive_mode = on
archive_command = 'cp %p /backup/wal/%f'
```

### Point-in-Time Recovery
```bash
# Restore to specific timestamp
pg_restore -h localhost -U flagwise_user -d flagwise_restored backup.sql
```

## Monitoring and Maintenance

### Database Statistics
```sql
-- Table sizes and row counts
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    pg_stat_get_tuples_returned(c.oid) as rows
FROM pg_tables t
JOIN pg_class c ON c.relname = t.tablename
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Performance Monitoring
```sql
-- Slow queries
SELECT query, mean_time, calls, total_time
FROM pg_stat_statements
WHERE mean_time > 1000
ORDER BY mean_time DESC;

-- Index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

### Maintenance Tasks
```sql
-- Update table statistics
ANALYZE llm_requests;

-- Rebuild indexes if needed
REINDEX INDEX idx_llm_requests_timestamp;

-- Vacuum to reclaim space
VACUUM ANALYZE llm_requests;
```

## Migration Scripts

### Schema Updates
```sql
-- Add new column with default value
ALTER TABLE llm_requests 
ADD COLUMN new_field TEXT DEFAULT 'default_value';

-- Create new index concurrently
CREATE INDEX CONCURRENTLY idx_new_field ON llm_requests(new_field);
```

### Data Migration
```sql
-- Migrate existing data
UPDATE llm_requests 
SET new_field = 'migrated_value' 
WHERE new_field = 'default_value';
```

## Troubleshooting

### Common Issues
- **Slow queries**: Check index usage and query plans
- **Lock contention**: Monitor pg_locks table
- **Storage growth**: Implement data retention policies
- **Connection limits**: Tune max_connections setting

### Diagnostic Queries
```sql
-- Current connections
SELECT * FROM pg_stat_activity WHERE state = 'active';

-- Lock information
SELECT * FROM pg_locks WHERE NOT granted;

-- Database size
SELECT pg_size_pretty(pg_database_size('flagwise'));
```