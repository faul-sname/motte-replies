# Motte Notifications

A serverless application that enables users to subscribe and receive notifications for replies on [themotte.org](https://themotte.org)

## Architecture Overview

For cost reasons, this system uses a fully serverless architecture on AWS. It should cost about $5 / year to run.

### Core Components

- **DynamoDB**: NoSQL database for storing user data, email addresses, and subscription information
- **S3 Buckets**: Object storage for managing workflow state and tracking processing
- **Lambda Functions**: Serverless compute for processing comments, managing subscriptions, and sending emails
- **SES**: Email service for receiving subscription requests and sending notifications
- **IAM**: Identity and access management for secure resource access
- **CloudWatch**: Monitoring and logging

## Infrastructure Setup

This project uses Terraform to create and manage all AWS resources across dedicated development and production environments.

### Account Structure

- **Management Account**: Billing and organization administration
- **Development Account**: Testing and development resources
- **Production Account**: Live service resources

## Setup Instructions

### Prerequisites

- AWS CLI configured with appropriate access
- Terraform installed (v1.0+)
- Python (for Lambda function development)

### Initial Setup

1. Clone this repository:
   ```bash
   git clone https://github.com/faul-sname/motte-replies.git
   cd motte-replies
   ```

2. Initialize Terraform:
   ```bash
   cd terraform/environments/dev
   terraform init
   ```

3. Deploy the development environment:
   ```bash
   terraform plan -out=tfplan
   terraform apply tfplan
   ```

4. Repeat for production when ready:
   ```bash
   cd ../prod
   terraform init
   terraform plan -out=tfplan
   terraform apply tfplan
   ```

## Database Schema

### DynamoDB Tables

#### `emails`
- `id` (String, Primary Key): UUID for the email record
- `email` (String, GSI): Email address
- `created_at` (String): ISO timestamp of record creation
- `deleted_at` (String, Optional): ISO timestamp of deletion

#### `users`
- `id` (String, Primary Key): UUID for the user record
- `username` (String, GSI): Forum username

#### `subscriptions`
- `email_id` (String, Primary Key): Foreign key to emails table
- `user_id` (String, Sort Key): Foreign key to users table
- `ses_inbound_message_id` (String): SES message ID that created the subscription
- `type` (String): Subscription type (e.g., "reply")
- `created_at` (String): ISO timestamp of subscription creation
- `deleted_at` (String, Optional): ISO timestamp of subscription deletion

## Workflow Details

### Comment Notification Process

1. **Fetch Comments**:
   - Lambda `fetch-recent-comments` runs every 5 minutes
   - Fetches recent forum comments using API key
   - Stores each comment as a JSON object in `comments-pending` S3 bucket

2. **Process Comments**:
   - Lambda `process-comment` triggered by S3 object creation
   - Checks for active subscriptions matching the comment
   - If match found, creates email notification object in `outbound-emails-pending`

3. **Send Emails**:
   - Lambda `process-outbound-email` triggered by S3 object creation
   - Sends email notification via SES
   - Records successful delivery

### Subscription Management

1. **Email Subscription Request**:
   - User sends email with subject "subscribe @username"
   - SES receipt rule places request in `subscription-actions-pending`

2. **Process Subscription**:
   - Lambda `process-subscription-action` triggered by S3 object creation
   - Creates necessary database records
   - Records subscription in DynamoDB

3. **Unsubscribe Process**:
   - User sends email with subject "unsubscribe @username"
   - Lambda `unsubscribe-email-from-user-replies` processes request
   - Marks subscription as deleted in DynamoDB

## S3 Bucket Structure

This project uses a state machine pattern with S3 buckets representing different stages of processing:

- **Pending**: Items awaiting processing
- **Processing**: Items currently being processed
- **Completed**: Successfully processed items
- **Failed**: Items that encountered errors during processing

Categories:
- `comments-*`: Forum comment objects
- `subscription-actions-*`: Subscription requests
- `outbound-emails-*`: Email notifications to be sent

## Lambda Functions

| Function | Description | Trigger | Runtime |
|----------|-------------|---------|---------|
| `fetch-recent-comments` | Retrieves forum comments | CloudWatch (5 min) | Python 3.13 |
| `process-comment` | Processes comments and creates notifications | S3 (`comments-pending`) | Python 3.13 |
| `process-subscription-action` | Manages subscription requests | S3 (`subscription-actions-pending`) | Python 3.13 |
| `process-outbound-email` | Sends email notifications | S3 (`outbound-emails-pending`) | Python 3.13 |

## Security Considerations

- All S3 buckets use server-side encryption (SSE-S3)
- DynamoDB tables are encrypted at rest
- IAM roles follow principle of least privilege
- Lambda functions have specific permissions for required resources only
- SES configured with appropriate DKIM and SPF records

## Monitoring and Maintenance

The system includes:
- CloudWatch alarms for Lambda errors
- Log groups with 30-day retention
- DynamoDB capacity monitoring
- S3 lifecycle policies (30-day retention for completed/failed objects)

## Local Development

1. Run Lambda functions locally using AWS SAM:
   ```bash
   sam local invoke FetchRecentCommentsFunction
   ```
## Deployment Pipeline

A GitHub Actions workflow is included for CI/CD:
1. Run tests
2. Lint code
3. Plan Terraform changes
4. Apply to development (on merge to develop branch)
5. Apply to production (on merge to main branch)

## Contributing

1. Create a feature branch from develop
2. Make changes and test locally
3. Submit a pull request to develop
4. After review, changes will be merged and deployed

## License

MIT

## Support

For questions or issues, please open a GitHub issue.
