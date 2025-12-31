# Camera Trap Species Detection Platform - Architecture

## Overview

An AI-powered platform for automated analysis of camera trap images, leveraging MegaDetector for animal detection and SpeciesNet for species classification. Built on AWS infrastructure with Next.js dashboard for real-time monitoring and geospatial visualization.

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Camera Trap Images                        │
│              (Uploaded to S3 via existing                    │
│               sensor data pipeline)                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              S3 Bucket (Image Storage)                       │
│   - Raw images: /project/country/client/sensor/date/        │
│   - S3 Event Trigger on .jpg/.png uploads                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│    Lambda: Image Detection Pipeline                         │
│                                                              │
│    Step 1: MegaDetector                                      │
│    - Detects animals in images                              │
│    - Returns bounding boxes + confidence scores             │
│    - Filters out empty/blank images                         │
│                                                              │
│    Step 2: SpeciesNet                                        │
│    - Classifies detected animals                            │
│    - Returns species prediction + confidence                │
│    - Supports 5,000+ wildlife species                       │
│                                                              │
│    Step 3: Data Storage                                      │
│    - Store results in PostgreSQL                            │
│    - Update DynamoDB tracking table                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│           PostgreSQL RDS (Detection Database)                │
│                                                              │
│   Tables:                                                    │
│   - detections: Core detection records                      │
│   - species: Species catalog with taxonomy                  │
│   - images: Image metadata + EXIF data                      │
│   - locations: GPS coordinates from camera traps            │
│   - detection_boxes: Bounding box coordinates               │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Next.js 16.1.1 Web Dashboard                    │
│                                                              │
│   Pages:                                                     │
│   - /dashboard: Overview metrics and statistics             │
│   - /species: Species list with filters and search          │
│   - /map: Interactive map with species geolocation          │
│   - /images: Image gallery with detections                  │
│   - /analytics: Trends and patterns analysis                │
│                                                              │
│   Features:                                                  │
│   - Real-time updates via Server Components                 │
│   - Interactive maps (Mapbox/Leaflet)                       │
│   - Species filtering and search                            │
│   - Export capabilities (CSV, JSON)                         │
│   - Image viewer with bounding boxes                        │
└─────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Image Processing Pipeline (Lambda)

**Technology Stack:**
- Python 3.11
- PyTorch for model inference
- MegaDetector v5.0
- SpeciesNet (iNaturalist model)
- PIL/Pillow for image processing

**Processing Flow:**
1. Lambda triggered by S3 event (new image upload)
2. Download image from S3
3. Run MegaDetector to detect animals
4. If animals detected, crop regions and run SpeciesNet
5. Extract EXIF data (GPS, timestamp, camera settings)
6. Store all results in PostgreSQL
7. Update DynamoDB tracking status

**Performance Optimization:**
- Use Lambda layers for ML models (cold start optimization)
- Batch processing for multiple detections per image
- Configurable confidence thresholds
- EFS mount for model caching

### 2. PostgreSQL Database Schema

**Tables:**

```sql
-- Species catalog
CREATE TABLE species (
    id SERIAL PRIMARY KEY,
    scientific_name VARCHAR(255) UNIQUE NOT NULL,
    common_name VARCHAR(255),
    taxonomy_class VARCHAR(100),
    taxonomy_order VARCHAR(100),
    taxonomy_family VARCHAR(100),
    conservation_status VARCHAR(50),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Image metadata
CREATE TABLE images (
    id SERIAL PRIMARY KEY,
    s3_key VARCHAR(500) UNIQUE NOT NULL,
    s3_bucket VARCHAR(255) NOT NULL,
    file_size BIGINT,
    width INTEGER,
    height INTEGER,
    captured_at TIMESTAMP,
    uploaded_at TIMESTAMP DEFAULT NOW(),
    camera_id VARCHAR(100),
    project_name VARCHAR(100),
    country VARCHAR(100),
    client VARCHAR(100),
    exif_data JSONB,
    processing_status VARCHAR(50) DEFAULT 'pending'
);

-- Location data from camera traps
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    camera_id VARCHAR(100) UNIQUE NOT NULL,
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    altitude DECIMAL(10, 2),
    location_name VARCHAR(255),
    habitat_type VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Detection results
CREATE TABLE detections (
    id SERIAL PRIMARY KEY,
    image_id INTEGER REFERENCES images(id) ON DELETE CASCADE,
    species_id INTEGER REFERENCES species(id),
    confidence DECIMAL(5, 4),
    detection_type VARCHAR(50), -- 'animal', 'person', 'vehicle'
    bbox_x DECIMAL(8, 4),
    bbox_y DECIMAL(8, 4),
    bbox_width DECIMAL(8, 4),
    bbox_height DECIMAL(8, 4),
    megadetector_confidence DECIMAL(5, 4),
    speciesnet_confidence DECIMAL(5, 4),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_detections_image ON detections(image_id);
CREATE INDEX idx_detections_species ON detections(species_id);
CREATE INDEX idx_images_camera ON images(camera_id);
CREATE INDEX idx_images_captured ON images(captured_at);
CREATE INDEX idx_locations_coords ON locations(latitude, longitude);
```

### 3. Next.js Dashboard

**Technology Stack:**
- Next.js 16.1.1 with App Router
- React 19
- TypeScript
- Tailwind CSS for styling
- Shadcn/ui for UI components
- Mapbox GL JS or Leaflet for maps
- Recharts or Chart.js for visualizations
- PostgreSQL client (node-postgres)

**Key Features:**

**Species List Page (`/species`):**
- Paginated table with all detected species
- Filters: conservation status, taxonomy, date range
- Search by scientific/common name
- Sortable columns
- Species count and observation frequency
- Link to individual species detail pages

**Map View (`/map`):**
- Interactive map showing all camera locations
- Markers color-coded by species diversity
- Click marker to view species detected at location
- Heatmap overlay for detection density
- Filter by species, date range, confidence threshold
- Cluster markers for nearby cameras

**Dashboard (`/dashboard`):**
- Total images processed
- Total species detected
- Detection confidence distribution
- Recent detections timeline
- Top 10 most detected species
- Processing status overview

### 4. AWS Infrastructure (Terraform)

**New Components:**
- RDS PostgreSQL instance (db.t3.micro for dev, scalable)
- Lambda Layer for ML models (MegaDetector + SpeciesNet)
- EFS for model caching (optional, for performance)
- VPC configuration for RDS + Lambda connectivity
- Security groups and network ACLs
- Secrets Manager for database credentials

**Existing Components (from global-sensor-data-management):**
- S3 buckets for image storage
- DynamoDB for file tracking
- Lambda for file validation
- CloudWatch for monitoring
- SNS for alerts

## Data Flow

1. **Image Upload:**
   - Ecologist uploads images via S3 (existing pipeline)
   - Images stored in hierarchical structure
   - S3 event triggers Lambda function

2. **AI Processing:**
   - Lambda downloads image from S3
   - MegaDetector scans for animals
   - If detected, SpeciesNet identifies species
   - Results stored in PostgreSQL with confidence scores

3. **Dashboard Display:**
   - Next.js fetches data from PostgreSQL
   - Server Components for real-time data
   - Interactive visualizations and maps
   - Filtering and search capabilities

4. **Ecologist Workflow:**
   - View species list sorted by frequency
   - Explore map to see geographic distribution
   - Filter low-confidence detections for review
   - Export data for further analysis
   - Track processing status of uploaded images

## ML Models

### MegaDetector v5.0
- **Purpose:** Detect animals, people, and vehicles in camera trap images
- **Architecture:** YOLOv5
- **Output:** Bounding boxes + confidence scores
- **Classes:** animal, person, vehicle
- **Model Size:** ~200MB
- **Performance:** ~95% accuracy on camera trap images

### SpeciesNet (iNaturalist)
- **Purpose:** Classify species from cropped animal images
- **Architecture:** ResNet-based CNN
- **Output:** Species predictions + confidence scores
- **Classes:** 5,000+ wildlife species
- **Model Size:** ~300MB
- **Performance:** Variable by species, typically 80-90% top-5 accuracy

## Scalability Considerations

- **Lambda Concurrency:** Configure reserved concurrency for consistent performance
- **RDS Scaling:** Start with db.t3.micro, upgrade to db.t3.medium+ for production
- **Connection Pooling:** Use RDS Proxy for Lambda connections
- **Caching:** Redis/ElastiCache for frequently accessed species data
- **Image Storage:** S3 Intelligent-Tiering for cost optimization
- **Batch Processing:** Process multiple images in parallel

## Cost Estimates

**Monthly costs for 10,000 images/month:**

| Service | Cost | Notes |
|---------|------|-------|
| RDS PostgreSQL (db.t3.micro) | $15 | 20GB storage included |
| Lambda Executions | $25 | ML inference ~30s per image |
| Lambda Storage (layers) | $5 | Model storage |
| S3 Storage | $10 | Image storage (incremental) |
| Data Transfer | $5 | Outbound transfers |
| **Total** | **~$60/month** | Scales with image volume |

## Security

- **Database:** RDS encryption at rest, SSL/TLS in transit
- **API:** Next.js API routes with authentication
- **S3:** Private buckets, pre-signed URLs for image access
- **Secrets:** AWS Secrets Manager for credentials
- **Network:** VPC with private subnets for RDS and Lambda

## Future Enhancements

- User authentication and multi-tenancy
- Manual species verification workflow
- Confidence threshold tuning per species
- Advanced analytics (activity patterns, population trends)
- Mobile app for field uploads
- Integration with conservation databases (GBIF, iNaturalist)
- Real-time notifications for rare species
- Collaborative annotation tools
