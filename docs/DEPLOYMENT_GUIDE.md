## Deployment Guide - Camera Trap Species Detection Platform

This guide provides step-by-step instructions for deploying the complete species detection platform on AWS.

## Prerequisites

- AWS Account with administrative permissions
- AWS CLI configured (`aws configure`)
- Terraform >= 1.6
- Node.js >= 20
- Python 3.11+
- PostgreSQL client (for migrations)

## Architecture Overview

The platform consists of:
1. **PostgreSQL RDS** - Species and detection database
2. **Lambda Function** - AI processing pipeline (MegaDetector + SpeciesNet)
3. **Next.js Dashboard** - Web interface
4. **S3 Integration** - Leverages existing sensor data bucket
5. **VPC** - Network isolation for security

## Step 1: Prepare ML Models

Download MegaDetector and SpeciesNet models:

```bash
cd backend/layers/ml-models

# Download MegaDetector v5
wget https://github.com/microsoft/CameraTraps/releases/download/v5.0/md_v5a.0.0.pt \
  -O megadetector_v5.pt

# Download SpeciesNet (iNaturalist) - example URL
# Replace with actual model URL
wget YOUR_SPECIESNET_MODEL_URL -O speciesnet_inat.pt

# Download taxonomy mapping
wget YOUR_TAXONOMY_JSON_URL -O taxonomy.json
```

## Step 2: Deploy Infrastructure with Terraform

```bash
cd infrastructure/terraform

# Copy and configure variables
cp terraform.tfvars.example terraform.tfvars

# Edit terraform.tfvars with your settings:
# - s3_bucket_name: Your existing sensor data bucket
# - alert_email: Your email for CloudWatch alerts
# - aws_region: Your preferred AWS region
# - environment: dev, staging, or production

nano terraform.tfvars

# Initialize Terraform
terraform init

# Review the plan
terraform plan

# Deploy infrastructure
terraform apply

# Note the outputs:
# - rds_endpoint
# - db_secret_arn
# - lambda_function_name
```

**Infrastructure created:**
- VPC with public/private subnets
- PostgreSQL RDS instance
- Lambda function with VPC access
- Security groups
- CloudWatch alarms
- SNS topic for alerts

## Step 3: Deploy Database Schema

```bash
# Get database credentials from Secrets Manager
export DB_SECRET=$(terraform output -raw db_secret_arn)
export DB_CREDENTIALS=$(aws secretsmanager get-secret-value \
  --secret-id $DB_SECRET \
  --query SecretString \
  --output text)

export DB_HOST=$(echo $DB_CREDENTIALS | jq -r .host)
export DB_USER=$(echo $DB_CREDENTIALS | jq -r .username)
export DB_PASSWORD=$(echo $DB_CREDENTIALS | jq -r .password)
export DB_NAME=$(echo $DB_CREDENTIALS | jq -r .dbname)

# Run database migrations
cd ../../database/migrations

psql "postgresql://$DB_USER:$DB_PASSWORD@$DB_HOST:5432/$DB_NAME" \
  -f 001_initial_schema.sql

# Verify schema
psql "postgresql://$DB_USER:$DB_PASSWORD@$DB_HOST:5432/$DB_NAME" \
  -c "\dt"
```

## Step 4: Package and Deploy Lambda Function

```bash
cd ../../backend/lambda/detection_pipeline

# Create deployment package
pip install -r requirements.txt -t package/

# Add Lambda code to package
cp handler.py megadetector.py speciesnet.py database.py package/
cp ../common/utils.py package/

# Create ZIP file
cd package
zip -r ../deployment.zip .
cd ..

# Upload to Lambda
LAMBDA_NAME=$(cd ../../infrastructure/terraform && terraform output -raw lambda_function_name)

aws lambda update-function-code \
  --function-name $LAMBDA_NAME \
  --zip-file fileb://deployment.zip

# Update environment variables
aws lambda update-function-configuration \
  --function-name $LAMBDA_NAME \
  --environment "Variables={
    DB_HOST=$DB_HOST,
    DB_NAME=$DB_NAME,
    DB_USER=$DB_USER,
    DB_PASSWORD_SECRET_ID=$DB_SECRET,
    MEGADETECTOR_THRESHOLD=0.6,
    SPECIESNET_THRESHOLD=0.5
  }"
```

## Step 5: Deploy Lambda Layer with ML Models

```bash
cd ../../layers/ml-models

# Create layer directory structure
mkdir -p python/ml-models

# Copy models
cp megadetector_v5.pt python/ml-models/
cp speciesnet_inat.pt python/ml-models/
cp taxonomy.json python/ml-models/

# Install dependencies
pip install torch torchvision --target python/

# Create layer ZIP
zip -r ml-models-layer.zip python/

# Publish layer
aws lambda publish-layer-version \
  --layer-name species-detection-ml-models \
  --zip-file fileb://ml-models-layer.zip \
  --compatible-runtimes python3.11

# Get layer ARN
LAYER_ARN=$(aws lambda list-layer-versions \
  --layer-name species-detection-ml-models \
  --query 'LayerVersions[0].LayerVersionArn' \
  --output text)

# Update Lambda to use layer
aws lambda update-function-configuration \
  --function-name $LAMBDA_NAME \
  --layers $LAYER_ARN
```

## Step 6: Configure S3 Event Trigger

```bash
# Get bucket name from terraform.tfvars
BUCKET_NAME="your-sensor-data-bucket-name"

# Create Lambda permission for S3
aws lambda add-permission \
  --function-name $LAMBDA_NAME \
  --statement-id AllowS3Invoke \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::$BUCKET_NAME

# Add S3 event notification
cat > s3-notification.json <<EOF
{
  "LambdaFunctionConfigurations": [
    {
      "LambdaFunctionArn": "$(aws lambda get-function --function-name $LAMBDA_NAME --query 'Configuration.FunctionArn' --output text)",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {
              "Name": "suffix",
              "Value": ".jpg"
            }
          ]
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-notification-configuration \
  --bucket $BUCKET_NAME \
  --notification-configuration file://s3-notification.json
```

## Step 7: Deploy Next.js Dashboard

```bash
cd ../../../dashboard

# Install dependencies
npm install

# Create environment file
cat > .env.local <<EOF
# Database connection
DATABASE_URL=postgresql://$DB_USER:$DB_PASSWORD@$DB_HOST:5432/$DB_NAME

# Optional: Mapbox for map functionality
NEXT_PUBLIC_MAPBOX_TOKEN=your_mapbox_token_here

# API URL (if deploying to production)
NEXT_PUBLIC_API_URL=https://your-domain.com
EOF

# Build the application
npm run build

# For development:
npm run dev

# For production (using PM2 or similar):
npm install -g pm2
pm2 start npm --name "species-dashboard" -- start
```

## Step 8: Deploy Dashboard to Production (Optional)

### Option A: Deploy to Vercel

```bash
# Install Vercel CLI
npm install -g vercel

# Deploy
vercel

# Add environment variables in Vercel dashboard
# - DATABASE_URL
# - NEXT_PUBLIC_MAPBOX_TOKEN
```

### Option B: Deploy to AWS (EC2 or ECS)

See `docs/AWS_DASHBOARD_DEPLOYMENT.md` for detailed instructions.

## Step 9: Test the Pipeline

```bash
# Upload a test image to S3
cd ../scripts

# Test upload script (requires sample images)
python test-upload.py \
  --bucket $BUCKET_NAME \
  --type jpeg \
  --count 5

# Monitor Lambda execution
aws logs tail /aws/lambda/$LAMBDA_NAME --follow

# Check database for results
psql "postgresql://$DB_USER:$DB_PASSWORD@$DB_HOST:5432/$DB_NAME" \
  -c "SELECT COUNT(*) FROM images; SELECT COUNT(*) FROM detections;"
```

## Step 10: Verify Deployment

1. **Check RDS:**
   ```bash
   aws rds describe-db-instances --db-instance-identifier species-detection-dev
   ```

2. **Check Lambda:**
   ```bash
   aws lambda get-function --function-name $LAMBDA_NAME
   ```

3. **Check Dashboard:**
   Open `http://localhost:3000` (or your production URL)
   - Verify dashboard loads
   - Check species list page
   - Test map functionality

4. **Check Monitoring:**
   - CloudWatch Logs: `/aws/lambda/$LAMBDA_NAME`
   - CloudWatch Metrics: Lambda invocations, errors, duration
   - RDS Performance Insights (production only)

## Monitoring and Maintenance

### CloudWatch Dashboards

Create custom dashboard:
```bash
aws cloudwatch put-dashboard \
  --dashboard-name species-detection \
  --dashboard-body file://cloudwatch-dashboard.json
```

### Log Queries

Query Lambda logs:
```bash
aws logs insights \
  --log-group-name /aws/lambda/$LAMBDA_NAME \
  --start-time $(date -u -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string "fields @timestamp, @message | filter @message like /ERROR/"
```

### Database Maintenance

Refresh materialized views daily:
```sql
-- Add to cron or Lambda schedule
SELECT refresh_statistics();
```

## Scaling Considerations

### Development Environment
- RDS: db.t3.micro
- Lambda: 3GB memory
- Cost: ~$60/month for 10K images

### Production Environment
- RDS: db.t3.medium or larger
- Lambda: Reserved concurrency
- RDS Proxy for connection pooling
- Read replicas for dashboard queries
- Cost: ~$200-500/month depending on volume

## Troubleshooting

### Lambda Timeouts
- Increase timeout in `terraform.tfvars` (max 15 minutes)
- Check model loading time
- Verify VPC NAT Gateway connectivity

### Database Connection Issues
- Verify security group rules
- Check Lambda VPC configuration
- Test connection from Lambda

### S3 Event Not Triggering
- Verify Lambda permission
- Check S3 event configuration
- Review CloudWatch Logs

### Dashboard Not Loading Data
- Check DATABASE_URL in .env.local
- Verify RDS security group allows connections
- Check API route logs

## Cost Optimization

1. **Use RDS Reserved Instances** for production (40% savings)
2. **Configure Lambda reserved concurrency** only if needed
3. **Enable S3 Intelligent-Tiering** for image storage
4. **Set CloudWatch log retention** to 30 days
5. **Use RDS storage autoscaling** instead of over-provisioning

## Security Checklist

- [x] RDS in private subnet
- [x] Database encryption at rest
- [x] Secrets in AWS Secrets Manager
- [x] Security groups restrict access
- [x] Lambda in VPC
- [x] CloudWatch logs enabled
- [ ] Enable AWS WAF for dashboard (production)
- [ ] Configure authentication (Cognito/Auth0)
- [ ] Enable audit logging

## Backup and Recovery

### Database Backups
```bash
# Manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier species-detection-dev \
  --db-snapshot-identifier manual-snapshot-$(date +%Y%m%d)

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier species-detection-restored \
  --db-snapshot-identifier manual-snapshot-20240101
```

### Export Data
```bash
# Export detections to CSV
psql "postgresql://$DB_USER:$DB_PASSWORD@$DB_HOST:5432/$DB_NAME" \
  -c "\COPY (SELECT * FROM detections) TO 'detections.csv' CSV HEADER"
```

## Next Steps

1. Populate species catalog with iNaturalist data
2. Configure Mapbox token for map functionality
3. Set up automated testing
4. Configure CI/CD pipeline
5. Enable authentication for dashboard
6. Set up monitoring alerts

## Support

For issues or questions:
- GitHub Issues: `your-repo/issues`
- Documentation: `docs/`
- Architecture: `ARCHITECTURE.md`
