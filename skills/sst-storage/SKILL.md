---
name: sst-storage
description: Handle file storage with SST using S3 buckets, presigned URLs, file uploads, EFS for persistent storage, and bucket subscribers. Use this skill when uploading files, serving static assets, processing uploaded files with Lambda triggers, or setting up persistent file storage for containers.
license: MIT
metadata:
  author: sst
  version: "3.0"
---

# SST Storage

SST provides components for file storage including S3 buckets for object storage and EFS for persistent file systems in VPC environments.

## When to use this skill

Use this skill when:
- Setting up file uploads in your application
- Creating presigned URLs for direct uploads
- Processing uploaded files with Lambda triggers
- Serving static assets from S3
- Setting up persistent storage for containers

## S3 Buckets

### Basic Bucket

```typescript
const bucket = new sst.aws.Bucket("MyBucket");
```

### Public Bucket

For serving static files publicly:

```typescript
const bucket = new sst.aws.Bucket("MyBucket", {
  access: "public"
});
```

### With CORS

For browser uploads:

```typescript
const bucket = new sst.aws.Bucket("MyBucket", {
  cors: {
    allowOrigins: ["https://my-app.com"],
    allowMethods: ["GET", "PUT", "POST", "DELETE"],
    allowHeaders: ["*"],
    maxAge: "1 day"
  }
});
```

### With Lifecycle Rules

Auto-delete old files:

```typescript
const bucket = new sst.aws.Bucket("MyBucket", {
  transform: {
    bucket: (args) => {
      args.lifecycleRules = [{
        enabled: true,
        expiration: { days: 30 },
        prefix: "temp/"
      }];
    }
  }
});
```

## File Uploads

### Presigned URL Upload

Generate a presigned URL for direct browser upload:

```typescript
// sst.config.ts
const bucket = new sst.aws.Bucket("Uploads");

new sst.aws.Function("UploadApi", {
  url: true,
  handler: "src/upload.handler",
  link: [bucket]
});
```

```typescript
// src/upload.ts
import { Resource } from "sst";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({});

export async function handler(event: any) {
  const { filename, contentType } = JSON.parse(event.body);
  const key = `uploads/${Date.now()}-${filename}`;
  
  const command = new PutObjectCommand({
    Bucket: Resource.Uploads.name,
    Key: key,
    ContentType: contentType
  });
  
  const url = await getSignedUrl(s3, command, { expiresIn: 3600 });
  
  return {
    statusCode: 200,
    body: JSON.stringify({ url, key })
  };
}
```

### Frontend Upload

```typescript
// Upload from browser
async function uploadFile(file: File) {
  // Get presigned URL
  const response = await fetch("/api/upload", {
    method: "POST",
    body: JSON.stringify({
      filename: file.name,
      contentType: file.type
    })
  });
  const { url, key } = await response.json();
  
  // Upload directly to S3
  await fetch(url, {
    method: "PUT",
    body: file,
    headers: {
      "Content-Type": file.type
    }
  });
  
  return key;
}
```

### Presigned URL Download

```typescript
import { GetObjectCommand } from "@aws-sdk/client-s3";

export async function getDownloadUrl(key: string) {
  const command = new GetObjectCommand({
    Bucket: Resource.Uploads.name,
    Key: key
  });
  
  return getSignedUrl(s3, command, { expiresIn: 3600 });
}
```

## Bucket Subscribers

### Lambda Trigger on Upload

```typescript
const bucket = new sst.aws.Bucket("Uploads");

bucket.subscribe("src/process.handler", {
  events: ["s3:ObjectCreated:*"]
});
```

```typescript
// src/process.ts
import { S3Event } from "aws-lambda";
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";

const s3 = new S3Client({});

export async function handler(event: S3Event) {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key);
    
    console.log(`Processing: ${bucket}/${key}`);
    
    // Get the uploaded file
    const response = await s3.send(new GetObjectCommand({
      Bucket: bucket,
      Key: key
    }));
    
    // Process the file...
    const body = await response.Body?.transformToString();
    console.log("File content:", body);
  }
}
```

### Filter by Prefix

```typescript
bucket.subscribe("src/images.handler", {
  events: ["s3:ObjectCreated:*"],
  filterPrefix: "images/"
});

bucket.subscribe("src/documents.handler", {
  events: ["s3:ObjectCreated:*"],
  filterPrefix: "documents/"
});
```

### Filter by Suffix

```typescript
bucket.subscribe("src/pdf.handler", {
  events: ["s3:ObjectCreated:*"],
  filterSuffix: ".pdf"
});
```

### Queue Subscriber

For high-volume processing:

```typescript
const queue = new sst.aws.Queue("ProcessingQueue");

bucket.subscribeQueue(queue, {
  events: ["s3:ObjectCreated:*"]
});

queue.subscribe("src/process.handler");
```

## Serving Files

### Through CloudFront

```typescript
const bucket = new sst.aws.Bucket("Assets", {
  access: "public"
});

const cdn = new sst.aws.Cdn("AssetsCdn", {
  origins: [{ domainName: bucket.domain }]
});
```

### With Router

```typescript
const bucket = new sst.aws.Bucket("Assets");

const router = new sst.aws.Router("MyRouter", {
  routes: {
    "/assets/*": bucket
  }
});
```

### Static Site Assets

```typescript
const bucket = new sst.aws.Bucket("Assets", {
  access: "public"
});

new sst.aws.Nextjs("MyWeb", {
  link: [bucket],
  environment: {
    NEXT_PUBLIC_ASSETS_URL: bucket.url
  }
});
```

## EFS (Elastic File System)

For persistent storage in VPC:

### Basic Setup

```typescript
const vpc = new sst.aws.Vpc("MyVpc");
const efs = new sst.aws.Efs("MyStorage", { vpc });

// Mount in Lambda
new sst.aws.Function("MyFunction", {
  handler: "src/handler.main",
  vpc,
  volume: {
    efs,
    path: "/mnt/data"
  }
});
```

### With Container

```typescript
const vpc = new sst.aws.Vpc("MyVpc");
const efs = new sst.aws.Efs("MyStorage", { vpc });
const cluster = new sst.aws.Cluster("MyCluster", { vpc });

new sst.aws.Service("MyService", {
  cluster,
  volumes: [{
    efs,
    path: "/data"
  }]
});
```

### Access in Lambda

```typescript
// src/handler.ts
import fs from "fs";
import path from "path";

export async function handler() {
  const dataPath = "/mnt/data/myfile.json";
  
  // Write to EFS
  fs.writeFileSync(dataPath, JSON.stringify({ hello: "world" }));
  
  // Read from EFS
  const data = fs.readFileSync(dataPath, "utf-8");
  
  return {
    statusCode: 200,
    body: data
  };
}
```

## Image Processing

### Resize on Upload

```typescript
const uploads = new sst.aws.Bucket("Uploads");
const processed = new sst.aws.Bucket("Processed", {
  access: "public"
});

uploads.subscribe({
  handler: "src/resize.handler",
  link: [processed],
  nodejs: {
    install: ["sharp"]
  }
}, {
  events: ["s3:ObjectCreated:*"],
  filterPrefix: "images/"
});
```

```typescript
// src/resize.ts
import { S3Event } from "aws-lambda";
import { S3Client, GetObjectCommand, PutObjectCommand } from "@aws-sdk/client-s3";
import sharp from "sharp";
import { Resource } from "sst";

const s3 = new S3Client({});

export async function handler(event: S3Event) {
  for (const record of event.Records) {
    const key = decodeURIComponent(record.s3.object.key);
    
    // Get original image
    const response = await s3.send(new GetObjectCommand({
      Bucket: record.s3.bucket.name,
      Key: key
    }));
    
    const buffer = Buffer.from(await response.Body!.transformToByteArray());
    
    // Create thumbnail
    const thumbnail = await sharp(buffer)
      .resize(200, 200, { fit: "cover" })
      .toBuffer();
    
    // Upload to processed bucket
    await s3.send(new PutObjectCommand({
      Bucket: Resource.Processed.name,
      Key: `thumbnails/${key}`,
      Body: thumbnail,
      ContentType: "image/jpeg"
    }));
  }
}
```

## Multipart Uploads

For large files (> 5GB):

```typescript
// src/multipart.ts
import { Resource } from "sst";
import {
  S3Client,
  CreateMultipartUploadCommand,
  UploadPartCommand,
  CompleteMultipartUploadCommand
} from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({});

export async function initMultipartUpload(key: string) {
  const command = new CreateMultipartUploadCommand({
    Bucket: Resource.Uploads.name,
    Key: key
  });
  
  const { UploadId } = await s3.send(command);
  return UploadId;
}

export async function getPartUploadUrl(
  key: string,
  uploadId: string,
  partNumber: number
) {
  const command = new UploadPartCommand({
    Bucket: Resource.Uploads.name,
    Key: key,
    UploadId: uploadId,
    PartNumber: partNumber
  });
  
  return getSignedUrl(s3, command, { expiresIn: 3600 });
}

export async function completeMultipartUpload(
  key: string,
  uploadId: string,
  parts: { ETag: string; PartNumber: number }[]
) {
  const command = new CompleteMultipartUploadCommand({
    Bucket: Resource.Uploads.name,
    Key: key,
    UploadId: uploadId,
    MultipartUpload: { Parts: parts }
  });
  
  return s3.send(command);
}
```

## Bucket Notifications to Queue

For reliable processing:

```typescript
const bucket = new sst.aws.Bucket("Uploads");
const queue = new sst.aws.Queue("ProcessQueue");

bucket.subscribeQueue(queue, {
  events: ["s3:ObjectCreated:*"]
});

queue.subscribe("src/process.handler", {
  batch: { size: 10 }
});
```

## Best Practices

1. **Use presigned URLs**: Avoid routing uploads through Lambda
2. **Set lifecycle rules**: Auto-delete temporary files
3. **Use queues for processing**: Handle failures gracefully
4. **Enable versioning for critical data**: Prevent accidental deletion
5. **Use EFS for shared state**: Lambda + containers can share files
6. **Filter subscribers**: Only process relevant files

## Security

### Restrict Access

```typescript
const bucket = new sst.aws.Bucket("PrivateBucket", {
  access: "private"  // default
});

// Only linked functions can access
new sst.aws.Function("MyFunction", {
  handler: "src/handler.main",
  link: [bucket]  // Grants S3 permissions
});
```

### Bucket Policy

```typescript
const bucket = new sst.aws.Bucket("MyBucket", {
  transform: {
    policy: (args) => {
      args.policy = JSON.stringify({
        Version: "2012-10-17",
        Statement: [{
          Effect: "Deny",
          Principal: "*",
          Action: "s3:*",
          Resource: [bucket.arn, `${bucket.arn}/*`],
          Condition: {
            Bool: { "aws:SecureTransport": "false" }
          }
        }]
      });
    }
  }
});
```

## Troubleshooting

### CORS Errors

Ensure CORS is configured:

```typescript
const bucket = new sst.aws.Bucket("Uploads", {
  cors: {
    allowOrigins: ["*"],
    allowMethods: ["GET", "PUT", "POST"],
    allowHeaders: ["*"]
  }
});
```

### Access Denied

Check that your function is linked:

```typescript
new sst.aws.Function("MyFunction", {
  handler: "src/handler.main",
  link: [bucket]  // Required for access
});
```

### Large File Timeout

Use multipart uploads for files > 100MB and presigned URLs to bypass Lambda.
