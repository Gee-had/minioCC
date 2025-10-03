# Video Transcoding Pipeline - Project Documentation

## Project Overview

A distributed video transcoding system that automatically processes uploaded videos through MinIO, Kafka, and worker nodes, with Firebase integration for authentication and metadata storage.

## Architecture

```
Desktop UI (Tauri + Native File Providers)
    ↓ (upload request)
Backend API Server (Docker on main server)
    ↓ (presigned URL)
MinIO (Object Storage - Ubuntu 22 server)
    ↓ (S3 event notification)
Kafka (Message Broker - Docker on main server)
    ↓ (via normalizer - Docker on main server)
Transcode Worker Nodes (Separate GPU servers)
    ↓ (processed videos + thumbnails)
MinIO (transcoded outputs)
    ↓ (update metadata)
Firebase Firestore (stored_videos collection)
```

### Infrastructure Layout

**Main Server (Ubuntu 22):**
- MinIO (existing installation, proxied via Nginx)
- Kafka + Normalizer (Docker containers)
- Backend API (Docker container)
- Kafka UI (Docker container - optional)

**Worker Nodes (Separate GPU servers):**
- Transcode Worker (standalone binary)
- FFmpeg with GPU acceleration (NVENC/QSV)
- Direct connection to Kafka and MinIO
- No Docker required (native deployment)

## Technology Stack

- **Object Storage**: MinIO (existing Ubuntu 22 instance)
- **Message Queue**: Apache Kafka 3.7 (KRaft mode) - Docker on main server
- **Event Normalizer**: Python 3.12 + aiokafka - Docker on main server
- **Transcoding Workers**: Rust + FFmpeg (GPU-accelerated) - Standalone on dedicated GPU servers
- **Backend API**: Node.js/Express - Docker on main server
- **Database**: Firebase Firestore
- **Authentication**: Firebase Auth (Email/Password)
- **Desktop UI**: 
  - **Core**: Tauri (Rust + React/TypeScript)
  - **Windows**: C++/WinRT using CfAPI (Cloud Filter API)
  - **macOS**: Swift File Provider extension
  - **Communication**: IPC/gRPC between UI and native adapters

## Project Structure

```
video-transcoding-pipeline/
├── main.md                          # This file
├── infrastructure/
│   ├── main-server/                 # Docker services on Ubuntu 22 server
│   │   ├── docker-compose.yml      # Kafka, Normalizer, API, Kafka UI
│   │   ├── .env                    # Environment configuration
│   │   └── nginx/                  # MinIO reverse proxy config
│   └── worker-nodes/               # Standalone deployment for GPU servers
│       ├── install.sh              # Worker node installation script
│       ├── worker-config.toml      # Worker configuration template
│       └── systemd/
│           └── transcode-worker.service
├── services/
│   ├── normalizer/                 # Dockerized on main server
│   │   ├── Dockerfile
│   │   ├── normalizer.py
│   │   └── requirements.txt
│   ├── transcode-worker/           # Standalone binary for GPU servers
│   │   ├── Cargo.toml
│   │   ├── build.sh               # Compile for target architecture
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── kafka_consumer.rs
│   │   │   ├── transcoder.rs      # FFmpeg GPU integration
│   │   │   ├── thumbnail.rs
│   │   │   ├── minio_client.rs
│   │   │   └── firebase_client.rs
│   │   └── README.md              # Worker deployment guide
│   ├── backend-api/                # Dockerized on main server
│   │   ├── Dockerfile
│   │   ├── server.js              # Presigned URL generation
│   │   ├── firebase-admin-config.js
│   │   └── package.json
│   └── kafka-ui/                   # Optional monitoring (Docker)
│       └── docker-compose.override.yml
├── desktop-ui/                      # Native file system integration
│   ├── tauri-app/                  # Main Tauri application
│   │   ├── src-tauri/             # Rust backend
│   │   │   ├── Cargo.toml
│   │   │   ├── src/
│   │   │   │   ├── main.rs
│   │   │   │   ├── grpc_client.rs # Talk to native adapters
│   │   │   │   └── firebase_client.rs
│   │   │   └── build.rs
│   │   └── src/                   # React frontend
│   │       ├── App.tsx
│   │       ├── components/
│   │       └── api/
│   ├── windows-adapter/            # C++/WinRT Cloud Filter API
│   │   ├── CloudMirror.sln
│   │   ├── CloudMirror/
│   │   │   ├── CloudProvider.h
│   │   │   ├── CloudProvider.cpp
│   │   │   └── FileSync.cpp      # MinIO sync logic
│   │   ├── grpc-server/           # gRPC server for Tauri IPC
│   │   │   └── server.cpp
│   │   └── README.md              # Based on Microsoft Cloud Mirror sample
│   ├── macos-adapter/              # Swift File Provider extension
│   │   ├── FileProvider.xcodeproj
│   │   ├── FileProvider/
│   │   │   ├── FileProviderExtension.swift
│   │   │   ├── FileProviderItem.swift
│   │   │   └── MinIOSync.swift   # MinIO sync logic
│   │   ├── grpc-server/           # gRPC server for Tauri IPC
│   │   │   └── Server.swift
│   │   └── README.md              # Based on Apple File Provider sample
│   └── proto/                      # Shared gRPC definitions
│       ├── file_provider.proto
│       └── sync_service.proto
├── scripts/
│   ├── setup-minio-events.sh       # MinIO event configuration
│   ├── create-kafka-topics.sh      # Kafka topic creation
│   ├── deploy-worker-node.sh       # Worker node deployment script
│   └── build-worker-binary.sh      # Cross-compile for different architectures
├── config/
│   ├── firebase-admin-key.json     # (gitignored)
│   ├── worker-nodes.json           # Worker node registry
│   └── transcode-profiles.json     # FFmpeg encoding profiles (GPU-aware)
└── docs/
    ├── SETUP.md                    # Step-by-step installation
    ├── WORKER_DEPLOYMENT.md        # GPU worker setup guide
    ├── DESKTOP_UI.md               # Native file provider integration
    ├── API.md                      # Backend API documentation
    ├── FIREBASE_SCHEMA.md          # Firestore data models
    └── DEPLOYMENT.md               # Production deployment guide
```

## Core Components

### 1. MinIO Configuration

**Existing Setup:**
- Ubuntu 22 server
- Proxied through Nginx
- Bucket: `creation-public` (public access)
- Additional buckets: `jm-test`, `public-jm-test`, `contentcreation`, `testbucket`

**Event Configuration:**
- Monitor `uploads/raw/` prefix
- Supported formats: `.mp4`, `.mov`, `.mkv`, `.avi`, `.r3d`, `.braw`, `.crm`, `.mxf`, `.arw`
- Events: `s3:ObjectCreated:*`
- Target: Kafka topic `minio-object-created`

### 2. Kafka Infrastructure

**Topics:**
- `minio-object-created` (6 partitions) - Raw MinIO events
- `video-transcode-request` (6 partitions) - Normalized transcode jobs
- `video-transcode-completed` (3 partitions) - Completed transcode notifications
- `video-transcode-deadletter` (1 partition) - Failed jobs

**Configuration:**
- Mode: KRaft (no Zookeeper)
- Network: `events-net` (Docker bridge network)
- Retention: 7 days (configurable)

### 3. Normalizer Service

**Purpose:** 
Converts raw MinIO S3 events into structured transcode job requests.

**Input:** Raw MinIO event from `minio-object-created` topic
```json
{
  "EventName": "s3:ObjectCreated:Put",
  "Key": "uploads/raw/video123.mp4",
  "Records": [...]
}
```

**Output:** Normalized job per quality profile to `video-transcode-request` topic
```json
{
  "job_id": "jm-test:uploads/raw/video123.mp4:etag123:h264_1080p",
  "bucket": "jm-test",
  "key": "uploads/raw/video123.mp4",
  "version_id": null,
  "etag": "etag123",
  "size_bytes": 52428800,
  "content_type": "video/mp4",
  "profile": "h264_1080p",
  "meta": {}
}
```

**Configuration:**
- Filters: Only process videos in `uploads/raw/` with allowed extensions
- Profiles: `h264_1080p`, `h264_720p`, `h264_480p`
- Consumer Group: `normalizer`

### 4. Transcode Worker (Rust) - Standalone GPU Servers

**Deployment Model:**
- Standalone Rust binary (not Docker)
- Deployed on dedicated GPU servers (separate from main infrastructure)
- Direct network access to Kafka broker and MinIO endpoint
- GPU-accelerated transcoding (NVIDIA NVENC or Intel QSV)

**Responsibilities:**
1. Consume jobs from `video-transcode-request` topic
2. Download source video from MinIO
3. Generate transcoded versions using GPU acceleration (3 quality levels)
4. Generate thumbnail (JPEG from first frame)
5. Upload outputs to MinIO
6. Update Firebase Firestore with URLs
7. Publish completion event to `video-transcode-completed` topic

**Concurrency:** 8 concurrent transcode jobs per node

**GPU Acceleration:**
- **NVIDIA GPUs**: Use NVENC encoder (`h264_nvenc`, `hevc_nvenc`)
- **Intel GPUs**: Use QuickSync (`h264_qsv`, `hevc_qsv`)
- **Fallback**: CPU-based encoding if GPU unavailable

**Quality Profiles (GPU-optimized):**
| Profile | Resolution | Video Codec | GPU Preset | Bitrate | Audio |
|---------|-----------|-------------|------------|---------|-------|
| h264_480p | 854x480 | h264_nvenc/qsv | p4 (medium) | 1.5 Mbps | AAC 128k |
| h264_720p | 1280x720 | h264_nvenc/qsv | p4 (medium) | 3 Mbps | AAC 192k |
| h264_1080p | 1920x1080 | h264_nvenc/qsv | p4 (medium) | 6 Mbps | AAC 256k |

**Output Structure:**
```
{bucket}/{video_id}/{filename}_480p.mp4
{bucket}/{video_id}/{filename}_720p.mp4
{bucket}/{video_id}/{filename}_1080p.mp4
{bucket}/{video_id}/{filename}_thumbnail.jpg
```

**Configuration File (worker-config.toml):**
```toml
[kafka]
brokers = ["kafka-server:9092"]
consumer_group = "transcode-workers"
topics = ["video-transcode-request"]

[minio]
endpoint = "https://s3.jumpermedia.co"
access_key = "WORKER_ACCESS_KEY"
secret_key = "WORKER_SECRET_KEY"
region = "us-central"

[firebase]
project_id = "your-project-id"
credentials_path = "/etc/transcode-worker/firebase-key.json"

[worker]
node_id = "worker-01"
max_concurrent = 8
temp_dir = "/mnt/nvme/transcode-temp"
gpu_enabled = true
gpu_type = "nvidia"  # or "intel" or "none"

[ffmpeg]
binary_path = "/usr/bin/ffmpeg"
hwaccel = "cuda"  # or "qsv" or "none"
```

**Dependencies:**
- `tokio` - Async runtime
- `rdkafka` - Kafka client
- `aws-sdk-s3` - MinIO/S3 client
- `serde_json` - JSON parsing
- `reqwest` - HTTP client for Firebase REST API
- FFmpeg (system binary with GPU support)

**Installation on Worker Node:**
```bash
# Install dependencies
sudo apt install -y librdkafka-dev libssl-dev pkg-config

# Install FFmpeg with GPU support (NVIDIA example)
sudo apt install -y ffmpeg nvidia-cuda-toolkit

# Copy pre-compiled binary
sudo cp transcode-worker /usr/local/bin/
sudo chmod +x /usr/local/bin/transcode-worker

# Setup configuration
sudo mkdir -p /etc/transcode-worker
sudo cp worker-config.toml /etc/transcode-worker/
sudo cp firebase-key.json /etc/transcode-worker/

# Install systemd service
sudo cp transcode-worker.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable transcode-worker
sudo systemctl start transcode-worker
```

**Systemd Service (/etc/systemd/system/transcode-worker.service):**
```ini
[Unit]
Description=Video Transcode Worker
After=network.target

[Service]
Type=simple
User=transcode
Group=transcode
ExecStart=/usr/local/bin/transcode-worker --config /etc/transcode-worker/worker-config.toml
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

# Resource limits
LimitNOFILE=65536
TasksMax=infinity

[Install]
WantedBy=multi-user.target
```

### 5. Backend API Server

**Endpoints:**

**POST /api/upload/request**
- Auth: Firebase ID token (Bearer)
- Body: `{ "filename": "video.mp4", "size": 52428800, "parent_id": "folder123" }`
- Response: Presigned MinIO upload URL + `video_id`
- Creates placeholder document in Firebase `stored_videos` collection

**GET /api/videos**
- Auth: Firebase ID token
- Query: `?parent_id=folder123`
- Response: List of videos/folders for authenticated user

**POST /api/folders**
- Auth: Firebase ID token
- Body: `{ "name": "New Folder", "parent_id": "folder123" }`
- Response: Created folder document

### 6. Firebase Firestore Schema

**Collection: `stored_videos`**

**Document Type: Folder**
```typescript
{
  id: string,                          // Auto-generated
  type: "Folder",
  name: string,
  owner_id: string,                    // Firebase Auth UID
  owner_name: string,
  owner_ref: DocumentReference,        // /users/{uid}
  parent_id: string | null,            // null for root
  parent_ref: DocumentReference | null,
  parents: DocumentReference[],        // Ancestor chain
  used_in_projects: DocumentReference[],
  created_time: Timestamp,
  updated_time: Timestamp
}
```

**Document Type: Video**
```typescript
{
  id: string,                          // UUID v4
  type: string,                        // MIME type (video/mp4, video/quicktime, etc.)
  name: string,                        // Original filename
  owner_id: string,
  owner_name: string,
  owner_ref: DocumentReference,
  parent_id: string,
  parent_ref: DocumentReference,
  parents: DocumentReference[],
  
  // Storage
  content_url: string,                 // Original video URL in MinIO
  size: number,                        // Bytes
  
  // Transcoding
  thumbnail_url?: string,              // Generated thumbnail
  transcoded_url?: {
    "high": string,                    // 480p
    "HD": string,                      // 720p
    "Full HD": string                  // 1080p
  },
  transcode_status?: "pending" | "processing" | "completed" | "failed",
  transcode_error?: string,
  
  // Metadata
  created_time: Timestamp,
  updated_time: Timestamp,
  used_in_projects?: DocumentReference[]
}
```

### 7. Desktop UI (Tauri + Native File Providers) - Phase 2

**Architecture:**

```
┌─────────────────────────────────────────────────┐
│         Tauri App (Rust + React/TS)            │
│  ┌──────────────┐        ┌──────────────┐      │
│  │ React UI     │◄──────►│ Rust Backend │      │
│  │ (TypeScript) │  IPC   │ (Tauri Core) │      │
│  └──────────────┘        └───────┬──────┘      │
└─────────────────────────────────┼──────────────┘
                                   │ gRPC/IPC
                    ┌──────────────┴──────────────┐
                    │                             │
         ┌──────────▼──────────┐    ┌────────────▼────────────┐
         │  Windows Adapter    │    │    macOS Adapter        │
         │  (C++/WinRT + CfAPI)│    │  (Swift File Provider)  │
         │  ┌────────────────┐ │    │  ┌────────────────┐     │
         │  │ Cloud Filter   │ │    │  │ File Provider  │     │
         │  │ API Integration│ │    │  │ Extension      │     │
         │  └────────────────┘ │    │  └────────────────┘     │
         │  ┌────────────────┐ │    │  ┌────────────────┐     │
         │  │ MinIO Sync     │ │    │  │ MinIO Sync     │     │
         │  │ Engine         │ │    │  │ Engine         │     │
         │  └────────────────┘ │    │  └────────────────┘     │
         └─────────────────────┘    └─────────────────────────┘
                    │                             │
                    └──────────────┬──────────────┘
                                   │
                            ┌──────▼──────┐
                            │   MinIO     │
                            │  + Firebase │
                            └─────────────┘
```

**Components:**

#### Tauri Core Application
- **Purpose**: Cross-platform UI shell and orchestration layer
- **Tech Stack**: Rust (backend) + React + TypeScript (frontend)
- **Responsibilities**:
  - User authentication (Firebase)
  - File/folder browsing and management UI
  - Upload progress tracking
  - Real-time sync status display
  - Communication with native adapters via gRPC/IPC

#### Windows Adapter (C++/WinRT)
- **Based on**: Microsoft Cloud Mirror sample ([Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/cfapi/build-a-cloud-file-sync-engine))
- **API**: Cloud Filter API (CfAPI)
- **Features**:
  - Virtual file system integration (files appear in Explorer without downloading)
  - On-demand hydration (download on first access)
  - Placeholder files with metadata
  - Upload sync to MinIO
  - Real-time file status overlay icons
- **gRPC Server**: Exposes sync control to Tauri app
- **Implementation**:
  - `CloudProvider.h/cpp`: Main sync engine
  - `FileSync.cpp`: MinIO upload/download logic
  - `GrpcServer.cpp`: IPC communication with Tauri

#### macOS Adapter (Swift)
- **Based on**: Apple File Provider extension ([Apple Developer](https://developer.apple.com/documentation/fileprovider))
- **Framework**: File Provider API
- **Features**:
  - Native Finder integration
  - On-demand content fetching
  - Upload monitoring and sync
  - Metadata caching
  - Quick Look preview generation
- **gRPC Server**: Swift-based server for Tauri communication
- **Implementation**:
  - `FileProviderExtension.swift`: Main extension entry point
  - `FileProviderItem.swift`: File/folder model
  - `MinIOSync.swift`: S3 sync logic
  - `Server.swift`: gRPC server for IPC

#### gRPC Protocol Definitions
**file_provider.proto:**
```protobuf
syntax = "proto3";

service FileProviderService {
  rpc GetSyncStatus(StatusRequest) returns (SyncStatus);
  rpc StartSync(SyncRequest) returns (SyncResponse);
  rpc PauseSync(PauseRequest) returns (PauseResponse);
  rpc UploadFile(stream UploadChunk) returns (UploadResponse);
  rpc DownloadFile(DownloadRequest) returns (stream DownloadChunk);
}

message StatusRequest {
  string path = 1;
}

message SyncStatus {
  string state = 1;  // "synced", "syncing", "error", "paused"
  int64 bytes_uploaded = 2;
  int64 bytes_downloaded = 3;
  repeated FileStatus files = 4;
}

message FileStatus {
  string path = 1;
  string state = 2;
  int64 progress_bytes = 3;
  int64 total_bytes = 4;
}
```

**Current Phase Features:**
- ✅ Firebase Authentication (Email/Password)
- ✅ Virtual file system integration (Windows CfAPI / macOS File Provider)
- ✅ Upload videos with progress tracking
- ✅ Folder navigation and creation
- ✅ On-demand file hydration (download when accessed)
- ✅ Automatic background sync
- ✅ Display user's file hierarchy

**Future Features (Phase 3+):**
- ⏳ Real-time transcode status monitoring in UI
- ⏳ Video preview/playback within app
- ⏳ Download original and transcoded versions
- ⏳ Selective sync configuration
- ⏳ Sharing and permissions management
- ⏳ Offline access for pinned files
- ⏳ Conflict resolution UI

**Implementation Resources:**
- **Windows**: [Microsoft Cloud Mirror Sample](https://github.com/microsoft/Windows-classic-samples/tree/main/Samples/CloudMirror)
- **macOS**: [Apple File Provider Documentation](https://developer.apple.com/documentation/fileprovider) + [WWDC Videos](https://developer.apple.com/videos/play/wwdc2019/719/)
- **Tauri + gRPC**: [tonic](https://github.com/hyperium/tonic) for Rust gRPC client

**Development Workflow:**
1. Develop Tauri UI with mocked sync backend
2. Build Windows adapter separately using Visual Studio
3. Build macOS adapter separately using Xcode
4. Integrate via gRPC IPC layer
5. Test end-to-end sync flows on each platform

## Configuration Files

### infrastructure/main-server/docker-compose.yml
Unified orchestration for main server:
- Kafka (KRaft mode)
- Normalizer service
- Backend API
- Kafka UI (optional monitoring)

### infrastructure/main-server/.env
```env
# Kafka
KAFKA_IMAGE=bitnami/kafka:3.7
KAFKA_BOOTSTRAP_SERVERS=kafka:9092

# MinIO (existing instance)
MINIO_ENDPOINT=https://s3.jumpermedia.co
MINIO_ACCESS_KEY=admin
MINIO_SECRET_KEY=OI*HFodkx
MINIO_REGION=us-central

# Kafka Topics
TOPIC_MINIO_EVENTS=minio-object-created
TOPIC_TRANSCODE_REQUEST=video-transcode-request
TOPIC_TRANSCODE_COMPLETED=video-transcode-completed
TOPIC_TRANSCODE_DLQ=video-transcode-deadletter

# Normalizer
NORMALIZER_PROFILES=h264_1080p,h264_720p,h264_480p
NORMALIZER_PREFIX=uploads/raw/
NORMALIZER_SUFFIXES=.mp4,.mov,.mkv,.avi,.r3d,.braw,.crm,.mxf,.arw

# Backend API
API_PORT=3000
API_HOST=0.0.0.0

# Firebase
FIREBASE_PROJECT_ID=your-project-id
FIREBASE_PRIVATE_KEY_PATH=/config/firebase-admin-key.json
```

### infrastructure/worker-nodes/worker-config.toml
Configuration template for GPU worker nodes:
```toml
[kafka]
brokers = ["10.0.1.100:9092"]  # Main server Kafka IP
consumer_group = "transcode-workers"
topics = ["video-transcode-request"]

[minio]
endpoint = "https://s3.jumpermedia.co"
access_key = "WORKER_ACCESS_KEY"
secret_key = "WORKER_SECRET_KEY"
region = "us-central"

[firebase]
project_id = "your-project-id"
credentials_path = "/etc/transcode-worker/firebase-key.json"

[worker]
node_id = "worker-01"  # Unique per node
max_concurrent = 8
temp_dir = "/mnt/nvme/transcode-temp"
gpu_enabled = true
gpu_type = "nvidia"  # Options: "nvidia", "intel", "none"

[ffmpeg]
binary_path = "/usr/bin/ffmpeg"
hwaccel = "cuda"  # Options: "cuda", "qsv", "none"
log_level = "warning"
```

### config/worker-nodes.json
```json
{
  "nodes": [
    {
      "id": "worker-01",
      "ip": "192.168.1.10",
      "status": "active",
      "max_concurrent": 8
    },
    {
      "id": "worker-02",
      "ip": "192.168.1.11",
      "status": "active",
      "max_concurrent": 8
    }
  ]
}
```

### config/transcode-profiles.json
GPU-aware transcoding profiles:
```json
{
  "profiles": {
    "h264_480p": {
      "resolution": "854x480",
      "video_codec": "libx264",
      "video_codec_gpu": "h264_nvenc",
      "video_bitrate": "1500k",
      "audio_codec": "aac",
      "audio_bitrate": "128k",
      "preset": "medium",
      "gpu_preset": "p4",
      "crf": 23
    },
    "h264_720p": {
      "resolution": "1280x720",
      "video_codec": "libx264",
      "video_codec_gpu": "h264_nvenc",
      "video_bitrate": "3000k",
      "audio_codec": "aac",
      "audio_bitrate": "192k",
      "preset": "medium",
      "gpu_preset": "p4",
      "crf": 23
    },
    "h264_1080p": {
      "resolution": "1920x1080",
      "video_codec": "libx264",
      "video_codec_gpu": "h264_nvenc",
      "video_bitrate": "6000k",
      "audio_codec": "aac",
      "audio_bitrate": "256k",
      "preset": "medium",
      "gpu_preset": "p4",
      "crf": 23
    }
  },
  "thumbnail": {
    "format": "image2",
    "codec": "mjpeg",
    "quality": 2,
    "frames": 1,
    "position": "00:00:01"
  }
}
```
      "audio_bitrate": "128k",
      "preset": "medium",
      "crf": 23
    },
    "h264_720p": {
      "resolution": "1280x720",
      "video_codec": "libx264",
      "video_bitrate": "3000k",
      "audio_codec": "aac",
      "audio_bitrate": "192k",
      "preset": "medium",
      "crf": 23
    },
    "h264_1080p": {
      "resolution": "1920x1080",
      "video_codec": "libx264",
      "video_bitrate": "6000k",
      "audio_codec": "aac",
      "audio_bitrate": "256k",
      "preset": "medium",
      "crf": 23
    }
  }
}
```

## Setup Instructions

### Prerequisites

**Main Server (Ubuntu 22):**
1. MinIO installed and running (existing)
2. Docker and Docker Compose installed
3. Nginx configured as reverse proxy for MinIO
4. `mc` (MinIO Client) installed for configuration
5. Firewall rules allowing Kafka port 9092 from worker nodes

**GPU Worker Nodes:**
1. Ubuntu 22 or compatible Linux
2. NVIDIA GPU with latest drivers (for NVENC) OR Intel GPU with QSV support
3. FFmpeg compiled with GPU support
4. Network access to main server (Kafka broker and MinIO endpoint)
5. Fast NVMe storage for temporary transcode files

**Development Environment:**
1. Rust toolchain (latest stable)
2. Node.js 18+ (for backend API)
3. Firebase project created with Admin SDK credentials

### Step 1: Main Server Setup

#### 1.1 Network Setup
```bash
# Create Docker network for services
docker network create events-net

# Connect existing MinIO container to network
docker network connect events-net minio
```

#### 1.2 Deploy Infrastructure Services
```bash
cd video-transcoding-pipeline/infrastructure/main-server

# Configure environment
cp .env.example .env
nano .env  # Edit with your credentials

# Start Kafka, Normalizer, Backend API, and Kafka UI
docker-compose up -d

# Verify services are running
docker-compose ps
```

#### 1.3 Create Kafka Topics
```bash
# Run topic creation script
../scripts/create-kafka-topics.sh

# Verify topics created
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list
```

#### 1.4 Configure MinIO Event Notifications
```bash
# Set MinIO alias
mc alias set local https://s3.jumpermedia.co admin 'OI*HFodkx'

# Configure Kafka notification target
mc admin config set local notify_kafka:kafka1 \
  brokers="kafka:9092" \
  topic="minio-object-created" \
  tls="off" \
  sasl="off" \
  queue_dir="/data/.minio-kafka-queue" \
  queue_limit="100000"

# Restart MinIO to apply config
mc admin service restart local

# Subscribe buckets to events
for b in creation-public contentcreation jm-test public-jm-test testbucket; do
  mc event add local/$b arn:minio:sqs:us-central:kafka1:kafka \
    --event put \
    --suffix .mp4 --suffix .mov --suffix .mkv --suffix .avi \
    --suffix .r3d --suffix .braw --suffix .crm \
    --suffix .mxf --suffix .arw \
    --prefix uploads/raw/
done

# Verify event configuration
mc event list local/creation-public
```

#### 1.5 Configure Firewall
```bash
# Allow Kafka port from worker nodes
sudo ufw allow from <worker-node-ip> to any port 9092 comment 'Kafka from worker'

# Allow MinIO API (if not already configured)
sudo ufw allow 443/tcp comment 'HTTPS MinIO API'
```

### Step 2: GPU Worker Node Setup

#### 2.1 Prepare Worker Server
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install basic dependencies
sudo apt install -y curl wget git build-essential pkg-config \
  libssl-dev librdkafka-dev

# Create dedicated user
sudo useradd -r -s /bin/bash -d /opt/transcode-worker transcode
sudo mkdir -p /opt/transcode-worker
sudo chown transcode:transcode /opt/transcode-worker
```

#### 2.2 Install FFmpeg with GPU Support

**For NVIDIA GPUs:**
```bash
# Install NVIDIA drivers and CUDA toolkit
sudo apt install -y nvidia-driver-535 nvidia-cuda-toolkit

# Verify GPU is detected
nvidia-smi

# Install FFmpeg with NVENC support
sudo apt install -y ffmpeg

# Verify NVENC encoder is available
ffmpeg -encoders | grep nvenc
```

**For Intel GPUs:**
```bash
# Install Intel Media SDK
sudo apt install -y intel-media-va-driver-non-free

# Install FFmpeg with QSV support
sudo apt install -y ffmpeg

# Verify QSV encoder is available
ffmpeg -encoders | grep qsv
```

#### 2.3 Create NVMe Temp Directory
```bash
# Create temp directory on fast NVMe storage
sudo mkdir -p /mnt/nvme/transcode-temp
sudo chown transcode:transcode /mnt/nvme/transcode-temp
sudo chmod 750 /mnt/nvme/transcode-temp
```

#### 2.4 Deploy Worker Binary
```bash
# Copy pre-compiled binary from build server
scp transcode-worker transcode@worker-node:/tmp/

# Install binary
sudo cp /tmp/transcode-worker /usr/local/bin/
sudo chmod +x /usr/local/bin/transcode-worker
sudo chown root:root /usr/local/bin/transcode-worker
```

#### 2.5 Configure Worker
```bash
# Create config directory
sudo mkdir -p /etc/transcode-worker
sudo chown transcode:transcode /etc/transcode-worker

# Copy configuration
sudo nano /etc/transcode-worker/worker-config.toml
```

**Edit worker-config.toml:**
```toml
[kafka]
brokers = ["10.0.1.100:9092"]  # Replace with main server IP
consumer_group = "transcode-workers"
topics = ["video-transcode-request"]

[minio]
endpoint = "https://s3.jumpermedia.co"
access_key = "your-worker-access-key"  # Create dedicated IAM user
secret_key = "your-worker-secret-key"
region = "us-central"

[firebase]
project_id = "your-firebase-project-id"
credentials_path = "/etc/transcode-worker/firebase-key.json"

[worker]
node_id = "worker-01"  # Unique identifier for this node
max_concurrent = 8
temp_dir = "/mnt/nvme/transcode-temp"
gpu_enabled = true
gpu_type = "nvidia"  # Change to "intel" or "none" as needed

[ffmpeg]
binary_path = "/usr/bin/ffmpeg"
hwaccel = "cuda"  # Change to "qsv" for Intel or "none" for CPU
log_level = "warning"
```

#### 2.6 Copy Firebase Credentials
```bash
# Copy Firebase service account key
sudo cp firebase-key.json /etc/transcode-worker/
sudo chown transcode:transcode /etc/transcode-worker/firebase-key.json
sudo chmod 600 /etc/transcode-worker/firebase-key.json
```

#### 2.7 Install Systemd Service
```bash
# Create service file
sudo nano /etc/systemd/system/transcode-worker.service
```

**Service file content:**
```ini
[Unit]
Description=Video Transcode Worker (GPU-accelerated)
After=network.target

[Service]
Type=simple
User=transcode
Group=transcode
WorkingDirectory=/opt/transcode-worker
ExecStart=/usr/local/bin/transcode-worker --config /etc/transcode-worker/worker-config.toml
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

# Resource limits
LimitNOFILE=65536
TasksMax=infinity

# Environment
Environment="RUST_LOG=info"

[Install]
WantedBy=multi-user.target
```

#### 2.8 Start Worker Service
```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable service to start on boot
sudo systemctl enable transcode-worker

# Start service
sudo systemctl start transcode-worker

# Check status
sudo systemctl status transcode-worker

# View logs
sudo journalctl -u transcode-worker -f
```

#### 2.9 Register Worker Node
Update `config/worker-nodes.json` on main server:
```json
{
  "nodes": [
    {
      "id": "worker-01",
      "ip": "192.168.1.10",
      "hostname": "gpu-worker-01",
      "gpu_type": "nvidia",
      "gpu_model": "RTX 4090",
      "status": "active",
      "max_concurrent": 8,
      "added_date": "2025-10-03T10:00:00Z"
    }
  ]
}
```

### Step 3: Test End-to-End Pipeline

#### 3.1 Upload Test Video
```bash
# Upload a test video to MinIO
mc cp test-video.mp4 local/creation-public/uploads/raw/

# Or use presigned URL from backend API
curl -X POST http://localhost:3000/api/upload/request \
  -H "Authorization: Bearer $FIREBASE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "test-video.mp4",
    "size": 52428800,
    "parent_id": "root"
  }'
```

#### 3.2 Monitor Pipeline
```bash
# Watch Kafka messages in UI
# Open browser: http://main-server-ip:8080

# Monitor normalizer logs
docker logs -f normalizer

# Monitor worker logs on GPU node
sudo journalctl -u transcode-worker -f

# Check MinIO for transcoded outputs
mc ls local/creation-public/test-video-id/
```

#### 3.3 Verify Firebase Update
Check Firestore console for updated document with `transcoded_url` field populated.

### Step 4: Deploy Backend API

```bash
cd services/backend-api

# Install dependencies
npm install

# Configure environment
cp .env.example .env
nano .env

# Start API server (or use PM2 for production)
npm start

# Or with Docker
docker build -t backend-api .
docker run -d -p 3000:3000 --name backend-api backend-api
```

## Monitoring

### Kafka UI
Access at: `http://main-server-ip:8080`

**Key Metrics to Monitor:**
- Consumer lag on `transcode-workers` group
- Message throughput on all topics
- Failed message count in DLQ topic
- Partition distribution

### MinIO Console
Access at: `https://console.jumpermedia.co` (via Nginx)

**Monitor:**
- Bucket storage usage
- Upload/download bandwidth
- API request rates
- Event notification queue status

### Worker Health Checks

**Check worker status:**
```bash
sudo systemctl status transcode-worker
```

**View real-time logs:**
```bash
sudo journalctl -u transcode-worker -f --since "10 minutes ago"
```

**Monitor GPU usage (NVIDIA):**
```bash
watch -n 1 nvidia-smi
```

**Monitor GPU usage (Intel):**
```bash
intel_gpu_top
```

**Check worker metrics (if exposed):**
```bash
curl http://worker-node-ip:9090/metrics
# Returns: active jobs, completed jobs, failed jobs, GPU utilization, etc.
```

### Infrastructure Monitoring

**Docker services on main server:**
```bash
docker-compose ps
docker stats kafka normalizer backend-api
```

**Kafka broker health:**
```bash
docker exec kafka kafka-broker-api-versions.sh --bootstrap-server localhost:9092
```

**Check topic lag:**
```bash
docker exec kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group transcode-workers \
  --describe
```

## Development Workflow

### Testing Upload Pipeline
1. Upload video to MinIO `uploads/raw/` prefix
2. Check Kafka UI for event in `minio-object-created` topic
3. Verify normalizer created jobs in `video-transcode-request` topic
4. Monitor worker logs for transcode progress
5. Check Firebase for updated document with `transcoded_url` field

### Adding New Worker Node
1. Deploy worker binary to new server
2. Update `config/worker-nodes.json`
3. Start worker service
4. Verify consumption from Kafka topic

### Changing Transcode Profiles
1. Update `config/transcode-profiles.json`
2. Update `NORMALIZER_PROFILES` in `.env`
3. Restart normalizer service
4. Restart all worker nodes

## Security Considerations

- [ ] Firebase Admin SDK credentials secured (gitignored)
- [ ] MinIO access keys in environment variables only
- [ ] Kafka exposed only on internal network
- [ ] Worker nodes authenticate to MinIO and Firebase
- [ ] Backend API validates Firebase tokens on all endpoints
- [ ] Presigned URLs have expiration (default: 1 hour)
- [ ] Firestore security rules (to be implemented)

## Scalability

### Horizontal Scaling

**Adding Kafka Brokers:**
```bash
# Update docker-compose.yml to add more Kafka nodes
# Configure KAFKA_CFG_CONTROLLER_QUORUM_VOTERS with all broker IDs
# Increase partition count for better distribution
```

**Adding Worker Nodes:**
```bash
# On new GPU server
./scripts/deploy-worker-node.sh --node-id worker-02 --ip 192.168.1.11

# Update config/worker-nodes.json
{
  "nodes": [
    {"id": "worker-01", "ip": "192.168.1.10", "status": "active"},
    {"id": "worker-02", "ip": "192.168.1.11", "status": "active"}
  ]
}

# No restart needed - Kafka automatically balances consumers
```

**Scaling Normalizer:**
```bash
# Increase replicas in docker-compose.yml
docker-compose up -d --scale normalizer=3
```

**Scaling Backend API:**
```bash
# Deploy behind Nginx load balancer
# Add multiple API containers
docker-compose up -d --scale backend-api=3

# Configure Nginx upstream
upstream backend_api {
    server api-01:3000;
    server api-02:3000;
    server api-03:3000;
}
```

### Vertical Scaling

**Worker Concurrency:**
- Increase `max_concurrent` in worker config based on GPU capacity
- NVIDIA RTX 4090: 8-12 concurrent jobs
- NVIDIA A100: 16-24 concurrent jobs
- Monitor GPU memory usage with `nvidia-smi`

**Kafka Partitions:**
```bash
# Increase partitions for better parallelism
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --alter --topic video-transcode-request \
  --partitions 12
```

**MinIO Storage:**
- Add drives to existing storage pool
- Configure erasure coding for redundancy
- Enable bucket replication for geo-distribution

### Performance Optimization

**Worker Node Optimization:**
- Use NVMe SSDs for temp storage (`temp_dir`)
- Enable GPU persistence mode: `nvidia-smi -pm 1`
- Increase system file descriptors: `ulimit -n 65536`
- Tune TCP settings for large file transfers

**Kafka Optimization:**
- Increase `log.retention.hours` for longer message retention
- Enable compression: `compression.type=lz4`
- Increase `num.network.threads` and `num.io.threads`

**FFmpeg GPU Optimization:**
- Use preset `p4` or `p5` for NVENC (balance quality/speed)
- Enable two-pass encoding for better quality at lower bitrates
- Utilize GPU-specific features (B-frames, lookahead)

## Future Enhancements

### Phase 2: Desktop UI (Tauri)
- [ ] Implement file browser with folder navigation
- [ ] Upload with progress tracking
- [ ] Real-time transcode status via Firebase listeners
- [ ] Video preview and playback
- [ ] Download management

### Phase 3: Advanced Features
- [ ] Adaptive bitrate streaming (HLS/DASH)
- [ ] AI-generated captions/subtitles
- [ ] Video analytics and insights
- [ ] Collaborative folders and sharing
- [ ] Mobile apps (React Native)

### Phase 4: Production Hardening
- [ ] Implement Firestore security rules
- [ ] Add comprehensive error handling and retry logic
- [ ] Set up monitoring and alerting (Prometheus/Grafana)
- [ ] Implement backup and disaster recovery
- [ ] Load testing and performance optimization

## Troubleshooting

### MinIO Not Sending Events to Kafka

**Check MinIO notification config:**
```bash
mc admin config get local notify_kafka
```

**Verify Kafka connectivity from MinIO container:**
```bash
docker exec minio ping -c 3 kafka
docker exec minio nc -zv kafka 9092
```

**Check event subscription:**
```bash
mc event list local/creation-public

# Expected output:
# arn:minio:sqs:us-central:kafka1:kafka   s3:ObjectCreated:*   Filter: prefix="uploads/raw/" suffix=".mp4"
```

**Check MinIO event queue:**
```bash
# If Kafka was down, MinIO queues events
docker exec minio ls -lh /data/.minio-kafka-queue/
```

**Restart MinIO notification:**
```bash
mc admin service restart local
```

### Normalizer Not Consuming Messages

**Check consumer group status:**
```bash
docker exec kafka kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group normalizer \
  --describe
```

**View normalizer logs:**
```bash
docker logs -f normalizer

# Look for connection errors or parsing issues
```

**Manually consume from raw topic:**
```bash
docker exec kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic minio-object-created \
  --from-beginning \
  --max-messages 1
```

**Restart normalizer:**
```bash
docker-compose restart normalizer
```

### Worker Not Processing Jobs

**Check worker service status:**
```bash
sudo systemctl status transcode-worker

# If failed:
sudo journalctl -u transcode-worker -n 50 --no-pager
```

**Common Issues:**

**1. Kafka Connection Failed:**
```bash
# Test connectivity from worker node
nc -zv main-server-ip 9092

# Check firewall rules
sudo ufw status | grep 9092
```

**2. MinIO Authentication Failed:**
```bash
# Test MinIO credentials
export AWS_ACCESS_KEY_ID=worker-key
export AWS_SECRET_ACCESS_KEY=worker-secret
aws s3 ls --endpoint-url https://s3.jumpermedia.co

# Verify worker has correct credentials in config
cat /etc/transcode-worker/worker-config.toml
```

**3. FFmpeg GPU Encoding Failed:**
```bash
# Check GPU availability
nvidia-smi  # For NVIDIA
intel_gpu_top  # For Intel

# Test FFmpeg GPU encoder manually
ffmpeg -hwaccel cuda -i input.mp4 -c:v h264_nvenc output.mp4

# Check FFmpeg build supports GPU
ffmpeg -encoders | grep nvenc  # For NVIDIA
ffmpeg -encoders | grep qsv    # For Intel
```

**4. Firebase Permission Denied:**
```bash
# Verify service account key is valid
cat /etc/transcode-worker/firebase-key.json | jq .project_id

# Check file permissions
ls -l /etc/transcode-worker/firebase-key.json
# Should be: -rw------- 1 transcode transcode
```

**5. Temp Directory Full:**
```bash
# Check disk space
df -h /mnt/nvme/transcode-temp

# Clean up stuck temp files
sudo find /mnt/nvme/transcode-temp -type f -mtime +1 -delete
```

**Restart worker:**
```bash
sudo systemctl restart transcode-worker
```

### Kafka Topics Not Created

**List existing topics:**
```bash
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list
```

**Manually create missing topics:**
```bash
# Create transcode request topic
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic video-transcode-request \
  --partitions 6 \
  --replication-factor 1

# Repeat for other topics
```

### Messages Stuck in Dead Letter Queue

**Check DLQ messages:**
```bash
docker exec kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic video-transcode-deadletter \
  --from-beginning
```

**Common reasons for DLQ:**
- Corrupted video file (cannot be decoded)
- Invalid MinIO URL or missing file
- GPU out of memory
- FFmpeg crash

**Replay DLQ messages after fixing:**
```bash
# Consume from DLQ and republish to main topic
# Use custom script or Kafka Connect
```

### Desktop UI Connection Issues

**gRPC Server Not Responding:**

**Windows:**
```powershell
# Check if gRPC server is running
Get-Process | Where-Object {$_.Name -like "*CloudMirror*"}

# Check port
netstat -ano | findstr :50051
```

**macOS:**
```bash
# Check File Provider extension status
pluginkit -m -v -p com.apple.FileProvider

# Check gRPC server logs
log show --predicate 'subsystem == "com.yourapp.fileprovider"' --last 1h
```

**Tauri Can't Connect to Native Adapter:**
```bash
# Test gRPC endpoint
grpcurl -plaintext localhost:50051 list

# Check Tauri logs
# Windows: %APPDATA%\com.yourapp.tauri\logs
# macOS: ~/Library/Logs/com.yourapp.tauri
```

### Performance Issues

**Slow Transcoding:**
- Check GPU utilization: `nvidia-smi` or `intel_gpu_top`
- Verify using GPU encoder, not CPU fallback
- Check temp directory is on fast NVMe, not HDD
- Reduce `max_concurrent` if GPU memory is exhausted

**High Kafka Lag:**
- Add more worker nodes
- Increase partition count on topics
- Check worker logs for errors causing slow processing

**MinIO Upload Slow:**
- Check network bandwidth between worker and MinIO
- Verify MinIO has sufficient IOPS
- Use MinIO multipart upload for large files

## License

[Add your license here]

## Contributors

[Add contributors here]
