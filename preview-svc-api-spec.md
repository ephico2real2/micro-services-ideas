# Preview Generation Service API Specification

This document outlines the API endpoints available in the Preview Generation Service and how to consume them.

## Base URL

The base URL follows this pattern:
```
http://{SERVICE_HOST}:{SERVICE_PORT}/api
```

Where:
- `SERVICE_HOST`: Hostname or IP address of the service (default: localhost)
- `SERVICE_PORT`: Port number the service is running on (default: 3001)

**Example:**
```
http://localhost:3001/api
```

For production environments, you might use a domain name:
```
https://preview-service.example.com/api
```

## Authentication

The current implementation does not include authentication. For production use, implement JWT token or API key authentication.

## API Endpoints

### Generate Previews

Generates preview clips from an uploaded video file.

**Endpoint:** `POST /preview/generate`

**Content-Type:** `multipart/form-data`

**Request Parameters:**

| Field         | Type   | Required | Description                                             |
|---------------|--------|----------|---------------------------------------------------------|
| source        | File   | Yes      | Video file to generate previews from (.mp4, .webm only) |
| sourceId      | String | Yes      | Unique identifier for the source video                  |
| userId        | String | Yes      | User identifier owning the previews                     |
| previewCount  | Number | No       | Number of previews to generate (default: 3)             |
| duration      | Number | No       | Duration of each preview in seconds (default: 5)        |
| quality       | String | No       | Quality of previews: 'low', 'medium', 'high' (default: 'medium') |
| format        | String | No       | Output format: 'mp4', 'webm' (default: 'mp4')           |

**Example Request:**

```bash
curl -X POST \
  http://localhost:3001/api/preview/generate \
  -H 'Content-Type: multipart/form-data' \
  -F 'source=@/path/to/video.mp4' \
  -F 'sourceId=video-123' \
  -F 'userId=user-456' \
  -F 'previewCount=3' \
  -F 'duration=5' \
  -F 'quality=medium'
```

**Response:**

```json
{
  "success": true,
  "message": "Generated 3 previews",
  "data": {
    "previews": [
      {
        "previewId": "550e8400-e29b-41d4-a716-446655440000",
        "sourceId": "video-123",
        "startTime": 0,
        "duration": 5,
        "position": "beginning",
        "userFolder": "user-456/previews"
      },
      {
        "previewId": "550e8400-e29b-41d4-a716-446655440001",
        "sourceId": "video-123",
        "startTime": 30,
        "duration": 5,
        "position": "middle",
        "userFolder": "user-456/previews"
      },
      {
        "previewId": "550e8400-e29b-41d4-a716-446655440002",
        "sourceId": "video-123",
        "startTime": 55,
        "duration": 5,
        "position": "end",
        "userFolder": "user-456/previews"
      }
    ]
  }
}
```

### Generate Previews from Existing Source

Generates previews from a source that's already available to the service.

**Endpoint:** `POST /preview/source/:sourceId/generate`

**Content-Type:** `application/json`

**URL Parameters:**

| Parameter | Type   | Required | Description                     |
|-----------|--------|----------|---------------------------------|
| sourceId  | String | Yes      | Identifier of the source video  |

**Request Body:**

```json
{
  "userId": "user-456",
  "sourceType": "segment",
  "previewCount": 3,
  "duration": 5,
  "quality": "medium",
  "format": "mp4"
}
```

**Example Request:**

```bash
curl -X POST \
  http://localhost:3001/api/preview/source/video-123/generate \
  -H 'Content-Type: application/json' \
  -d '{
    "userId": "user-456",
    "previewCount": 3,
    "duration": 5,
    "quality": "medium"
  }'
```

**Response:** Same as `/preview/generate`

### Get Previews by Source

Retrieves all previews associated with a source.

**Endpoint:** `GET /preview/source/:sourceId`

**URL Parameters:**

| Parameter | Type   | Required | Description                    |
|-----------|--------|----------|--------------------------------|
| sourceId  | String | Yes      | Identifier of the source video |

**Query Parameters:**

| Parameter  | Type   | Required | Description                                     |
|------------|--------|----------|-------------------------------------------------|
| sourceType | String | No       | Type of source: 'segment', 'stream', 'video' (default: 'segment') |

**Example Request:**

```bash
curl -X GET http://localhost:3001/api/preview/source/video-123
```

**Response:**

```json
{
  "success": true,
  "data": {
    "previews": [
      {
        "previewId": "550e8400-e29b-41d4-a716-446655440000",
        "sourceId": "video-123",
        "startTime": 0,
        "duration": 5,
        "position": "beginning",
        "width": 640,
        "height": 360,
        "format": "mp4",
        "userFolder": "user-456/previews",
        "previewUrl": "http://localhost:3001/api/preview/550e8400-e29b-41d4-a716-446655440000/stream",
        "thumbnailUrl": "http://localhost:3001/api/preview/550e8400-e29b-41d4-a716-446655440000/thumbnail"
      },
      {
        "previewId": "550e8400-e29b-41d4-a716-446655440001",
        "sourceId": "video-123",
        "startTime": 30,
        "duration": 5,
        "position": "middle",
        "width": 640,
        "height": 360,
        "format": "mp4",
        "userFolder": "user-456/previews",
        "previewUrl": "http://localhost:3001/api/preview/550e8400-e29b-41d4-a716-446655440001/stream",
        "thumbnailUrl": "http://localhost:3001/api/preview/550e8400-e29b-41d4-a716-446655440001/thumbnail"
      }
    ]
  }
}
```

### Get Preview Details

Retrieves details for a specific preview.

**Endpoint:** `GET /preview/:previewId`

**URL Parameters:**

| Parameter | Type   | Required | Description              |
|-----------|--------|----------|--------------------------|
| previewId | String | Yes      | Identifier of the preview |

**Example Request:**

```bash
curl -X GET http://localhost:3001/api/preview/550e8400-e29b-41d4-a716-446655440000
```

**Response:**

```json
{
  "success": true,
  "data": {
    "previewId": "550e8400-e29b-41d4-a716-446655440000",
    "sourceId": "video-123",
    "sourceType": "segment",
    "startTime": 0,
    "duration": 5,
    "position": "beginning",
    "width": 640,
    "height": 360,
    "format": "mp4",
    "userFolder": "user-456/previews",
    "previewUrl": "http://localhost:3001/api/preview/550e8400-e29b-41d4-a716-446655440000/stream",
    "thumbnailUrl": "http://localhost:3001/api/preview/550e8400-e29b-41d4-a716-446655440000/thumbnail"
  }
}
```

### Stream Preview Video

Streams a preview video file.

**Endpoint:** `GET /preview/:previewId/stream`

**URL Parameters:**

| Parameter | Type   | Required | Description              |
|-----------|--------|----------|--------------------------|
| previewId | String | Yes      | Identifier of the preview |

**Example Request:**

```bash
curl -X GET http://localhost:3001/api/preview/550e8400-e29b-41d4-a716-446655440000/stream
```

**Response:**
Binary video data with appropriate content type (video/mp4 or video/webm)

### Get Preview Thumbnail

Returns the thumbnail image for a preview.

**Endpoint:** `GET /preview/:previewId/thumbnail`

**URL Parameters:**

| Parameter | Type   | Required | Description              |
|-----------|--------|----------|--------------------------|
| previewId | String | Yes      | Identifier of the preview |

**Example Request:**

```bash
curl -X GET http://localhost:3001/api/preview/550e8400-e29b-41d4-a716-446655440000/thumbnail
```

**Response:**
Binary image data with content type image/jpeg

### Download Preview File

Downloads a preview file securely.

**Endpoint:** `GET /preview/:previewId/download`

**URL Parameters:**

| Parameter | Type   | Required | Description              |
|-----------|--------|----------|--------------------------|
| previewId | String | Yes      | Identifier of the preview |

**Example Request:**

```bash
curl -X GET http://localhost:3001/api/preview/550e8400-e29b-41d4-a716-446655440000/download
```

**Response:**
Binary file data with appropriate content type based on file extension

### Delete Preview

Deletes a specific preview and its associated files.

**Endpoint:** `DELETE /preview/:previewId`

**URL Parameters:**

| Parameter | Type   | Required | Description              |
|-----------|--------|----------|--------------------------|
| previewId | String | Yes      | Identifier of the preview |

**Example Request:**

```bash
curl -X DELETE http://localhost:3001/api/preview/550e8400-e29b-41d4-a716-446655440000
```

**Response:**

```json
{
  "success": true,
  "message": "Preview 550e8400-e29b-41d4-a716-446655440000 deleted successfully"
}
```

### Service Health Check

Checks the health of the service and its dependencies.

**Endpoint:** `GET /health`

**Example Request:**

```bash
curl -X GET http://localhost:3001/api/health
```

**Response:**

```json
{
  "status": "healthy",
  "system": {
    "service": "preview-generation-service",
    "version": "1.0.0",
    "timestamp": "2023-06-12T14:32:45.123Z",
    "environment": "development"
  },
  "database": {
    "connected": true,
    "name": "preview-service",
    "host": "mongodb:27017"
  },
  "storage": {
    "healthy": true,
    "type": "s3",
    "bucket": "previews"
  }
}
```

## SDK Integration Examples

### JavaScript/TypeScript

```typescript
import axios from 'axios';
import FormData from 'form-data';
import fs from 'fs';

class PreviewServiceClient {
  private baseUrl: string;
  
  constructor({
    host = 'localhost',
    port = 3001,
    useHttps = false,
    basePath = '/api'
  } = {}) {
    const protocol = useHttps ? 'https' : 'http';
    this.baseUrl = `${protocol}://${host}:${port}${basePath}`;
  }
  
  async generatePreviews(videoPath: string, sourceId: string, userId: string, options = {}) {
    const formData = new FormData();
    formData.append('source', fs.createReadStream(videoPath));
    formData.append('sourceId', sourceId);
    formData.append('userId', userId);
    
    // Add optional parameters
    Object.entries(options).forEach(([key, value]) => {
      formData.append(key, value);
    });
    
    const response = await axios.post(`${this.baseUrl}/preview/generate`, formData, {
      headers: formData.getHeaders()
    });
    
    return response.data;
  }
  
  async getPreviewsBySource(sourceId: string, sourceType = 'segment') {
    const response = await axios.get(
      `${this.baseUrl}/preview/source/${sourceId}`,
      { params: { sourceType } }
    );
    
    return response.data;
  }
  
  async getPreviewDetails(previewId: string) {
    const response = await axios.get(`${this.baseUrl}/preview/${previewId}`);
    return response.data;
  }
  
  async deletePreview(previewId: string) {
    const response = await axios.delete(`${this.baseUrl}/preview/${previewId}`);
    return response.data;
  }
  
  getPreviewStreamUrl(previewId: string) {
    return `${this.baseUrl}/preview/${previewId}/stream`;
  }
  
  getThumbnailUrl(previewId: string) {
    return `${this.baseUrl}/preview/${previewId}/thumbnail`;
  }
  
  getPreviewDownloadUrl(previewId: string) {
    return `${this.baseUrl}/preview/${previewId}/download`;
  }
  
  async checkHealth() {
    const response = await axios.get(`${this.baseUrl}/health`);
    return response.data;
  }
}

// Usage
(async () => {
  // Default configuration (localhost:3001)
  const client = new PreviewServiceClient();
  
  // Or with custom configuration
  const productionClient = new PreviewServiceClient({
    host: 'preview-service.example.com',
    port: 443,
    useHttps: true
  });
  
  try {
    // Generate previews
    const result = await client.generatePreviews(
      './test-video.mp4',
      'video-123',
      'user-456',
      { previewCount: 3, quality: 'medium' }
    );
    
    console.log(`Generated ${result.data.previews.length} previews`);
    
    // Get previews for a source
    const previews = await client.getPreviewsBySource('video-123');
    console.log(previews.data.previews);
    
  } catch (error) {
    console.error('Error:', error.message);
  }
})();
```

### React Native Example

```jsx
import React, { useState, useEffect } from 'react';
import { View, Text, Image, FlatList, Button, StyleSheet } from 'react-native';
import Video from 'react-native-video';
import axios from 'axios';
import DocumentPicker from 'react-native-document-picker';

// Configure these based on your environment
const SERVICE_HOST = 'your-service-url.com'; // or 'localhost' for local development
const SERVICE_PORT = 3001;  
const USE_HTTPS = false;    // Set to true for production
const API_URL = `${USE_HTTPS ? 'https' : 'http'}://${SERVICE_HOST}:${SERVICE_PORT}/api`;

const PreviewScreen = () => {
  const [previews, setPreviews] = useState([]);
  const [selectedPreview, setSelectedPreview] = useState(null);
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    // Fetch previews for a source
    fetchPreviews('video-123');
  }, []);
  
  const fetchPreviews = async (sourceId) => {
    try {
      setLoading(true);
      const response = await axios.get(`${API_URL}/preview/source/${sourceId}`);
      setPreviews(response.data.data.previews);
      setLoading(false);
    } catch (error) {
      console.error('Error fetching previews:', error);
      setLoading(false);
    }
  };
  
  const uploadVideo = async () => {
    try {
      // Pick a video file
      const result = await DocumentPicker.pick({
        type: [DocumentPicker.types.video],
      });
      
      const video = result[0];
      
      // Check for supported formats
      const fileExtension = video.name.split('.').pop().toLowerCase();
      if (!['mp4', 'webm'].includes(fileExtension)) {
        alert('Unsupported format. Please select an MP4 or WebM file.');
        return;
      }
      
      setLoading(true);
      
      // Create form data
      const formData = new FormData();
      formData.append('source', {
        uri: video.uri,
        type: video.type,
        name: video.name,
      });
      formData.append('sourceId', 'upload-' + Date.now());
      formData.append('userId', 'user-123');
      formData.append('previewCount', '3');
      formData.append('quality', 'medium');
      
      // Upload and generate previews
      const response = await axios.post(`${API_URL}/preview/generate`, formData, {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      });
      
      // Fetch the newly created previews
      if (response.data.success) {
        await fetchPreviews(response.data.data.previews[0].sourceId);
      }
      
      setLoading(false);
    } catch (error) {
      console.error('Error uploading video:', error);
      setLoading(false);
    }
  };
  
  const renderPreviewItem = ({ item }) => (
    <View style={styles.previewItem}>
      <Image
        source={{ uri: item.thumbnailUrl }}
        style={styles.thumbnail}
      />
      <Text>Position: {item.position}</Text>
      <Text>Duration: {item.duration}s</Text>
      <Button
        title="Play"
        onPress={() => setSelectedPreview(item)}
      />
    </View>
  );
  
  return (
    <View style={styles.container}>
      <Button title="Upload Video" onPress={uploadVideo} />
      
      {loading ? (
        <Text>Loading...</Text>
      ) : (
        <>
          <Text style={styles.title}>Previews</Text>
          <FlatList
            data={previews}
            renderItem={renderPreviewItem}
            keyExtractor={item => item.previewId}
            horizontal
          />
          
          {selectedPreview && (
            <View style={styles.videoContainer}>
              <Text>Playing: {selectedPreview.position}</Text>
              <Video
                source={{ uri: selectedPreview.previewUrl }}
                style={styles.video}
                controls
                resizeMode="contain"
              />
              <Button
                title="Close"
                onPress={() => setSelectedPreview(null)}
              />
            </View>
          )}
        </>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
  },
  title: {
    fontSize: 18,
    fontWeight: 'bold',
    marginVertical: 8,
  },
  previewItem: {
    marginRight: 12,
    width: 160,
  },
  thumbnail: {
    width: 160,
    height: 90,
    borderRadius: 4,
  },
  videoContainer: {
    marginTop: 24,
    alignItems: 'center',
  },
  video: {
    width: '100%',
    height: 200,
  },
});

export default PreviewScreen;
```

## Environment Configuration

The service host and port can be configured through environment variables:

```env
# Server configuration
PREVIEW_SERVICE_HOST=localhost
PREVIEW_SERVICE_PORT=3001
PREVIEW_SERVICE_USE_HTTPS=false
API_PREFIX=/api

# Storage settings
ALLOW_S3_BUCKET_CREATE=false  # Set to 'true' only in development
```

When using Docker Compose, you can configure these in your `.env` file or pass them directly to the containers.

## S3 Key Structure Convention

All files stored by the service follow this structure:

```
{userId}/previews/{previewId}.{format}
```

Example:
```
user-123/previews/550e8400-e29b-41d4-a716-446655440000.mp4
user-123/previews/550e8400-e29b-41d4-a716-446655440000_thumb.jpg
```

Each user's content is isolated in their own folder to maintain separation and security.

## Error Handling

The API returns standard HTTP status codes along with a JSON response body:

### Success Response

```json
{
  "success": true,
  "message": "Operation successful",
  "data": { ... }
}
```

### Error Response

```json
{
  "success": false,
  "message": "Error description"
}
```

### Common Status Codes

- 200: Success
- 400: Bad Request (invalid parameters)
- 404: Resource Not Found
- 500: Internal Server Error

## Rate Limiting

The current implementation does not include rate limiting. For production use, implement rate limiting to prevent abuse.

## Concurrency Control

Preview generation is limited to processing 3 concurrent preview extractions to prevent CPU/memory spikes.

## Webhook Support

For long-running preview generation tasks, consider implementing webhook support to notify your application when previews are ready.

## Summary of Changes

The API documentation now reflects the latest enhancements:

1. **Secure File Access**: Removed insecure file path endpoint and added secure `/preview/:previewId/download` endpoint
2. **Input Format Validation**: Specified supported formats (.mp4, .webm) in request parameters
3. **User Folder Structure**: Added userFolder field to response examples
4. **S3 Key Structure**: Added documentation on the structure convention
5. **Enhanced Environment Configuration**: Added ALLOW_S3_BUCKET_CREATE variable
6. **Concurrency Control**: Added notes about the limitation on concurrent processing
