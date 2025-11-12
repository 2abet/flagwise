# User Management

FlagWise supports role-based access control with two user types: **Admin** and **Read-only**. This guide covers user account management, permissions, and security best practices.

## User Roles

### Admin Users
- Full system access and configuration
- Create, edit, and delete detection rules
- Manage user accounts and permissions
- Access sensitive data (full prompts and responses)
- Configure system settings and alerts
- Export data and generate reports

### Read-only Users
- View dashboards and analytics
- Browse LLM requests (prompt previews only)
- View flagged prompts and alerts
- Access live feed and session data
- Cannot modify system configuration

## Managing Users

### Default Account
- **Username**: `admin`
- **Password**: `admin123`
- **⚠️ Change this password immediately after first login**

### Creating New Users

1. Navigate to **Settings → User Management**
2. Click **Add User**
3. Fill in user details:
   - Username (3-50 characters, alphanumeric + hyphens/underscores)
   - Password (minimum 8 characters recommended for production)
   - Role (Admin or Read-only)
   - First/Last name (optional)
4. Click **Create User**

### Editing Users

1. Go to **Settings → User Management**
2. Click the edit icon next to a user
3. Modify role, active status, or personal information
4. Save changes

### Password Management

#### Self-Service Password Change
1. Click your profile icon in the top-right
2. Select **Change Password**
3. Enter current and new password
4. Confirm changes

#### Admin Password Reset
Admins can reset any user's password:
1. Go to **Settings → User Management**
2. Click the reset icon next to a user
3. Enter new password
4. User will be notified to change it on next login

### Deactivating Users

Instead of deleting users (which removes audit trails):
1. Edit the user account
2. Set **Active Status** to **Disabled**
3. User can no longer log in but historical data remains

## Security Best Practices

### Password Requirements
- **Production**: Minimum 8 characters (strongly recommend 12+)
- **Development**: Minimum 6 characters (for testing only)
- Use unique passwords for each account
- Implement password complexity rules in production
- Regular password rotation for admin accounts

### Access Control
- Follow principle of least privilege
- Regularly review user permissions
- Remove access for departed team members
- Monitor user activity in audit logs

### Session Management
- Sessions expire after inactivity
- Users are logged out when role changes
- Multiple concurrent sessions are allowed
- JWT tokens are used for authentication

## API Access

Users can access the FlagWise API using their credentials:

```bash
# Login to get access token
curl -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "your_username", "password": "your_password"}'

# Note: Replace YOUR_ACCESS_TOKEN with the actual token from login response
# Use token in subsequent requests
curl -X GET http://localhost:8000/api/requests \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

## Troubleshooting

### Cannot Login
- Verify username and password
- Check if account is active
- Ensure FlagWise services are running
- Check browser console for errors

### Permission Denied
- Verify user role has required permissions
- Admin actions require Admin role
- Some data requires specific access levels

### Forgot Password
- Contact an Admin user for password reset
- **Known Limitation**: No self-service password recovery currently available
- **Enhancement Needed**: Email-based or security-question recovery system
- This is a significant usability gap that should be prioritized for future releases

## Audit and Compliance

### User Activity Tracking
- All user actions are logged
- Login/logout events recorded
- Data access and modifications tracked
- Export capabilities for compliance reporting

### Data Access Controls
- Read-only users see truncated prompts only
- Full prompt/response data requires Admin role
- IP-based access restrictions can be configured
- Session timeouts enforce security policies