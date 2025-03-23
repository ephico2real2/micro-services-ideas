# Running the Preview Generation Service Locally

This guide will walk you through setting up and running the Preview Generation Service locally. I'll cover everything including project structure, dependencies, configuration, and testing.

## 1. Project Setup

### Initial Setup

```bash
# Create a new directory for the service
mkdir preview-generation-service
cd preview-generation-service

# Initialize a new Node.js project
npm init -y

# Install required dependencies
npm install express cors dotenv mongoose multer ffmpeg-static fluent-ffmpeg aws-sdk fs-extra uuid morgan winston rimraf

# Install NFS-related dependencies
npm install nfs-client async-lock mount-node

# Install development dependencies
npm install nodemon concurrently --save-dev
```

### Project Structure

Create the following folder structure:

```
preview-generation-service/
├── .env                         # Environment variables
├── package.json                 # Project configuration
├── server.js                    # Main entry point
├── config/
│   ├── index.js                 # Configuration loader
│   ├── database.js              # Database configuration
│   └── storage.js               # Storage configuration
├── src/
│   ├── controllers/
│   │   └── previewController.js # HTTP handlers for preview routes
│   ├── models/
│   │   └── Preview.js           # MongoDB model for previews
│   ├── routes/
│   │   └── previewRoutes.js     # API routes
│   ├── services/
│   │   ├── previewService.js    # Core preview generation logic
│   │   └── storageService.js    # Storage abstraction (S3, NFS, local)
│   └── utils/
│       ├── logger.js            # Logging utility
│       └── ffmpeg.js            # FFmpeg helper functions
├── storage/
│   ├── local/                   # Local storage directory
│   │   └── previews/            # Local preview storage
│   └── temp/                    # Temporary processing directory
└── tests/
    └── preview.test.js          # Tests for preview generation
```

## 2. Configuration Files

### `.env` File

```env
# Server configuration
PORT=3001
NODE_ENV=development
API_PREFIX=/api

# Database configuration
MONGODB_URI=mongodb://localhost:27017/preview-service

# Storage configuration
STORAGE_TYPE=local      # Options: local, s3, nfs
FFMPEG_PATH=            # Optional: Custom ffmpeg path if not using ffmpeg-static

# Local storage options
LOCAL_STORAGE_PATH=./storage/local

# S3 configuration
S3_ACCESS_KEY=your-access-key
S3_SECRET_KEY=your-secret-key
S3_BUCKET=your-bucket-name
S3_REGION=us-east-1
S3_ENDPOINT=            # Custom endpoint for S3-compatible storage (e.g., MinIO, Ceph)
S3_USE_SSL=true         # Whether to use SSL for S3 connections
S3_FORCE_PATH_STYLE=false  # Use path-style URLs instead of virtual-hosted URLs
S3_SIGNATURE_VERSION=v4    # Signature version for AWS S3 API
S3_SSE_ALGORITHM=          # Server-side encryption (AES256, aws:kms)
S3_RETRY_MAX=3             # Maximum number of retry attempts
S3_RETRY_DELAY=1000        # Base delay between retries in milliseconds

# NFS configuration
NFS_SERVER=192.168.1.100   # NFS server hostname or IP
NFS_PORT=2049              # NFS port (default: 2049)
NFS_EXPORT_PATH=/exports/media    # Export path on the NFS server
NFS_MOUNT_PATH=/mnt/nfs/previews  # Local mount point
NFS_MOUNT_OPTIONS=rw,sync,hard,intr  # Mount options
NFS_SECURITY_MODE=sys     # Options: sys, krb5, krb5i, krb5p
NFS_USER=                 # Optional username for authentication
NFS_PASSWORD=             # Optional password
NFS_UID=                  # Optional UID mapping
NFS_GID=                  # Optional GID mapping
NFS_AUTO_MOUNT=true       # Whether to mount automatically on startup
NFS_RETRY_COUNT=3         # Number of times to retry NFS operations
NFS_TIMEOUT=30000         # Timeout for NFS operations in milliseconds

# Preview generation settings
MAX_PREVIEW_DURATION=5    # Maximum preview duration in seconds
DEFAULT_PREVIEW_COUNT=3   # Number of previews to generate per video
DEFAULT_PREVIEW_FORMAT=mp4
```

### `config/index.js`

```javascript
const dotenv = require('dotenv');
const path = require('path');

// Load environment variables
dotenv.config();

module.exports = {
  server: {
    port: process.env.PORT || 3001,
    env: process.env.NODE_ENV || 'development',
    apiPrefix: process.env.API_PREFIX || '/api'
  },
  
  mongodb: {
    uri: process.env.MONGODB_URI || 'mongodb://localhost:27017/preview-service'
  },
  
  storage: {
    // Storage type: local, s3, or nfs
    type: process.env.STORAGE_TYPE || 'local',
    
    // Local storage options
    local: {
      path: process.env.LOCAL_STORAGE_PATH || path.join(__dirname, '../storage/local')
    },
    
    // S3 storage options
    s3: {
      accessKey: process.env.S3_ACCESS_KEY,
      secretKey: process.env.S3_SECRET_KEY,
      bucket: process.env.S3_BUCKET,
      region: process.env.S3_REGION || 'us-east-1',
      endpoint: process.env.S3_ENDPOINT,
      useSsl: process.env.S3_USE_SSL === 'true',
      forcePathStyle: process.env.S3_FORCE_PATH_STYLE === 'true',
      signatureVersion: process.env.S3_SIGNATURE_VERSION || 'v4',
      sseAlgorithm: process.env.S3_SSE_ALGORITHM,
      retryMax: parseInt(process.env.S3_RETRY_MAX) || 3,
      retryDelay: parseInt(process.env.S3_RETRY_DELAY) || 1000
    },
    
    // NFS storage options
    nfs: {
      server: process.env.NFS_SERVER,
      port: parseInt(process.env.NFS_PORT) || 2049,
      exportPath: process.env.NFS_EXPORT_PATH,
      mountPath: process.env.NFS_MOUNT_PATH,
      mountOptions: process.env.NFS_MOUNT_OPTIONS || 'rw,sync,hard,intr',
      securityMode: process.env.NFS_SECURITY_MODE || 'sys',
      user: process.env.NFS_USER,
      password: process.env.NFS_PASSWORD,
      uid: process.env.NFS_UID,
      gid: process.env.NFS_GID,
      autoMount: process.env.NFS_AUTO_MOUNT === 'true',
      retryCount: parseInt(process.env.NFS_RETRY_COUNT) || 3,
      timeout: parseInt(process.env.NFS_TIMEOUT) || 30000
    },
    
    // Temporary storage path for processing
    tempPath: path.join(__dirname, '../storage/temp')
  },
  
  ffmpeg: {
    path: process.env.FFMPEG_PATH || require('ffmpeg-static')
  },
  
  preview: {
    maxDuration: parseInt(process.env.MAX_PREVIEW_DURATION) || 5,
    defaultCount: parseInt(process.env.DEFAULT_PREVIEW_COUNT) || 3,
    defaultFormat: process.env.DEFAULT_PREVIEW_FORMAT || 'mp4',
    
    // Quality presets
    quality: {
      low: {
        videoBitrate: 500,    // kbps
        audioBitrate: 64,     // kbps
        width: 480,
        height: 270,          // 16:9 aspect ratio
        framerate: 24
      },
      medium: {
        videoBitrate: 800,
        audioBitrate: 96,
        width: 640,
        height: 360,
        framerate: 24
      },
      high: {
        videoBitrate: 1200,
        audioBitrate: 128,
        width: 854,
        height: 480,
        framerate: 30
      }
    }
  }
};
```

### `config/database.js`

```javascript
const mongoose = require('mongoose');
const config = require('./index');
const logger = require('../src/utils/logger');

// Configure Mongoose
mongoose.set('strictQuery', false);

// Connect to MongoDB
async function connectDB() {
  try {
    await mongoose.connect(config.mongodb.uri, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    
    logger.info(`MongoDB connected: ${mongoose.connection.host}`);
    return mongoose.connection;
  } catch (error) {
    logger.error(`MongoDB connection error: ${error.message}`);
    process.exit(1);
  }
}

module.exports = { connectDB };
```

### `config/storage.js`

```javascript
const fs = require('fs-extra');
const path = require('path');
const AWS = require('aws-sdk');
const { exec } = require('child_process');
const util = require('util');
const execPromise = util.promisify(exec);
const AsyncLock = require('async-lock');
const config = require('./index');
const logger = require('../src/utils/logger');

// Create async lock for NFS operations
const lock = new AsyncLock();

// Initialize storage based on configuration
async function initializeStorage() {
  const storageType = config.storage.type;
  logger.info(`Initializing ${storageType} storage`);
  
  try {
    switch (storageType) {
      case 'local':
        await initializeLocalStorage();
        break;
      case 's3':
        await initializeS3Storage();
        break;
      case 'nfs':
        await initializeNFSStorage();
        break;
      default:
        throw new Error(`Unsupported storage type: ${storageType}`);
    }
    
    // Always ensure temp directory exists
    await fs.ensureDir(config.storage.tempPath);
    logger.info('Temporary directory initialized');
    
    return true;
  } catch (error) {
    logger.error(`Storage initialization error: ${error.message}`);
    throw error;
  }
}

// Initialize local storage
async function initializeLocalStorage() {
  const localPath = config.storage.local.path;
  const previewsPath = path.join(localPath, 'previews');
  
  // Ensure directories exist
  await fs.ensureDir(localPath);
  await fs.ensureDir(previewsPath);
  
  logger.info(`Local storage initialized at ${localPath}`);
  return true;
}

// Initialize S3 storage
async function initializeS3Storage() {
  const s3Config = config.storage.s3;
  
  // Validate required settings
  if (!s3Config.accessKey || !s3Config.secretKey || !s3Config.bucket) {
    throw new Error('Missing required S3 configuration');
  }
  
  // Create S3 client
  const s3Options = {
    accessKeyId: s3Config.accessKey,
    secretAccessKey: s3Config.secretKey,
    region: s3Config.region,
    maxRetries: s3Config.retryMax,
    retryDelayOptions: {
      base: s3Config.retryDelay
    },
    signatureVersion: s3Config.signatureVersion
  };
  
  // Add SSL configuration
  if (s3Config.useSsl !== undefined) {
    s3Options.sslEnabled = s3Config.useSsl;
  }
  
  // Add custom endpoint if provided (for MinIO, Ceph, etc.)
  if (s3Config.endpoint) {
    s3Options.endpoint = s3Config.endpoint;
    s3Options.s3ForcePathStyle = s3Config.forcePathStyle;
  }
  
  const s3Client = new AWS.S3(s3Options);
  
  // Test connection by checking if bucket exists
  try {
    await s3Client.headBucket({ Bucket: s3Config.bucket }).promise();
    logger.info(`S3 storage initialized with bucket: ${s3Config.bucket}`);
  } catch (error) {
    // If bucket doesn't exist and we have permission, create it
    if (error.code === 'NotFound' || error.code === 'NoSuchBucket') {
      try {
        const createParams = { Bucket: s3Config.bucket };
        
        // Add location constraint if region is specified
        if (s3Config.region && s3Config.region !== 'us-east-1') {
          createParams.CreateBucketConfiguration = {
            LocationConstraint: s3Config.region
          };
        }
        
        await s3Client.createBucket(createParams).promise();
        logger.info(`Created S3 bucket: ${s3Config.bucket}`);
      } catch (createError) {
        throw new Error(`Failed to create S3 bucket: ${createError.message}`);
      }
    } else {
      throw new Error(`S3 initialization error: ${error.message}`);
    }
  }
  
  // Set global s3Client for use throughout the application
  global.s3Client = s3Client;
  
  return true;
}

// Helper to check if a directory is mounted
async function checkIfMounted(mountPath) {
  try {
    // Use mount command to check if path is mounted
    const { stdout } = await execPromise('mount');
    return stdout.includes(mountPath);
  } catch (error) {
    logger.error(`Error checking mount status: ${error.message}`);
    return false;
  }
}

// Initialize NFS storage
async function initializeNFSStorage() {
  const nfsConfig = config.storage.nfs;
  
  // Validate required settings
  if (!nfsConfig.server || !nfsConfig.exportPath) {
    throw new Error('Missing required NFS configuration (server or exportPath)');
  }
  
  if (!nfsConfig.mountPath) {
    throw new Error('Missing required NFS mount path');
  }
  
  // Ensure mount path directory exists
  await fs.ensureDir(nfsConfig.mountPath);
  
  // Check if NFS is already mounted
  const isMounted = await checkIfMounted(nfsConfig.mountPath);
  
  if (!isMounted && nfsConfig.autoMount) {
    logger.info(`Attempting to mount NFS: ${nfsConfig.server}:${nfsConfig.exportPath} to ${nfsConfig.mountPath}`);
    
    // Construct mount command with options
    let mountCmd = `mount -t nfs`;
    
    // Add port if specified and not default
    if (nfsConfig.port && nfsConfig.port !== 2049) {
      mountCmd += ` -o port=${nfsConfig.port}`;
    }
    
    // Start building options string
    let optionsArray = [];
    
    // Add security options if specified
    if (nfsConfig.securityMode && nfsConfig.securityMode !== 'sys') {
      optionsArray.push(`sec=${nfsConfig.securityMode}`);
    }
    
    // Add user/password if provided
    if (nfsConfig.user) {
      optionsArray.push(`user=${nfsConfig.user}`);
      
      if (nfsConfig.password) {
        // Note: Exposing password in command line is a security risk
        // Better to use credentials file in production
        optionsArray.push(`password=${nfsConfig.password}`);
      }
    }
    
    // Add UID/GID mapping if provided
    if (nfsConfig.uid) {
      optionsArray.push(`uid=${nfsConfig.uid}`);
    }
    if (nfsConfig.gid) {
      optionsArray.push(`gid=${nfsConfig.gid}`);
    }
    
    // Add custom mount options if specified
    if (nfsConfig.mountOptions) {
      // Split by comma and add to options array
      const customOptions = nfsConfig.mountOptions.split(',');
      optionsArray = [...optionsArray, ...customOptions];
    }
    
    // Add options to command if we have any
    if (optionsArray.length > 0) {
      mountCmd += ` -o ${optionsArray.join(',')}`;
    }
    
    // Complete the command with source and target
    mountCmd += ` ${nfsConfig.server}:${nfsConfig.exportPath} ${nfsConfig.mountPath}`;
    
    // Execute mount command
    try {
      await execPromise(mountCmd);
      logger.info(`NFS mounted successfully: ${nfsConfig.server}:${nfsConfig.exportPath} -> ${nfsConfig.mountPath}`);
    } catch (error) {
      throw new Error(`Failed to mount NFS: ${error.message}`);
    }
  } else if (!isMounted && !nfsConfig.autoMount) {
    throw new Error(`NFS not mounted at ${nfsConfig.mountPath} and auto-mount is disabled`);
  } else {
    logger.info(`NFS already mounted at ${nfsConfig.mountPath}`);
  }
  
  // Create previews directory within the NFS mount
  try {
    const previewsPath = path.join(nfsConfig.mountPath, 'previews');
    await fs.ensureDir(previewsPath);
    logger.info(`Created previews directory at ${previewsPath}`);
  } catch (error) {
    throw new Error(`Failed to create previews directory in NFS: ${error.message}`);
  }
  
  logger.info(`NFS storage initialized at ${nfsConfig.mountPath}`);
  return true;
}

// Handle cleanup on application shutdown
function shutdownStorage() {
  const storageType = config.storage.type;
  
  if (storageType === 'nfs' && config.storage.nfs.autoMount) {
    const mountPath = config.storage.nfs.mountPath;
    
    // Unmount NFS if it was auto-mounted
    try {
      // Using sync version for shutdown
      execSync(`umount ${mountPath}`);
      logger.info(`NFS unmounted: ${mountPath}`);
    } catch (error) {
      logger.error(`Failed to unmount NFS: ${error.message}`);
    }
  }
}

module.exports = { 
  initializeStorage,
  shutdownStorage
};
```

## 3. Core Service Implementation

### `src/services/storageService.js`

```javascript
const fs = require('fs-extra');
const path = require('path');
const AWS = require('aws-sdk');
const { promisify } = require('util');
const AsyncLock = require('async-lock');
const config = require('../../config');
const logger = require('../utils/logger');

// Create async lock for file operations
const lock = new AsyncLock();

/**
 * Retry decorator function
 * @param {Function} fn - Function to retry
 * @param {number} maxRetries - Maximum number of retries
 * @param {number} baseDelay - Base delay in ms between retries
 * @returns {Function} - Decorated function with retry logic
 */
function withRetry(fn, maxRetries, baseDelay) {
  return async (...args) => {
    let lastError;
    
    for (let attempt = 1; attempt <= maxRetries + 1; attempt++) {
      try {
        return await fn(...args);
      } catch (error) {
        lastError = error;
        
        // Don't delay after the last attempt
        if (attempt <= maxRetries) {
          const delay = baseDelay * Math.pow(1.5, attempt - 1); // Exponential backoff
          logger.warn(`Operation failed, retrying (${attempt}/${maxRetries}): ${error.message}`);
          await new Promise(resolve => setTimeout(resolve, delay));
        }
      }
    }
    
    throw lastError;
  };
}

/**
 * Storage service with support for multiple backends
 */
class StorageService {
  constructor() {
    this.storageType = config.storage.type;
    
    // Set up retry configurations
    if (this.storageType === 's3') {
      this.retryConfig = {
        maxRetries: config.storage.s3.retryMax,
        baseDelay: config.storage.s3.retryDelay
      };
    } else if (this.storageType === 'nfs') {
      this.retryConfig = {
        maxRetries: config.storage.nfs.retryCount,
        baseDelay: 1000
      };
    } else {
      this.retryConfig = {
        maxRetries: 3,
        baseDelay: 1000
      };
    }
  }
  
  /**
   * Check storage health
   * @returns {Promise<Object>} Health status
   */
  async checkHealth() {
    try {
      switch (this.storageType) {
        case 'local':
          return await this.checkLocalHealth();
        case 's3':
          return await this.checkS3Health();
        case 'nfs':
          return await this.checkNFSHealth();
        default:
          throw new Error(`Unsupported storage type: ${this.storageType}`);
      }
    } catch (error) {
      logger.error(`Storage health check failed: ${error.message}`);
      return {
        healthy: false,
        error: error.message
      };
    }
  }
  
  /**
   * Check local storage health
   * @returns {Promise<Object>} Health status
   */
  async checkLocalHealth() {
    const localPath = config.storage.local.path;
    
    try {
      // Check if path exists and is writable
      await fs.access(localPath, fs.constants.W_OK);
      
      // Get disk space information
      const { stdout } = await promisify(require('child_process').exec)(`df -k "${localPath}" | tail -1`);
      const parts = stdout.trim().split(/\s+/);
      
      // Parse disk space info (typical df output format)
      const total = parseInt(parts[1]) * 1024; // Convert to bytes
      const used = parseInt(parts[2]) * 1024;
      const available = parseInt(parts[3]) * 1024;
      const usedPercent = parseInt(parts[4].replace('%', ''));
      
      return {
        healthy: true,
        type: 'local',
        path: localPath,
        space: {
          total,
          used,
          available,
          usedPercent
        }
      };
    } catch (error) {
      throw new Error(`Local storage health check failed: ${error.message}`);
    }
  }
  
  /**
   * Check S3 storage health
   * @returns {Promise<Object>} Health status
   */
  async checkS3Health() {
    try {
      const s3Client = global.s3Client;
      const bucketName = config.storage.s3.bucket;
      
      // Check if bucket exists
      await s3Client.headBucket({ Bucket: bucketName }).promise();
      
      return {
        healthy: true,
        type: 's3',
        bucket: bucketName,
        region: config.storage.s3.region,
        endpoint: config.storage.s3.endpoint || 'default AWS'
      };
    } catch (error) {
      throw new Error(`S3 storage health check failed: ${error.message}`);
    }
  }
  
  /**
   * Check NFS storage health
   * @returns {Promise<Object>} Health status
   */
  async checkNFSHealth() {
    const nfsPath = config.storage.nfs.mountPath;
    
    try {
      // Check if mount point exists and is writable
      await fs.access(nfsPath, fs.constants.W_OK);
      
      // Check if it's actually mounted
      const { stdout: mountOutput } = await promisify(require('child_process').exec)('mount');
      const isMounted = mountOutput.includes(nfsPath);
      
      if (!isMounted) {
        throw new Error(`NFS path ${nfsPath} exists but is not mounted`);
      }
      
      // Get disk space information
      const { stdout } = await promisify(require('child_process').exec)(`df -k "${nfsPath}" | tail -1`);
      const parts = stdout.trim().split(/\s+/);
      
      // Parse disk space info
      const total = parseInt(parts[1]) * 1024; // Convert to bytes
      const used = parseInt(parts[2]) * 1024;
      const available = parseInt(parts[3]) * 1024;
      const usedPercent = parseInt(parts[4].replace('%', ''));
      
      return {
        healthy: true,
        type: 'nfs',
        path: nfsPath,
        server: config.storage.nfs.server,
        exportPath: config.storage.nfs.exportPath,
        space: {
          total,
          used,
          available,
          usedPercent
        }
      };
    } catch (error) {
      throw new Error(`NFS storage health check failed: ${error.message}`);
    }
  }
  
  /**
   * Store a file in the configured storage system
   * @param {Buffer} fileBuffer - File content
   * @param {string} filePath - Relative path within storage
   * @param {string} contentType - MIME type of the file
   * @returns {Promise<Object>} - Storage details
   */
  async storeFile(fileBuffer, filePath, contentType) {
    try {
      switch (this.storageType) {
        case 'local':
          return await this.storeFileLocal(fileBuffer, filePath);
        case 's3':
          return await this.storeFileS3(fileBuffer, filePath, contentType);
        case 'nfs':
          return await this.storeFileNFS(fileBuffer, filePath);
        default:
          throw new Error(`Unsupported storage type: ${this.storageType}`);
      }
    } catch (error) {
      logger.error(`Error storing file: ${error.message}`);
      throw error;
    }
  }
  
  /**
   * Store file in local filesystem
   * @param {Buffer} fileBuffer - File content
   * @param {string} filePath - Relative path within local storage
   * @returns {Promise<Object>} - Storage details
   */
  async storeFileLocal(fileBuffer, filePath) {
    const fullPath = path.join(config.storage.local.path, filePath);
    
    // Wrap with retry
    const writeFileWithRetry = withRetry(
      async () => {
        // Ensure directory exists
        await fs.ensureDir(path.dirname(fullPath));
        
        // Write file
        await fs.writeFile(fullPath, fileBuffer);
      },
      this.retryConfig.maxRetries,
      this.retryConfig.baseDelay
    );
    
    await writeFileWithRetry();
    
    return {
      storageType: 'local',
      storagePath: filePath,
      fullPath,
      size: fileBuffer.length
    };
  }
  
  /**
   * Store file in S3-compatible storage
   * @param {Buffer} fileBuffer - File content
   * @param {string} filePath - Key path within S3 bucket
   * @param {string} contentType - MIME type of the file
   * @returns {Promise<Object>} - Storage details
   */
  async storeFileS3(fileBuffer, filePath, contentType) {
    // Use global s3Client
    const s3Client = global.s3Client;
    const bucket = config.storage.s3.bucket;
    
    // Prepare upload parameters
    const params = {
      Bucket: bucket,
      Key: filePath,
      Body: fileBuffer,
      ContentType: contentType
    };
    
    // Add server-side encryption if configured
    if (config.storage.s3.sseAlgorithm) {
      params.ServerSideEncryption = config.storage.s3.sseAlgorithm;
    }
    
    // Upload to S3 with built-in retry (AWS SDK handles retries)
    await s3Client.putObject(params).promise();
    
    return {
      storageType: 's3',
      storagePath: filePath,
      bucket,
      size: fileBuffer.length
    };
  }
  
  /**
   * Store file in NFS mount
   * @param {Buffer} fileBuffer - File content
   * @param {string} filePath - Relative path within NFS mount
   * @returns {Promise<Object>} - Storage details
   */
  async storeFileNFS(fileBuffer, filePath) {
    const fullPath = path.join(config.storage.nfs.mountPath, filePath);
    
    // Use lock to prevent concurrent access issues on NFS
    return await lock.acquire('nfs-write', async () => {
      // Wrap with retry
      const writeFileWithRetry = withRetry(
        async () => {
          // Ensure directory exists
          await fs.ensureDir(path.dirname(fullPath));
          
          // Write file with timeout
          const writePromise = fs.writeFile(fullPath, fileBuffer);
          
          // Add timeout to detect stalled NFS operations
          const timeoutPromise = new Promise((_, reject) => {
            setTimeout(() => {
              reject(new Error(`NFS write operation timed out after ${config.storage.nfs.timeout}ms`));
            }, config.storage.nfs.timeout);
          });
          
          await Promise.race([writePromise, timeoutPromise]);
        },
        this.retryConfig.maxRetries,
        this.retryConfig.baseDelay
      );
      
      await writeFileWithRetry();
      
      return {
        storageType: 'nfs',
        storagePath: filePath,
        fullPath,
        size: fileBuffer.length
      };
    });
  }
  
  /**
   * Get a file from storage
   * @param {string} storagePath - Path in storage
   * @param {string} storageType - Type of storage (defaults to configured type)
   * @returns {Promise<Buffer>} - File content
   */
  async getFile(storagePath, storageType = null) {
    const type = storageType || this.storageType;
    
    try {
      switch (type) {
        case 'local':
          return await this.getFileLocal(storagePath);
        case 's3':
          return await this.getFileS3(storagePath);
        case 'nfs':
          return await this.getFileNFS(storagePath);
        default:
          throw new Error(`Unsupported storage type: ${type}`);
      }
    } catch (error) {
      logger.error(`Error retrieving file: ${error.message}`);
      throw error;
    }
  }
  
  /**
   * Get file from local filesystem
   * @param {string} storagePath - Relative path within local storage
   * @returns {Promise<Buffer>} - File content
   */
  async getFileLocal(storagePath) {
    const fullPath = path.join(config.storage.local.path, storagePath);
    
    // Wrap with retry
    const readFileWithRetry = withRetry(
      () => fs.readFile(fullPath),
      this.retryConfig.maxRetries,
      this.retryConfig.baseDelay
    );
    
    return await readFileWithRetry();
  }
  
  /**
   * Get file from S3-compatible storage
   * @param {string} storagePath - Key path within S3 bucket
   * @returns {Promise<Buffer>} - File content
   */
  async getFileS3(storagePath) {
    // Use global s3Client
    const s3Client = global.s3Client;
    const bucket = config.storage.s3.bucket;
    
    // Get from S3 (AWS SDK handles retries internally)
    const response = await s3Client.getObject({
      Bucket: bucket,
      Key: storagePath
    }).promise();
    
    return response.Body;
  }
  
  /**
   * Get file from NFS mount
   * @param {string} storagePath - Relative path within NFS mount
   * @returns {Promise<Buffer>} - File content
   */
  async getFileNFS(storagePath) {
    const fullPath = path.join(config.storage.nfs.mountPath, storagePath);
    
    // Use lock to prevent concurrent access issues on NFS
    return await lock.acquire('nfs-read', async () => {
      // Wrap with retry
      const readFileWithRetry = withRetry(
        async () => {
          // Read file with timeout
          const readPromise = fs.readFile(fullPath);
          
          // Add timeout to detect stalled NFS operations
          const timeoutPromise = new Promise((_, reject) => {
            setTimeout(() => {
              reject(new Error(`NFS read operation timed out after ${config.storage.nfs.timeout}ms`));
            }, config.storage.nfs.timeout);
          });
          
          return await Promise.race([readPromise, timeoutPromise]);
        },
        this.retryConfig.maxRetries,
        this.retryConfig.baseDelay
      );
      
      return await readFileWithRetry();
    });
  }
  
  /**
   * Generate a URL for accessing a file
   * @param {string} storagePath - Path in storage
   * @param {string} storageType - Type of storage (defaults to configured type)
   * @param {number} expiresIn - Expiration time in seconds (for S3)
   * @returns {Promise<string>} - URL to access the file
   */
  async getFileUrl(storagePath, storageType = null, expiresIn = 3600) {
    const type = storageType || this.storageType;
    
    try {
      switch (type) {
        case 'local':
        case 'nfs':
          // For local/NFS, we'll serve through the Express server
          return `/api/preview/file/${encodeURIComponent(storagePath)}`;
        case 's3':
          // For S3, generate a pre-signed URL
          const s3Client = global.s3Client;
          const bucket = config.storage.s3.bucket;
          
          return await s3Client.getSignedUrlPromise('getObject', {
            Bucket: bucket,
            Key: storagePath,
            Expires: expiresIn
          });
        default:
          throw new Error(`Unsupported storage type: ${type}`);
      }
    } catch (error) {
      logger.error(`Error generating URL: ${error.message}`);
      throw error;
    }
  }
  
  /**
   * Delete a file from storage
   * @param {string} storagePath - Path in storage
   * @param {string} storageType - Type of storage (defaults to configured type)
   * @returns {Promise<boolean>} - Success status
   */
  async deleteFile(storagePath, storageType = null) {
    const type = storageType || this.storageType;
    
    try {
      switch (type) {
        case 'local':
          return await this.deleteFileLocal(storagePath);
        case 's3':
          return await this.deleteFileS3(storagePath);
        case 'nfs':
          return await this.deleteFileNFS(storagePath);
        default:
          throw new Error(`Unsupported storage type: ${type}`);
      }
    } catch (error) {
      logger.error(`Error deleting file: ${error.message}`);
      throw error;
    }
  }
  
  /**
   * Delete file from local filesystem
   * @param {string} storagePath - Relative path within local storage
   * @returns {Promise<boolean>} - Success status
   */
  async deleteFileLocal(storagePath) {
    const fullPath = path.join(config.storage.local.path, storagePath);
    
    // Wrap with retry
    const deleteFileWithRetry = withRetry(
      async () => {
        if (await fs.pathExists(fullPath)) {
          await fs.unlink(fullPath);
          return true;
        }
        return false;
      },
      this.retryConfig.maxRetries,
      this.retryConfig.baseDelay
    );
    
    return await deleteFileWithRetry();
  }
  
  /**
   * Delete file from S3-compatible storage
   * @param {string} storagePath - Key path within S3 bucket
   * @returns {Promise<boolean>} - Success status
   */
  async deleteFileS3(storagePath) {
    // Use global s3Client
    const s3Client = global.s3Client;
    const bucket = config.storage.s3.bucket;
    
    // Delete from S3 (AWS SDK handles retries internally)
    await s3Client.deleteObject({
      Bucket: bucket,
      Key: storagePath
    }).promise();
    
    return true;
  }
  
  /**
   * Delete file from NFS mount
   * @param {string} storagePath - Relative path within NFS mount
   * @returns {Promise<boolean>} - Success status
   */
  async deleteFileNFS(storagePath) {
    const fullPath = path.join(config.storage.nfs.mountPath, storagePath);
    
    // Use lock to prevent concurrent access issues on NFS
    return await lock.acquire('nfs-delete', async () => {
      // Wrap with retry
      const deleteFileWithRetry = withRetry(
        async () => {
          if (await fs.pathExists(fullPath)) {
            // Delete with timeout
            const deletePromise = fs.unlink(fullPath);
            
            // Add timeout to detect stalled NFS operations
            const timeoutPromise = new Promise((_, reject) => {
              setTimeout(() => {
                reject(new Error(`NFS delete operation timed out after ${config.storage.nfs.timeout}ms`));
              }, config.storage.nfs.timeout);
            });
            
            await Promise.race([deletePromise, timeoutPromise]);
            return true;
          }
          return false;
        },
        this.retryConfig.maxRetries,
        this.retryConfig.baseDelay
      );
      
      return await deleteFileWithRetry();
    });
  }
}

module.exports = new StorageService();

```

### `src/services/previewService.js`

```javascript
const path = require('path');
const fs = require('fs-extra');
const { v4: uuidv4 } = require('uuid');
const config = require('../../config');
const logger = require('../utils/logger');
const ffmpegUtils = require('../utils/ffmpeg');
const storageService = require('./storageService');
const Preview = require('../models/Preview');

/**
 * Preview Generation Service
 */
class PreviewService {
  /**
   * Generate previews for a video
   * @param {Object} options - Preview generation options
   * @returns {Promise<Array>} - Array of generated preview IDs
   */
  async generatePreviews(options) {
    const {
      sourceId,
      sourceType = 'segment',
      userId,
      inputPath,
      previewCount = config.preview.defaultCount,
      duration = config.preview.maxDuration,
      quality = 'medium',
      format = config.preview.defaultFormat
    } = options;
    
    if (!sourceId || !userId || !inputPath) {
      throw new Error('Missing required parameters');
    }
    
    try {
      // Create temporary directory for processing
      const tempDir = path.join(config.storage.tempPath, uuidv4());
      await fs.ensureDir(tempDir);
      
      try {
        // Get video duration
        const videoDuration = await ffmpegUtils.getVideoDuration(inputPath);
        logger.info(`Video duration: ${videoDuration}s`);
        
        // Calculate preview positions
        const previewPositions = this.calculatePreviewPositions(
          videoDuration, 
          previewCount, 
          duration
        );
        logger.info(`Generating ${previewPositions.length} previews`);
        
        // Generate each preview
        const previewPromises = previewPositions.map((position, index) => 
          this.generatePreview({
            sourceId,
            sourceType,
            userId,
            inputPath,
            startTime: position.start,
            duration,
            quality,
            format,
            position: position.position,
            tempDir,
            index
          })
        );
        
        // Wait for all previews to complete
        const results = await Promise.allSettled(previewPromises);
        
        // Filter successful previews
        const successfulPreviews = results
          .filter(result => result.status === 'fulfilled')
          .map(result => result.value);
          
        // Log failures
        results
          .filter(result => result.status === 'rejected')
          .forEach(result => logger.error(`Preview generation failed: ${result.reason}`));
        
        logger.info(`Generated ${successfulPreviews.length} previews successfully`);
        return successfulPreviews;
      } finally {
        // Clean up temp directory
        await fs.remove(tempDir);
      }
    } catch (error) {
      logger.error(`Preview generation error: ${error.message}`);
      throw error;
    }
  }
  
  /**
   * Generate a single preview
   * @param {Object} options - Preview options
   * @returns {Promise<Object>} - Preview data
   */
  async generatePreview(options) {
    const {
      sourceId,
      sourceType,
      userId,
      inputPath,
      startTime,
      duration,
      quality,
      format,
      position,
      tempDir,
      index
    } = options;
    
    // Generate preview ID
    const previewId = uuidv4();
    logger.info(`Generating preview ${previewId} at position ${startTime}s`);
    
    try {
      // Create preview record in database
      const preview = new Preview({
        previewId,
        sourceId,
        sourceType,
        userId,
        startTime,
        duration,
        format,
        storageType: config.storage.type,
        storagePath: '',
        quality,
        position,
        status: 'processing'
      });
      
      await preview.save();
      
      // Define output paths
      const previewFilename = `${previewId}.${format}`;
      const thumbnailFilename = `${previewId}_thumb.jpg`;
      const tempPreviewPath = path.join(tempDir, previewFilename);
      const tempThumbnailPath = path.join(tempDir, thumbnailFilename);
      
      // Get quality settings
      const qualitySettings = config.preview.quality[quality];
      
      // Generate preview clip
      const previewResult = await ffmpegUtils.generatePreviewClip(
        inputPath,
        tempPreviewPath,
        startTime,
        duration,
        {
          ...qualitySettings,
          format
        }
      );
      
      // Generate thumbnail
      await ffmpegUtils.generateThumbnail(
        tempPreviewPath, 
        tempThumbnailPath,
        {
          timestamp: 1,
          width: 320,
          height: 180
        }
      );
      
      // Read generated files
      const previewBuffer = await fs.readFile(tempPreviewPath);
      const thumbnailBuffer = await fs.readFile(tempThumbnailPath);
      
      // Determine storage paths
      const previewsDir = 'previews';
      const storagePath = path.join(previewsDir, previewFilename);
      const thumbnailPath = path.join(previewsDir, thumbnailFilename);
      
      // Store preview file
      const previewStorage = await storageService.storeFile(
        previewBuffer,
        storagePath,
        `video/${format}`
      );
      
      // Store thumbnail
      const thumbnailStorage = await storageService.storeFile(
        thumbnailBuffer,
        thumbnailPath,
        'image/jpeg'
      );
      
      // Update preview record with storage info and metadata
      preview.storagePath = previewStorage.storagePath;
      preview.thumbnailPath = thumbnailStorage.storagePath;
      preview.size = previewStorage.size;
      preview.width = previewResult.width;
      preview.height = previewResult.height;
      preview.bitrate = previewResult.bitrate;
      preview.status = 'completed';
      
      await preview.save();
      
      logger.info(`Preview ${previewId} generated successfully`);
      return preview;
    } catch (error) {
      // Update preview record with error
      await Preview.findOneAndUpdate(
        { previewId },
        { 
          status: 'failed',
          errorMessage: error.message
        }
      );
      
      logger.error(`Preview generation error: ${error.message}`);
      throw error;
    }
  }
  
  /**
   * Calculate optimal positions for preview clips
   * @param {number} duration - Video duration in seconds
   * @param {number} count - Number of previews to generate
   * @param {number} clipDuration - Duration of each preview clip
   * @returns {Array} - Array of start positions
   */
  calculatePreviewPositions(duration, count, clipDuration) {
    const positions = [];
    
    // Handle case where video is shorter than desired total preview duration
    if (duration <= count * clipDuration) {
      // Just use the beginning for a single preview
      positions.push({ start: 0, position: 'beginning' });
      return positions;
    }
    
    // Always include the beginning
    positions.push({ start: 0, position: 'beginning' });
    
    // For videos longer than twice the clip duration, add middle positions
    if (count > 1 && duration > clipDuration * 2) {
      // Calculate interval for middle positions
      const middleInterval = (duration - (clipDuration * 2)) / (count - 1);
      
      for (let i = 1; i < count - 1; i++) {
        const start = Math.round(clipDuration + (middleInterval * i));
        positions.push({ start, position: 'middle' });
      }
      
      // Add end position, leave room for the full clip
      const endStart = Math.max(0, Math.floor(duration - clipDuration));
      positions.push({ start: endStart, position: 'end' });
    }
    
    return positions.slice(0, count);
  }
  
  /**
   * Get previews for a source
   * @param {string} sourceId - Source ID (stream, segment, etc.)
   * @param {string} sourceType - Source type
   * @returns {Promise<Array>} - Array of previews
   */
  async getPreviewsBySource(sourceId, sourceType = 'segment') {
    try {
      const previews = await Preview.find({
        sourceId,
        sourceType,
        status: 'completed'
      }).sort({ startTime: 1 });
      
      return previews;
    } catch (error) {
      logger.error(`Error getting previews: ${error.message}`);
      throw error;
    }
  }
  
  /**
   * Get a specific preview by ID
   * @param {string} previewId - Preview ID
   * @returns {Promise<Object>} - Preview data
   */
  async getPreviewById(previewId) {
    try {
      const preview = await Preview.findOne({ previewId });
      
      if (!preview) {
        throw new Error(`Preview not found: ${previewId}`);
      }
      
      return preview;
    } catch (error) {
      logger.error(`Error getting preview: ${error.message}`);
      throw error;
    }
  }
  
  /**
   * Get preview and thumbnail URLs
   * @param {string} previewId - Preview ID
   * @param {number} expiresIn - URL expiration in seconds
   * @returns {Promise<Object>} - URLs for the preview
   */
  async getPreviewUrls(previewId, expiresIn = 3600) {
    try {
      const preview = await this.getPreviewById(previewId);
      
      // Get URLs for preview and thumbnail
      const previewUrl = await storageService.getFileUrl(
        preview.storagePath,
        preview.storageType,
        expiresIn
      );
      
      let thumbnailUrl = null;
      if (preview.thumbnailPath) {
        thumbnailUrl = await storageService.getFileUrl(
          preview.thumbnailPath,
          preview.storageType,
          expiresIn
        );
      }
      
      return {
        previewId,
        previewUrl,
        thumbnailUrl,
        startTime: preview.startTime,
        duration: preview.duration,
        format: preview.format,
        width: preview.width,
        height: preview.height,
        position: preview.position,
        expiresIn
      };
    } catch (error) {
      logger.error(`Error getting preview URLs: ${error.message}`);
      throw error;
    }
  }
  
  /**
   * Delete a preview
   * @param {string} previewId - Preview ID
   * @returns {Promise<boolean>} - Success status
   */
  async deletePreview(previewId) {
    try {
      const preview = await this.getPreviewById(previewId);
      
      // Delete preview file
      await storageService.deleteFile(
        preview.storagePath,
        preview.storageType
      );
      
      // Delete thumbnail if exists
      if (preview.thumbnailPath) {
        await storageService.deleteFile(
          preview.thumbnailPath,
          preview.storageType
        );
      }
      
      // Delete preview record
      await Preview.deleteOne({ previewId });
      
      logger.info(`Preview ${previewId} deleted successfully`);
      return true;
    } catch (error) {
      logger.error(`Error deleting preview: ${error.message}`);
      throw error;
    }
  }
}

module.exports = new PreviewService();
```

### `src/controllers/previewController.js`

```javascript
const path = require('path');
const fs = require('fs-extra');
const multer = require('multer');
const { v4: uuidv4 } = require('uuid');
const config = require('../../config');
const logger = require('../utils/logger');
const previewService = require('../services/previewService');
const storageService = require('../services/storageService');

// Configure multer for file uploads
const upload = multer({
  storage: multer.diskStorage({
    destination: (req, file, cb) => {
      const uploadDir = path.join(config.storage.tempPath, 'uploads');
      fs.ensureDirSync(uploadDir);
      cb(null, uploadDir);
    },
    filename: (req, file, cb) => {
      const ext = path.extname(file.originalname);
      cb(null, `${uuidv4()}${ext}`);
    }
  }),
  limits: {
    fileSize: 500 * 1024 * 1024, // 500MB max file size
  }
});

/**
 * Generate previews from a source video
 * @param {Object} req - Express request
 * @param {Object} res - Express response
 */
async function generatePreviews(req, res) {
  const sourceUpload = upload.single('source');
  
  sourceUpload(req, req.res, async (err) => {
    if (err) {
      logger.error(`Upload error: ${err.message}`);
      return res.status(400).json({
        success: false,
        message: `Upload failed: ${err.message}`
      });
    }
    
    try {
      // Get file path of uploaded video
      const inputPath = req.file.path;
      
      // Get parameters from request
      const sourceId = req.body.sourceId;
      const sourceType = req.body.sourceType || 'segment';
      const userId = req.body.userId;
      const previewCount = parseInt(req.body.previewCount) || config.preview.defaultCount;
      const duration = Math.min(
        parseInt(req.body.duration) || config.preview.maxDuration,
        config.preview.maxDuration
      );
      const quality = req.body.quality || 'medium';
      const format = req.body.format || config.preview.defaultFormat;
      
      // Validate required parameters
      if (!sourceId || !userId) {
        throw new Error('Missing required parameters: sourceId, userId');
      }
      
      // Generate previews
      const previews = await previewService.generatePreviews({
        sourceId,
        sourceType,
        userId,
        inputPath,
        previewCount,
        duration,
        quality,
        format
      });
      
      // Return preview IDs
      res.json({
        success: true,
        message: `Generated ${previews.length} previews`,
        data: {
          previews: previews.map(preview => ({
            previewId: preview.previewId,
            sourceId: preview.sourceId,
            startTime: preview.startTime,
            duration: preview.duration,
            position: preview.position
          }))
        }
      });
    } catch (error) {
      logger.error(`Preview generation error: ${error.message}`);
      res.status(500).json({
        success: false,
        message: `Preview generation failed: ${error.message}`
      });
    } finally {
      // Clean up the uploaded file
      if (req.file && req.file.path) {
        fs.removeSync(req.file.path);
      }
    }
  });
}

/**
 * Generate previews from an existing source ID
 * @param {Object} req - Express request
 * @param {Object} res - Express response
 */
async function generatePreviewsFromSource(req, res) {
  try {
    const { sourceId } = req.params;
    
    // Get parameters from request
    const sourceType = req.body.sourceType || 'segment';
    const userId = req.body.userId;
    const previewCount = parseInt(req.body.previewCount) || config.preview.defaultCount;
    const duration = Math.min(
      parseInt(req.body.duration) || config.preview.maxDuration,
      config.preview.maxDuration
    );
    const quality = req.body.quality || 'medium';
    const format = req.body.format || config.preview.defaultFormat;
    
    // Validate required parameters
    if (!userId) {
      throw new Error('Missing required parameter: userId');
    }
    
    // Get source video path (implementation depends on your system)
    // This is a placeholder - you need to integrate with your storage service
    // to get the actual path to the source video
    const sourcePath = path.join(config.storage.tempPath, `${sourceId}.mp4`);
    
    // Generate previews
    const previews = await previewService.generatePreviews({
      sourceId,
      sourceType,
      userId,
      inputPath: sourcePath,
      previewCount,
      duration,
      quality,
      format
    });
    
    // Return preview IDs
    res.json({
      success: true,
      message: `Generated ${previews.length} previews`,
      data: {
        previews: previews.map(preview => ({
          previewId: preview.previewId,
          sourceId: preview.sourceId,
          startTime: preview.startTime,
          duration: preview.duration,
          position: preview.position
        }))
      }
    });
  } catch (error) {
    logger.error(`Preview generation error: ${error.message}`);
    res.status(500).json({
      success: false,
      message: `Preview generation failed: ${error.message}`
    });
  }
}

/**
 * Get previews for a source
 * @param {Object} req - Express request
 * @param {Object} res - Express response
 */
async function getPreviewsBySource(req, res) {
  try {
    const { sourceId } = req.params;
    const sourceType = req.query.sourceType || 'segment';
    
    // Get previews
    const previews = await previewService.getPreviewsBySource(sourceId, sourceType);
    
    // Return previews with URLs
    const previewsWithUrls = await Promise.all(
      previews.map(async (preview) => {
        const urls = await previewService.getPreviewUrls(preview.previewId);
        return {
          previewId: preview.previewId,
          sourceId: preview.sourceId,
          startTime: preview.startTime,
          duration: preview.duration,
          position: preview.position,
          width: preview.width,
          height: preview.height,
          format: preview.format,
          previewUrl: urls.previewUrl,
          thumbnailUrl: urls.thumbnailUrl
        };
      })
    );
    
    res.json({
      success: true,
      data: {
        previews: previewsWithUrls
      }
    });
  } catch (error) {
    logger.error(`Error getting previews: ${error.message}`);
    res.status(500).json({
      success: false,
      message: `Failed to get previews: ${error.message}`
    });
  }
}

/**
 * Get a specific preview
 * @param {Object} req - Express request
 * @param {Object} res - Express response
 */
async function getPreview(req, res) {
  try {
    const { previewId } = req.params;
    
    // Get preview with URLs
    const preview = await previewService.getPreviewById(previewId);
    const urls = await previewService.getPreviewUrls(previewId);
    
    res.json({
      success: true,
      data: {
        previewId: preview.previewId,
        sourceId: preview.sourceId,
        sourceType: preview.sourceType,
        startTime: preview.startTime,
        duration: preview.duration,
        position: preview.position,
        width: preview.width,
        height: preview.height,
        format: preview.format,
        previewUrl: urls.previewUrl,
        thumbnailUrl: urls.thumbnailUrl
      }
    });
  } catch (error) {
    logger.error(`Error getting preview: ${error.message}`);
    res.status(500).json({
      success: false,
      message: `Failed to get preview: ${error.message}`
    });
  }
}

/**
 * Stream a preview file
 * @param {Object} req - Express request
 * @param {Object} res - Express response
 */
async function streamPreview(req, res) {
  try {
    const { previewId } = req.params;
    
    // Get preview
    const preview = await previewService.getPreviewById(previewId);
    
    // Get preview content
    const fileBuffer = await storageService.getFile(
      preview.storagePath,
      preview.storageType
    );
    
    // Set content type
    res.setHeader('Content-Type', `video/${preview.format}`);
    res.setHeader('Content-Length', fileBuffer.length);
    
    // Send the file
    res.send(fileBuffer);
  } catch (error) {
    logger.error(`Error streaming preview: ${error.message}`);
    res.status(500).json({
      success: false,
      message: `Failed to stream preview: ${error.message}`
    });
  }
}

/**
 * Get a preview thumbnail
 * @param {Object} req - Express request
 * @param {Object} res - Express response
 */
async function getPreviewThumbnail(req, res) {
  try {
    const { previewId } = req.params;
    
    // Get preview
    const preview = await previewService.getPreviewById(previewId);
    
    if (!preview.thumbnailPath) {
      throw new Error('Thumbnail not available');
    }
    
    // Get thumbnail content
    const fileBuffer = await storageService.getFile(
      preview.thumbnailPath,
      preview.storageType
    );
    
    // Set content type
    res.setHeader('Content-Type', 'image/jpeg');
    res.setHeader('Content-Length', fileBuffer.length);
    
    // Send the file
    res.send(fileBuffer);
  } catch (error) {
    logger.error(`Error getting thumbnail: ${error.message}`);
    res.status(500).json({
      success: false,
      message: `Failed to get thumbnail: ${error.message}`
    });
  }
}

/**
 * Delete a preview
 * @param {Object} req - Express request
 * @param {Object} res - Express response
 */
async function deletePreview(req, res) {
  try {
    const { previewId } = req.params;
    
    // Delete preview
    await previewService.deletePreview(previewId);
    
    res.json({
      success: true,
      message: `Preview ${previewId} deleted successfully`
    });
  } catch (error) {
    logger.error(`Error deleting preview: ${error.message}`);
    res.status(500).json({
      success: false,
      message: `Failed to delete preview: ${error.message}`
    });
  }
}

/**
 * Serve a file directly from storage
 * @param {Object} req - Express request
 * @param {Object} res - Express response
 */
async function serveFile(req, res) {
  try {
    const { filePath } = req.params;
    
    // Get file content
    const fileBuffer = await storageService.getFile(filePath);
    
    // Determine content type based on extension
    const ext = path.extname(filePath).toLowerCase();
    let contentType = 'application/octet-stream';
    
    switch (ext) {
      case '.mp4':
        contentType = 'video/mp4';
        break;
      case '.webm':
        contentType = 'video/webm';
        break;
      case '.jpg':
      case '.jpeg':
        contentType = 'image/jpeg';
        break;
      case '.png':
        contentType = 'image/png';
        break;
    }
    
    // Set headers
    res.setHeader('Content-Type', contentType);
    res.setHeader('Content-Length', fileBuffer.length);
    
    // Send the file
    res.send(fileBuffer);
  } catch (error) {
    logger.error(`Error serving file: ${error.message}`);
    res.status(500).json({
      success: false,
      message: `Failed to serve file: ${error.message}`
    });
  }
}

module.exports = {
  generatePreviews,
  generatePreviewsFromSource,
  getPreviewsBySource,
  getPreview,
  streamPreview,
  getPreviewThumbnail,
  deletePreview,
  serveFile
};
```

### `src/routes/previewRoutes.js`

```javascript
const express = require('express');
const previewController = require('../controllers/previewController');
const router = express.Router();

// Generate previews from uploaded file
router.post('/generate', previewController.generatePreviews);

// Generate previews from existing source
router.post('/source/:sourceId/generate', previewController.generatePreviewsFromSource);

// Get previews for a source
router.get('/source/:sourceId', previewController.getPreviewsBySource);

// Get a specific preview
router.get('/:previewId', previewController.getPreview);

// Stream a preview
router.get('/:previewId/stream', previewController.streamPreview);

// Get a preview thumbnail
router.get('/:previewId/thumbnail', previewController.getPreviewThumbnail);

// Delete a preview
router.delete('/:previewId', previewController.deletePreview);

// Serve a file directly
router.get('/file/:filePath(*)', previewController.serveFile);

module.exports = router;
```

## 4. Main Application Entry Point

### `server.js`

```javascript
const express = require('express');
const cors = require('cors');
const morgan = require('morgan');
const config = require('./config');
const { connectDB } = require('./config/database');
const { initializeStorage } = require('./config/storage');
const logger = require('./src/utils/logger');
const previewRoutes = require('./src/routes/previewRoutes');

// Initialize Express app
const app = express();

// Apply middleware
app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(morgan('dev'));

// API routes
app.use(`${config.server.apiPrefix}/preview`, previewRoutes);

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    service: 'preview-generation-service',
    timestamp: new Date().toISOString()
  });
});

// Error handling middleware
app.use((err, req, res, next) => {
  logger.error(`Error: ${err.message}`);
  res.status(500).json({
    success: false,
    message: err.message
  });
});

// Initialize and start the server
async function startServer() {
  try {
    // Connect to MongoDB
    await connectDB();
    
    // Initialize storage
    await initializeStorage();
    
    // Start server
    const PORT = config.server.port;
    app.listen(PORT, () => {
      logger.info(`Preview Generation Service running on port ${PORT}`);
      logger.info(`Storage type: ${config.storage.type}`);
      logger.info(`Environment: ${config.server.env}`);
    });
  } catch (error) {
    logger.error(`Failed to start server: ${error.message}`);
    process.exit(1);
  }
}

// Start the server
startServer();
```

## 5. Package.json Scripts

Update your `package.json` with the following scripts:

```json
"scripts": {
  "start": "node server.js",
  "dev": "nodemon server.js",
  "test": "jest",
  "lint": "eslint ."
}
```

## 6. Running the Service Locally

Now you're ready to run the service:

```bash
# Install dependencies
npm install

# Start MongoDB (if running locally)
# In a separate terminal, run:
mongod --dbpath=/data/db  # adjust path as needed

# Start the service in development mode
npm run dev
```

The service will be available at http://localhost:3001 with the API prefix `/api/preview`.

## 7. Testing the Service

### Using curl

You can test the service using curl:

```bash
# Upload a video and generate previews
curl -X POST \
  http://localhost:3001/api/preview/generate \
  -H 'Content-Type: multipart/form-data' \
  -F 'source=@/path/to/your/video.mp4' \
  -F 'sourceId=test-video-123' \
  -F 'userId=test-user-123' \
  -F 'previewCount=3' \
  -F 'duration=5' \
  -F 'quality=medium'

# Get previews for a source
curl http://localhost:3001/api/preview/source/test-video-123

# Get a specific preview
curl http://localhost:3001/api/preview/[previewId]
```

### Using Postman

1. Create a new collection for testing
2. Add a POST request to `http://localhost:3001/api/preview/generate`
3. In the "Body" tab, select "form-data"
4. Add the following key-value pairs:
   - source: Select "File" and choose a video file
   - sourceId: test-video-123
   - userId: test-user-123
   - previewCount: 3
   - duration: 5
   - quality: medium
5. Send the request

### Example Test Video Sources

If you don't have a video for testing, you can download sample videos from:
- https://sample-videos.com/
- https://test-videos.co.uk/

### Automated Testing

Create a simple test in `tests/preview.test.js`:

```javascript
const fs = require('fs');
const path = require('path');
const axios = require('axios');
const FormData = require('form-data');

const API_URL = 'http://localhost:3001/api/preview';

// This test assumes the service is already running
describe('Preview Generation Service', () => {
  let testPreviewId;

  test('Generate previews from a video file', async () => {
    // Create form data with a test video
    const form = new FormData();
    form.append('source', fs.createReadStream(path.join(__dirname, 'test-video.mp4')));
    form.append('sourceId', 'test-video-' + Date.now());
    form.append('userId', 'test-user-123');
    form.append('previewCount', '2');
    form.append('duration', '3');
    form.append('quality', 'low');

    // Send request
    const response = await axios.post(`${API_URL}/generate`, form, {
      headers: form.getHeaders()
    });

    // Check response
    expect(response.status).toBe(200);
    expect(response.data.success).toBe(true);
    expect(response.data.data.previews.length).toBeGreaterThan(0);

    // Store a preview ID for later tests
    testPreviewId = response.data.data.previews[0].previewId;
  }, 30000); // 30 second timeout for video processing

  test('Get a specific preview', async () => {
    // Skip if no preview ID from previous test
    if (!testPreviewId) {
      console.warn('Skipping test because no preview ID is available');
      return;
    }

    const response = await axios.get(`${API_URL}/${testPreviewId}`);

    // Check response
    expect(response.status).toBe(200);
    expect(response.data.success).toBe(true);
    expect(response.data.data.previewId).toBe(testPreviewId);
    expect(response.data.data.previewUrl).toBeDefined();
  });
});
```

To run the test, you'll need to:
1. Put a small test video file in the `tests` directory named `test-video.mp4`
2. Install test dependencies: `npm install jest axios form-data --save-dev`
3. Run: `npm test`

## 8. Integration with Mobile Apps

### iOS Integration Example

To integrate with your iOS app, you'll call the API to get previews:

```swift
func fetchPreviews(for sourceId: String, completion: @escaping ([PreviewModel]?, Error?) -> Void) {
    let url = URL(string: "http://your-server.com:3001/api/preview/source/\(sourceId)")!
    
    URLSession.shared.dataTask(with: url) { data, response, error in
        guard let data = data, error == nil else {
            completion(nil, error)
            return
        }
        
        do {
            let result = try JSONDecoder().decode(PreviewsResponse.self, from: data)
            completion(result.data.previews, nil)
        } catch {
            completion(nil, error)
        }
    }.resume()
}

struct PreviewsResponse: Codable {
    let success: Bool
    let data: PreviewsData
}

struct PreviewsData: Codable {
    let previews: [PreviewModel]
}

struct PreviewModel: Codable {
    let previewId: String
    let sourceId: String
    let startTime: Double
    let duration: Double
    let position: String
    let previewUrl: String
    let thumbnailUrl: String?
}
```

### Android Integration Example

```java
private void fetchPreviews(String sourceId) {
    String url = "http://your-server.com:3001/api/preview/source/" + sourceId;
    
    JsonObjectRequest request = new JsonObjectRequest(
        Request.Method.GET, 
        url, 
        null,
        response -> {
            try {
                boolean success = response.getBoolean("success");
                if (success) {
                    JSONObject data = response.getJSONObject("data");
                    JSONArray previews = data.getJSONArray("previews");
                    
                    List<PreviewModel> previewList = new ArrayList<>();
                    for (int i = 0; i < previews.length(); i++) {
                        JSONObject preview = previews.getJSONObject(i);
                        previewList.add(new PreviewModel(
                            preview.getString("previewId"),
                            preview.getString("sourceId"),
                            preview.getDouble("startTime"),
                            preview.getDouble("duration"),
                            preview.getString("position"),
                            preview.getString("previewUrl"),
                            preview.has("thumbnailUrl") ? preview.getString("thumbnailUrl") : null
                        ));
                    }
                    
                    // Use the previews
                    displayPreviews(previewList);
                }
            } catch (JSONException e) {
                Log.e(TAG, "JSON parsing error", e);
            }
        },
        error -> {
            Log.e(TAG, "Network error", error);
        }
    );
    
    // Add the request to your RequestQueue
    requestQueue.add(request);
}
```

## 9. Troubleshooting

### Common Issues and Solutions

1. **MongoDB Connection Issues**
   - Check that MongoDB is running
   - Verify connection string in `.env`
   - Ensure network access is permitted

2. **FFmpeg Problems**
   - Make sure `ffmpeg-static` is installed
   - If using a custom FFmpeg path, verify it's correct
   - Check that you have execute permissions

3. **Storage Permission Issues**
   - Ensure the service has write access to storage directories
   - For NFS, check that mount is accessible
   - For S3, verify credentials and bucket permissions

4. **Large File Upload Issues**
   - Check multer file size limits (currently set to 500MB)
   - Verify Node.js memory limits
   - Consider adjusting the Express body parser limits

## 10. Production Considerations

For a production deployment, consider the following:

1. **Use environment variables** for all configuration
2. **Set up proper logging** with rotation
3. **Add rate limiting** to prevent abuse
4. **Implement authentication** for API endpoints
5. **Configure HTTPS** for secure communication
6. **Set up monitoring** for service health
7. **Add CI/CD pipeline** for automated testing and deployment
8. **Consider containerization** with Docker for easier deployment

This completes the setup for your Preview Generation Service. You now have a fully functional service that can generate optimized video previews for auto-playing in your mobile apps, with support for multiple storage options including S3 and NFS.
