# Updated Architecture with Preview Generation Service

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                  CLIENT LAYER                                    │
│                                                                                 │
│   ┌───────────────────────────┐           ┌───────────────────────────┐         │
│   │                           │           │                           │         │
│   │      Swift iOS App        │           │      Android App          │         │
│   │   - Native UI             │           │   - Native UI             │         │
│   │   - WKWebView for YouTube │           │   - WebView for YouTube   │         │
│   │   - Auto-play previews    │           │   - Auto-play previews    │         │
│   │   - Push notifications    │           │   - Push notifications    │         │
│   │   - Local storage         │           │   - Local storage         │         │
│   │                           │           │                           │         │
│   └───────────────┬───────────┘           └───────────────┬───────────┘         │
│                   │                                       │                     │
└───────────────────┼───────────────────────────────────────┼─────────────────────┘
                    │                                       │                      
                    │            REST/GraphQL APIs          │                      
                    └───────────────────┬───────────────────┘                      
                                        │                                          
┌───────────────────────────────────────┼───────────────────────────────────────┐
│                                       ▼                                        │
│                              API GATEWAY LAYER                                 │
│                                                                               │
│     ┌───────────────────────────────────────────────────────────────────┐     │
│     │                                                                   │     │
│     │                    Next.js Backend API                            │     │
│     │   - Authentication & Authorization                                │     │
│     │   - API orchestration                                            │     │
│     │   - Request routing                                              │     │
│     │   - Response formatting                                          │     │
│     │                                                                   │     │
│     └────┬────────────┬─────────────────────┬──────────────────┬───────┘     │
│          │            │                     │                  │              │
└──────────┼────────────┼─────────────────────┼──────────────────┼──────────────┘
           │            │                     │                  │               
           │            │                     │                  │               
┌──────────┼────────────┼─────────────────────┼──────────────────┼──────────────┐
│          │   SERVICE  │   LAYER             │                  │              │
│          ▼            ▼                     ▼                  ▼              │
│  ┌────────────────┐ ┌───────────────┐ ┌────────────────┐ ┌───────────────┐   │
│  │                │ │               │ │                │ │               │   │
│  │ Wowza Streaming│ │ Social Media  │ │ Notification   │ │ Preview      │   │
│  │ Service        │ │ Service       │ │ Service        │ │ Generation   │   │
│  │ - Live streams │ │ - YouTube     │ │ - Push notif.  │ │ Service      │   │
│  │ - Processing   │ │ - OAuth       │ │ - Event mgmt   │ │ - Create     │   │
│  │ - Segments     │ │ - Uploads     │ │ - Preferences  │ │   previews   │   │
│  │                │ │               │ │ - History      │ │ - Optimize   │   │
│  │                │ │               │ │                │ │   for mobile │   │
│  │                │ │               │ │                │ │ - Generate   │   │
│  │                │ │               │ │                │ │   thumbnails │   │
│  └───────┬────────┘ └────────┬──────┘ └────────┬───────┘ └───────┬───────┘   │
│          │                   │                 │                 │           │
└──────────┼───────────────────┼─────────────────┼─────────────────┼───────────┘
           │                   │                 │                 │            
           │                   │                 │                 │            
┌──────────┼───────────────────┼─────────────────┼─────────────────┼───────────┐
│ MESSAGING│& STORAGE LAYER    │                 │                 │           │
│          ▼                   ▼                 ▼                 ▼           │
│     ┌────────────────────────────────────────────────────────────┐           │
│     │                                                            │           │
│     │                   RabbitMQ                                 │           │
│     │   - Exchange-based routing                                 │           │
│     │   - Multiple queue patterns                                │           │
│     │   - Message acknowledgment                                 │           │
│     │                                                            │           │
│     └───────────────────────────────┬────────────────────────────┘           │
│                                     │                                        │
│     ┌─────────────────┐    ┌────────┴──────────┐    ┌────────────────────┐  │
│     │                 │    │                   │    │                    │  │
│     │     Redis       │    │     MongoDB       │    │  Storage Provider  │  │
│     │  - Caching      │    │  - User data      │    │  - S3 Storage      │  │
│     │  - Session mgmt │    │  - Stream metadata│    │    OR             │  │
│     │  - Rate limiting│    │  - Notifications  │    │  - NFS Server      │  │
│     │                 │    │  - Previews data  │    │  - Video content   │  │
│     │                 │    │  - Analytics      │    │  - Previews        │  │
│     │                 │    │                   │    │  - Thumbnails      │  │
│     └─────────────────┘    └───────────────────┘    └────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │                                         
                                     ▼                                         
┌────────────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL SERVICES                                  │
│                                                                            │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                │
│  │                │  │                │  │                │                │
│  │  Firebase      │  │  Apple         │  │  YouTube       │                │
│  │  Cloud         │  │  Push          │  │  API           │                │
│  │  Messaging     │  │  Notification  │  │                │                │
│  │  (FCM)         │  │  Service       │  │                │                │
│  │                │  │  (APNS)        │  │                │                │
│  └────────────────┘  └────────────────┘  └────────────────┘                │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

## Preview Generation Service in Detail

The Preview Generation Service is now a distinct microservice in your architecture, responsible for creating optimized video previews that enable the auto-play functionality in your mobile apps. Here's how it integrates with your system:

### Core Responsibilities

1. **Preview Creation**
   - Extracts multiple short clips (3-5 seconds) from stream recordings
   - Creates variations from different parts of the video (start, middle, end)
   - Optimizes encoding for mobile playback

2. **Mobile Optimization**
   - Reduces resolution for quick loading (typically 480p or lower)
   - Uses efficient codecs (H.264/AAC) for broad device compatibility
   - Applies bitrate optimization (800kbps video, 64kbps audio)
   - Creates lightweight files optimized for cellular networks

3. **Thumbnail Generation**
   - Creates thumbnail images for each preview
   - Optimizes thumbnails for feed display
   - Provides fast initial loading before video plays

4. **Storage Management**
   - Stores previews in your chosen storage system (S3 or NFS)
   - Manages preview lifecycle (creation, retrieval, deletion)
   - Maintains metadata about previews for efficient retrieval

### Workflow Integration

1. **Triggered After Stream Processing**
   - When a stream segment is created or uploaded
   - RabbitMQ message initiates preview generation
   - Works asynchronously to not block other operations

2. **Feeds Content to Mobile Apps**
   - Previews are fetched by mobile apps when scrolling through content
   - Multiple preview options allow for variety in the feed
   - Preview URLs are included in feed API responses

3. **Performance Benefits**
   - Reduces bandwidth usage compared to full video playback
   - Improves scroll performance with optimized file sizes
   - Enhances user experience with immediate visual feedback

### Technical Components

- **FFmpeg Processing Pipeline**: For extracting and encoding video segments
- **Preview Selection Algorithm**: Chooses visually interesting segments
- **Quality Optimization**: Balances file size vs. quality for mobile
- **Caching Strategy**: Efficiently stores frequently accessed previews

