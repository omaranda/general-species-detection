# Quick Start Guide

Get the Camera Trap Species Detection Platform running in under 30 minutes!

## Prerequisites

Install these before starting:
- [AWS CLI](https://aws.amazon.com/cli/) configured with credentials
- [Terraform](https://www.terraform.io/downloads) >= 1.6
- [Node.js](https://nodejs.org/) >= 20
- [Python](https://www.python.org/) 3.11+
- [PostgreSQL client](https://www.postgresql.org/download/)

## 5-Step Setup

### 1. Clone and Configure (5 minutes)

```bash
# Clone repository
cd /Users/omiranda/Documents/GitHub/speciesDetection

# Configure Terraform
cd infrastructure/terraform
cp terraform.tfvars.example terraform.tfvars

# Edit with your settings:
nano terraform.tfvars
# - Set s3_bucket_name (your existing sensor data bucket)
# - Set alert_email
# - Adjust region if needed
```

### 2. Deploy Infrastructure (10 minutes)

```bash
# Initialize and deploy
terraform init
terraform apply

# Save these outputs:
# - rds_endpoint
# - lambda_function_name
# - db_secret_arn
```

### 3. Deploy Database (2 minutes)

```bash
# Get credentials from Secrets Manager
export DB_SECRET=$(terraform output -raw db_secret_arn)
export DB_INFO=$(aws secretsmanager get-secret-value --secret-id $DB_SECRET --query SecretString --output text)

export DB_HOST=$(echo $DB_INFO | jq -r .host)
export DB_USER=$(echo $DB_INFO | jq -r .username)
export DB_PASS=$(echo $DB_INFO | jq -r .password)
export DB_NAME=$(echo $DB_INFO | jq -r .dbname)

# Run migrations
cd ../../database/migrations
PGPASSWORD=$DB_PASS psql -h $DB_HOST -U $DB_USER -d $DB_NAME -f 001_initial_schema.sql
```

### 4. Deploy Lambda (5 minutes)

```bash
cd ../../backend/lambda/detection_pipeline

# Package Lambda
pip3 install -r requirements.txt -t package/
cp *.py package/
cd package && zip -r ../deployment.zip . && cd ..

# Deploy
LAMBDA_NAME=$(cd ../../infrastructure/terraform && terraform output -raw lambda_function_name)
aws lambda update-function-code --function-name $LAMBDA_NAME --zip-file fileb://deployment.zip
```

### 5. Start Dashboard (3 minutes)

```bash
cd ../../../dashboard

# Install dependencies
npm install

# Configure environment
cat > .env.local <<EOF
DATABASE_URL=postgresql://$DB_USER:$DB_PASS@$DB_HOST:5432/$DB_NAME
NEXT_PUBLIC_MAPBOX_TOKEN=
EOF

# Start development server
npm run dev
```

## Access the Platform

Open your browser to: **http://localhost:3000**

You should see:
- Dashboard with statistics
- Species list page
- Map view (add Mapbox token to enable)

## Test the Pipeline

Upload a test image to your S3 bucket:

```bash
# Upload to trigger Lambda
aws s3 cp test-image.jpg s3://your-bucket/test-project/germany/client-a/camera-001/2024-01-01/

# Watch Lambda logs
aws logs tail /aws/lambda/$LAMBDA_NAME --follow

# Check results in dashboard
# Visit http://localhost:3000/species
```

## Next Steps

1. **Download ML Models** - See [docs/DEPLOYMENT_GUIDE.md](docs/DEPLOYMENT_GUIDE.md)
2. **Get Mapbox Token** - Sign up at [mapbox.com](https://mapbox.com) for free
3. **Configure S3 Events** - Automatically process new uploads
4. **Customize Thresholds** - Adjust detection confidence levels
5. **Add Species Data** - Import iNaturalist taxonomy

## Troubleshooting

**Dashboard shows "Loading..." forever**
- Check DATABASE_URL in .env.local
- Verify RDS security group allows your IP
- Check if database schema was deployed

**Lambda not triggering**
- Verify S3 event notification is configured
- Check Lambda permissions for S3
- Review CloudWatch Logs

**"Connection refused" errors**
- Check VPC security groups
- Verify NAT Gateway is working
- Test database connection from Lambda

## Cost Estimate

Development environment (~10K images/month):
- RDS: $15/month
- Lambda: $25/month
- Storage: $10/month
- **Total: ~$50-60/month**

## Support

- Documentation: [docs/](docs/)
- Architecture: [ARCHITECTURE.md](ARCHITECTURE.md)
- Deployment Guide: [docs/DEPLOYMENT_GUIDE.md](docs/DEPLOYMENT_GUIDE.md)
- Issues: GitHub Issues

---

**Ready to deploy to production?** See [docs/DEPLOYMENT_GUIDE.md](docs/DEPLOYMENT_GUIDE.md) for complete instructions including ML model setup and production configuration.
