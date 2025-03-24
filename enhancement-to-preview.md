/**
 * Preview Generation Service Implementation
 * This implementation includes all enhancements from the enhancement guide
 */

// ===== models/Preview.js =====
const mongoose = require('mongoose');

const previewSchema = new mongoose.Schema({
  sourceId: { type: String, required: true },
  sourceType: { type: String, required: true },
  userId: { type: String, required: true },
  position: { type: Number, required: true },
  storagePath: { type: String, required: true },
  storageType: { type: String, required: true },
  thumbnailPath: { type: String, required: true },
  format: { type: String, required: true },
  createdAt: { type: Date, default: Date.now },
  userFolder: { type: String, required: true }, // ✅ Enhancement #11: Add userFolder field
});

const Preview = mongoose.model('Preview', previewSchema);
module.exports = Preview;

// ===== routes/previewRoutes.js =====
const express = require('express');
const router = express.Router();
const previewController = require('../controllers/previewController');
const multer = require('multer');
const upload = multer({ dest: 'uploads/' });

// ✅ Enhancement #1: Secure File URL Exposure Fix
// Removed insecure route: router.get('/file/:filePath(*)', previewController.getFile);
// Add secure download route:
router.get('/:previewId/download', previewController.downloadPreview);

// Other routes
router.post('/generate', upload.single('video'), previewController.generatePreviews);
router.get('/:id', previewController.getPreview);
router.get('/source/:sourceId/:sourceType', previewController.getPreviewsBySource);

module.exports = router;

// ===== controllers/previewController.js =====
const path = require('path');
const fs = require('fs-extra');
const previewService = require('../services/previewService');
const storageService = require('../services/storageService');
const logger = require('../utils/logger');
const mime = require('mime-types'); // ✅ Enhancement #7: Use mime-types

async function generatePreviews(req, res) {
  try {
    if (!req.file) {
      return res.status(400).json({ success: false, message: 'No file uploaded' });
    }

    // ✅ Enhancement #8: Input Format Validation
    const allowedFormats = ['.mp4', '.webm'];
    const inputExt = path.extname(req.file.path).toLowerCase();

    if (!allowedFormats.includes(inputExt)) {
      fs.removeSync(req.file.path);
      return res.status(400).json({
        success: false,
        message: `Unsupported file format: ${inputExt}`
      });
    }

    const { sourceId, sourceType, userId, quality, format, duration, count } = req.body;

    const result = await previewService.generatePreviews({
      sourceId,
      sourceType,
      userId,
      inputPath: req.file.path,
      quality: quality || 'medium',
      format: format || 'mp4',
      duration: parseInt(duration) || 3,
      count: parseInt(count) || 3
    });

    res.json({ success: true, previews: result });
  } catch (error) {
    logger.error(`Error generating previews: ${error.message}`);
    res.status(500).json({ success: false, message: error.message });
  } finally {
    // Clean up uploaded file
    if (req.file && req.file.path) {
      fs.removeSync(req.file.path);
    }
  }
}

// ✅ Enhancement #1: Secure File URL Exposure Fix
async function downloadPreview(req, res) {
  try {
    const { previewId } = req.params;
    const preview = await previewService.getPreviewById(previewId);
    
    if (!preview) {
      return res.status(404).json({ success: false, message: 'Preview not found' });
    }
    
    const fileBuffer = await storageService.getFile(preview.storagePath, preview.storageType);
    const ext = path.extname(preview.storagePath).toLowerCase();
    
    // ✅ Enhancement #7: Use mime-types for Content-Type
    const contentType = mime.lookup(ext) || 'application/octet-stream';
    res.setHeader('Content-Type', contentType);
    res.setHeader('Content-Length', fileBuffer.length);
    res.send(fileBuffer);
  } catch (error) {
    logger.error(`Error downloading preview: ${error.message}`);
    res.status(500).json({ success: false, message: error.message });
  }
}

async function getPreview(req, res) {
  try {
    const preview = await previewService.getPreviewById(req.params.id);
    if (!preview) {
      return res.status(404).json({ success: false, message: 'Preview not found' });
    }
    res.json({ success: true, preview });
  } catch (error) {
    logger.error(`Error getting preview: ${error.message}`);
    res.status(500).json({ success: false, message: error.message });
  }
}

async function getPreviewsBySource(req, res) {
  try {
    const { sourceId, sourceType } = req.params;
    const previews = await previewService.getPreviewsBySource(sourceId, sourceType);
    res.json({ success: true, previews });
  } catch (error) {
    logger.error(`Error getting previews by source: ${error.message}`);
    res.status(500).json({ success: false, message: error.message });
  }
}

module.exports = {
  generatePreviews,
  downloadPreview, // ✅ Enhancement #1: Added secure download method
  getPreview,
  getPreviewsBySource
};

// ===== services/previewService.js =====
const fs = require('fs-extra');
const path = require('path');
const { v4: uuidv4 } = require('uuid');
const ffmpeg = require('fluent-ffmpeg');
const Preview = require('../models/Preview');
const storageService = require('./storageService');
const logger = require('../utils/logger');
const limit = require('p-limit'); // ✅ Enhancement #5: Concurrency Control

// ✅ Enhancement #5: Concurrency Control on Preview Generation
const concurrencyLimit = 3;
const limiter = limit(concurrencyLimit);

async function generatePreviews({ sourceId, sourceType, userId, inputPath, quality, format, duration, count }) {
  const tempDir = path.join('temp', uuidv4());
  await fs.ensureDir(tempDir);

  try {
    // Get video duration
    const videoDuration = await getVideoDuration(inputPath);
    
    // Calculate preview positions
    const previewPositions = calculatePreviewPositions(videoDuration, count);
    
    // ✅ Enhancement #5: Concurrency Control with p-limit
    const previewPromises = previewPositions.map((position, index) =>
      limiter(() => generatePreview({
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
      }))
    );

    const results = await Promise.allSettled(previewPromises);
    
    // Filter successful previews
    const previews = results
      .filter(result => result.status === 'fulfilled')
      .map(result => result.value);
    
    return previews;
  } catch (error) {
    logger.error(`Error generating previews: ${error.message}`);
    throw error;
  } finally {
    // Clean up temp directory
    fs.removeSync(tempDir);
  }
}

async function generatePreview({ sourceId, sourceType, userId, inputPath, startTime, duration, quality, format, position, tempDir, index }) {
  try {
    // ✅ Enhancement #10: Enforce User Folder Structure in All Storage Backends
    // Changed from 'previews' to user-specific folder
    const previewsDir = `${userId}/previews`;
    
    const previewFilename = `${uuidv4()}.${format}`;
    const thumbnailFilename = `${uuidv4()}_thumb.jpg`;
    
    const previewTempPath = path.join(tempDir, previewFilename);
    const thumbnailTempPath = path.join(tempDir, thumbnailFilename);
    
    // Generate video preview
    await generateVideoPreview(inputPath, previewTempPath, startTime, duration, quality, format);
    
    // Generate thumbnail
    await generateThumbnail(inputPath, thumbnailTempPath, startTime);
    
    // Read files to buffer
    const previewBuffer = await fs.readFile(previewTempPath);
    const thumbnailBuffer = await fs.readFile(thumbnailTempPath);
    
    // ✅ Enhancement #2: S3 Folder Structure per User
    // Store files with user-specific paths
    const storagePath = path.join(previewsDir, previewFilename);
    const thumbnailPath = path.join(previewsDir, thumbnailFilename);
    
    const previewStorage = await storageService.storeFile(
      previewBuffer,
      storagePath,
      `video/${format}`
    );
    
    const thumbnailStorage = await storageService.storeFile(
      thumbnailBuffer,
      thumbnailPath,
      'image/jpeg'
    );
    
    // Create preview document
    const preview = new Preview({
      sourceId,
      sourceType,
      userId,
      position,
      storagePath: previewStorage.path,
      storageType: previewStorage.type,
      thumbnailPath: thumbnailStorage.path,
      format,
      userFolder: `${userId}/previews` // ✅ Enhancement #11: Add userFolder field
    });
    
    await preview.save();
    return preview;
  } catch (error) {
    logger.error(`Error generating preview: ${error.message}`);
    throw error;
  }
}

async function getVideoDuration(filepath) {
  return new Promise((resolve, reject) => {
    ffmpeg.ffprobe(filepath, (err, metadata) => {
      if (err) return reject(err);
      resolve(metadata.format.duration);
    });
  });
}

function calculatePreviewPositions(duration, count) {
  const positions = [];
  const interval = duration / (count + 1);
  
  for (let i = 1; i <= count; i++) {
    positions.push({
      position: i,
      start: interval * i
    });
  }
  
  return positions;
}

async function generateVideoPreview(input, output, startTime, duration, quality, format) {
  const qualitySettings = {
    low: { videoBitrate: '500k', audioBitrate: '64k' },
    medium: { videoBitrate: '1000k', audioBitrate: '128k' },
    high: { videoBitrate: '2000k', audioBitrate: '256k' }
  };
  
  const settings = qualitySettings[quality] || qualitySettings.medium;
  
  return new Promise((resolve, reject) => {
    ffmpeg(input)
      .setStartTime(startTime)
      .duration(duration)
      .outputOptions([
        `-b:v ${settings.videoBitrate}`,
        `-b:a ${settings.audioBitrate}`
      ])
      .output(output)
      .on('end', resolve)
      .on('error', reject)
      .run();
  });
}

async function generateThumbnail(input, output, startTime) {
  return new Promise((resolve, reject) => {
    ffmpeg(input)
      .setStartTime(startTime)
      .outputOptions(['-vframes 1'])
      .output(output)
      .on('end', resolve)
      .on('error', reject)
      .run();
  });
}

async function getPreviewById(id) {
  return Preview.findById(id);
}

async function getPreviewsBySource(sourceId, sourceType) {
  return Preview.find({ sourceId, sourceType }).sort('position');
}

module.exports = {
  generatePreviews,
  generatePreview,
  getPreviewById,
  getPreviewsBySource
};

// ===== services/storageService.js =====
const fs = require('fs-extra');
const path = require('path');
const AWS = require('aws-sdk');
const config = require('../config');
const logger = require('../utils/logger');
const AsyncLock = require('async-lock');

// ✅ Enhancement #6: Granular Locking on NFS
const lock = new AsyncLock();

class StorageService {
  constructor() {
    this.setupS3();
    this.setupLocalStorage();
    this.setupNFSStorage();
  }
  
  setupS3() {
    try {
      this.s3 = new AWS.S3({
        accessKeyId: config.s3.accessKey,
        secretAccessKey: config.s3.secretKey,
        endpoint: config.s3.endpoint,
        s3ForcePathStyle: true
      });
      
      // ✅ Enhancement #4: Disable S3 Auto Bucket Creation in Production
      this.s3.headBucket({ Bucket: config.s3.bucket }, (error) => {
        if (error) {
          if ((error.code === 'NotFound' || error.code === 'NoSuchBucket') && 
              process.env.ALLOW_S3_BUCKET_CREATE === 'true') {
            this.s3.createBucket({ Bucket: config.s3.bucket }, (err) => {
              if (err) {
                logger.error(`Failed to create S3 bucket: ${err.message}`);
              } else {
                logger.info(`Created S3 bucket: ${config.s3.bucket}`);
              }
            });
          } else {
            logger.error(`S3 initialization error: ${error.message}`);
          }
        }
      });
    } catch (error) {
      logger.error(`S3 setup error: ${error.message}`);
    }
  }
  
  setupLocalStorage() {
    try {
      this.localStoragePath = config.storage.localPath;
      fs.ensureDirSync(this.localStoragePath);
    } catch (error) {
      logger.error(`Local storage setup error: ${error.message}`);
    }
  }
  
  setupNFSStorage() {
    try {
      this.nfsStoragePath = config.storage.nfsPath;
      fs.ensureDirSync(this.nfsStoragePath);
    } catch (error) {
      logger.error(`NFS storage setup error: ${error.message}`);
    }
  }
  
  async storeFile(buffer, filePath, contentType) {
    try {
      const storageType = config.storage.preferredType;
      
      switch (storageType) {
        case 's3':
          return await this.storeFileS3(buffer, filePath, contentType);
        case 'nfs':
          return await this.storeFileNFS(buffer, filePath);
        case 'local':
        default:
          return await this.storeFileLocal(buffer, filePath);
      }
    } catch (error) {
      logger.error(`Error storing file: ${error.message}`);
      throw error;
    }
  }
  
  async storeFileS3(buffer, filePath, contentType) {
    try {
      const params = {
        Bucket: config.s3.bucket,
        Key: filePath, // ✅ Enhancement #2: Use S3 key as-is
        Body: buffer,
        ContentType: contentType
      };
      
      await this.s3.putObject(params).promise();
      
      return {
        type: 's3',
        path: filePath
      };
    } catch (error) {
      logger.error(`S3 storage error: ${error.message}`);
      throw error;
    }
  }
  
  // ✅ Enhancement #6: Granular Locking on NFS
  async storeFileNFS(buffer, filePath) {
    try {
      const fullPath = path.join(this.nfsStoragePath, filePath);
      await fs.ensureDir(path.dirname(fullPath));
      
      // Use granular lock with specific key for the file
      return await lock.acquire(`nfs-write:${filePath}`, async () => {
        await fs.writeFile(fullPath, buffer);
        return {
          type: 'nfs',
          path: filePath
        };
      });
    } catch (error) {
      logger.error(`NFS storage error: ${error.message}`);
      throw error;
    }
  }
  
  async storeFileLocal(buffer, filePath) {
    try {
      const fullPath = path.join(this.localStoragePath, filePath);
      await fs.ensureDir(path.dirname(fullPath));
      await fs.writeFile(fullPath, buffer);
      
      return {
        type: 'local',
        path: filePath
      };
    } catch (error) {
      logger.error(`Local storage error: ${error.message}`);
      throw error;
    }
  }
  
  async getFile(filePath, storageType) {
    try {
      switch (storageType) {
        case 's3':
          return await this.getFileS3(filePath);
        case 'nfs':
          return await this.getFileNFS(filePath);
        case 'local':
        default:
          return await this.getFileLocal(filePath);
      }
    } catch (error) {
      logger.error(`Error getting file: ${error.message}`);
      throw error;
    }
  }
  
  async getFileS3(filePath) {
    try {
      const params = {
        Bucket: config.s3.bucket,
        Key: filePath
      };
      
      const data = await this.s3.getObject(params).promise();
      return data.Body;
    } catch (error) {
      logger.error(`S3 retrieval error: ${error.message}`);
      throw error;
    }
  }
  
  // ✅ Enhancement #6: Granular Locking on NFS
  async getFileNFS(filePath) {
    try {
      const fullPath = path.join(this.nfsStoragePath, filePath);
      
      // Use granular lock with specific key for the file
      return await lock.acquire(`nfs-read:${filePath}`, async () => {
        return await fs.readFile(fullPath);
      });
    } catch (error) {
      logger.error(`NFS retrieval error: ${error.message}`);
      throw error;
    }
  }
  
  async getFileLocal(filePath) {
    try {
      const fullPath = path.join(this.localStoragePath, filePath);
      return await fs.readFile(fullPath);
    } catch (error) {
      logger.error(`Local retrieval error: ${error.message}`);
      throw error;
    }
  }
  
  // ✅ Enhancement #6: Granular Locking on NFS
  async deleteFile(filePath, storageType) {
    try {
      switch (storageType) {
        case 's3':
          return await this.deleteFileS3(filePath);
        case 'nfs':
          return await this.deleteFileNFS(filePath);
        case 'local':
        default:
          return await this.deleteFileLocal(filePath);
      }
    } catch (error) {
      logger.error(`Error deleting file: ${error.message}`);
      throw error;
    }
  }
  
  async deleteFileS3(filePath) {
    try {
      const params = {
        Bucket: config.s3.bucket,
        Key: filePath
      };
      
      await this.s3.deleteObject(params).promise();
      return true;
    } catch (error) {
      logger.error(`S3 deletion error: ${error.message}`);
      throw error;
    }
  }
  
  // ✅ Enhancement #6: Granular Locking on NFS
  async deleteFileNFS(filePath) {
    try {
      const fullPath = path.join(this.nfsStoragePath, filePath);
      
      // Use granular lock with specific key for the file
      return await lock.acquire(`nfs-delete:${filePath}`, async () => {
        await fs.remove(fullPath);
        return true;
      });
    } catch (error) {
      logger.error(`NFS deletion error: ${error.message}`);
      throw error;
    }
  }
  
  async deleteFileLocal(filePath) {
    try {
      const fullPath = path.join(this.localStoragePath, filePath);
      await fs.remove(fullPath);
      return true;
    } catch (error) {
      logger.error(`Local deletion error: ${error.message}`);
      throw error;
    }
  }
}

module.exports = new StorageService();

// ===== database.js =====
const mongoose = require('mongoose');
const config = require('./config');
const logger = require('./utils/logger');

// ✅ Enhancement #3: MongoDB Retry on Startup
async function connectWithRetry(retries = 5, delay = 3000) {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      await mongoose.connect(config.mongodb.uri, {
        useNewUrlParser: true,
        useUnifiedTopology: true
      });
      logger.info(`MongoDB connected: ${mongoose.connection.host}`);
      return;
    } catch (error) {
      logger.error(`MongoDB connection failed (Attempt ${attempt}): ${error.message}`);
      if (attempt === retries) process.exit(1);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

module.exports = { connectDB: connectWithRetry };

// ===== .env =====
// Add this to your .env file:
// ✅ Enhancement #4: Disable S3 Auto Bucket Creation in Production
// ALLOW_S3_BUCKET_CREATE=false

// ===== app.js =====
const express = require('express');
const path = require('path');
const cors = require('cors');
const { connectDB } = require('./database');
const previewRoutes = require('./routes/previewRoutes');
const logger = require('./utils/logger');

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Routes
app.use('/api/preview', previewRoutes);

// Start server
async function startServer() {
  try {
    // ✅ Enhancement #3: MongoDB Retry on Startup
    await connectDB();
    
    app.listen(PORT, () => {
      logger.info(`Server started on port ${PORT}`);
    });
  } catch (error) {
    logger.error(`Server startup failed: ${error.message}`);
    process.exit(1);
  }
}

startServer();
