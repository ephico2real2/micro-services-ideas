
# Auto-Play Video Previews on Scroll Feature

Adding auto-play preview functionality as users scroll through content is a powerful engagement feature for your streaming app. Here's how to implement this across both platforms:


# Auto-Play Video Preview Architecture

## System Components

### 1. Preview Generation Service
- Generates low-resolution, short (3-5 second) preview clips
- Creates multiple preview segments for each stream
- Optimizes for mobile playback (low bitrate, efficient codec)

### 2. Content Delivery System
- Pre-loads previews based on scroll position
- Efficient caching strategy
- Progressive loading for smooth experience

### 3. Client-Side Implementations
- Platform-specific scroll tracking
- Video playback management
- Resource optimization

### 4. Analytics Integration
- Tracks preview engagement
- Measures conversion from preview to full video view
- Personalizes preview selection based on user behavior

## Workflow Diagram

```bash
┌─────────────────────────────────────────────────────────────────┐
│                        Client Applications                       │
│                                                                 │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐
│  │                             │  │                             │
│  │      iOS Swift App          │  │      Android App            │
│  │                             │  │                             │
│  │  ┌─────────────────────┐    │  │  ┌─────────────────────┐    │
│  │  │ Scroll Tracking     │    │  │  │ Scroll Tracking     │    │
│  │  │ - Visibility calc   │    │  │  │ - Visibility calc   │    │
│  │  │ - Event throttling  │    │  │  │ - Event throttling  │    │
│  │  └─────────────────────┘    │  │  └─────────────────────┘    │
│  │                             │  │                             │
│  │  ┌─────────────────────┐    │  │  ┌─────────────────────┐    │
│  │  │ Preview Player      │    │  │  │ Preview Player      │    │
│  │  │ - Auto-play mgmt    │    │  │  │ - Auto-play mgmt    │    │
│  │  │ - Mute control      │    │  │  │ - Mute control      │    │
│  │  │ - Resource mgmt     │    │  │  │ - Resource mgmt     │    │
│  │  └─────────────────────┘    │  │  └─────────────────────┘    │
│  │                             │  │                             │
│  │  ┌─────────────────────┐    │  │  ┌─────────────────────┐    │
│  │  │ Preloading System   │    │  │  │ Preloading System   │    │
│  │  │ - Buffer mgmt       │    │  │  │ - Buffer mgmt       │    │
│  │  │ - Network aware     │    │  │  │ - Network aware     │    │
│  │  └─────────────────────┘    │  │  └─────────────────────┘    │
│  │                             │  │                             │
│  └─────────────────────────────┘  └─────────────────────────────┘
│                 ▲                               ▲                │
└─────────────────┼───────────────────────────────┼────────────────┘
                  │                               │                 
                  │           REST API            │                 
                  └───────────────────┬───────────┘                 
                                      │                             
┌─────────────────────────────────────┼─────────────────────────────┐
│                                     ▼                             │
│                          Backend Services                         │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                 Preview Generation Service                   │  │
│  │                                                             │  │
│  │  ┌─────────────────────┐      ┌───────────────────────────┐ │  │
│  │  │                     │      │                           │ │  │
│  │  │ Stream Segmenter    │      │ Preview Optimizer         │ │  │
│  │  │ - Multiple previews │      │ - Format optimization     │ │  │
│  │  │ - Time selection    │      │ - Multi-resolution        │ │  │
│  │  │ - Duration control  │      │ - Thumbnail generation    │ │  │
│  │  │                     │      │                           │ │  │
│  │  └─────────────────────┘      └───────────────────────────┘ │  │
│  │                                                             │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                   Preview Delivery API                       │  │
│  │                                                             │  │
│  │  ┌─────────────────────┐      ┌───────────────────────────┐ │  │
│  │  │                     │      │                           │ │  │
│  │  │ Feed Management     │      │ Preview Manifest API      │ │  │
│  │  │ - Video discovery   │      │ - HLS/DASH manifests      │ │  │
│  │  │ - Pagination        │      │ - Preview URLs            │ │  │
│  │  │ - Feed composition  │      │ - Adaptive bitrates       │ │  │
│  │  │                     │      │                           │ │  │
│  │  └─────────────────────┘      └───────────────────────────┘ │  │
│  │                                                             │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    Analytics Integration                     │  │
│  │                                                             │  │
│  │  ┌─────────────────────┐      ┌───────────────────────────┐ │  │
│  │  │                     │      │                           │ │  │
│  │  │ Engagement Tracking │      │ Personalization Engine    │ │  │
│  │  │ - View events       │      │ - Preview selection       │ │  │
│  │  │ - Play duration     │      │ - Content recommendation  │ │  │
│  │  │ - Conversion rate   │      │ - A/B testing            │ │  │
│  │  │                     │      │                           │ │  │
│  │  └─────────────────────┘      └───────────────────────────┘ │  │
│  │                                                             │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
                                 │                                   
                                 ▼                                   
┌───────────────────────────────────────────────────────────────────┐
│                         Storage Layer                             │
│                                                                   │
│   ┌────────────────────────┐        ┌────────────────────────┐    │
│   │                        │        │                        │    │
│   │  Preview Storage       │        │  CDN / Edge Cache      │    │
│   │  - Optimized segments  │        │  - Global distribution │    │
│   │  - Multiple formats    │        │  - Low latency access  │    │
│   │  - Thumbnails          │        │  - Preview caching     │    │
│   │                        │        │                        │    │
│   └────────────────────────┘        └────────────────────────┘    │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

## Key Technical Components

1. **Preview Generation**
   - Automatically generate 3-5 second preview clips from each stream
   - Create multiple previews from different parts of the content
   - Optimize for mobile playback (low resolution, efficient codecs)
   - Store alongside the main content with appropriate metadata

2. **Feed API with Preview Support**
   - Enhance content listing APIs to include preview URLs
   - Support pagination with preview metadata
   - Implement sorting/filtering with preview availability

3. **Client-Side Scroll Tracking**
   - Implement efficient scroll position monitoring
   - Calculate visibility percentage of video items
   - Trigger preview playback at appropriate thresholds
   - Implement playback prioritization (play most visible)

4. **Resource Management**
   - Pause previews when scrolled out of view
   - Limit concurrent preview playback 
   - Respect user data preferences and network conditions
   - Memory management to prevent app slowdowns

5. **Performance Optimization**
   - Pre-loading of nearby preview content
   - Progressive quality improvement
   - Caching strategies for frequently viewed content
   - Bandwidth-aware playback decisions


############### IOS  Start

// AutoPlayVideoManager.swift - Handles auto-play preview functionality for iOS

import UIKit
import AVFoundation

// MARK: - AutoPlayConfiguration

struct AutoPlayConfiguration {
    /// Minimum visibility percentage to trigger autoplay (0.0-1.0)
    let visibilityThreshold: CGFloat = 0.7
    
    /// Maximum number of simultaneous preview players
    let maxConcurrentPreviews: Int = 2
    
    /// Whether previews should autoplay on cellular data
    let allowAutoplayOnCellular: Bool = false
    
    /// Whether videos should play with sound
    let playWithSound: Bool = false
    
    /// Preview preloading range (how many items ahead to preload)
    let preloadRange: Int = 3
}

// MARK: - PreviewItem Protocol

protocol PreviewItem: AnyObject {
    var videoId: String { get }
    var previewURL: URL? { get }
    var playerView: UIView { get }
    var playerLayer: AVPlayerLayer? { get set }
    var player: AVPlayer? { get set }
    var isPlayingPreview: Bool { get set }
    
    func prepareForReuse()
    func configurePlayer(with url: URL)
}

// MARK: - AutoPlayVideoManager

class AutoPlayVideoManager {
    // Singleton instance
    static let shared = AutoPlayVideoManager()
    
    private let configuration = AutoPlayConfiguration()
    private var visibleItems = [String: PreviewItem]()
    private var preloadedPlayers = [String: AVPlayer]()
    private var playingItems = [String]()
    
    private var isWifiOnly: Bool {
        return configuration.allowAutoplayOnCellular == false
    }
    
    private var networkType: String {
        // Simplified network check - implement actual Reachability in production
        return "wifi" // or "cellular"
    }
    
    private init() {
        // Register for app state notifications
        NotificationCenter.default.addObserver(self, 
                                              selector: #selector(handleAppDidEnterBackground), 
                                              name: UIApplication.didEnterBackgroundNotification, 
                                              object: nil)
        
        NotificationCenter.default.addObserver(self, 
                                              selector: #selector(handleAppWillEnterForeground), 
                                              name: UIApplication.willEnterForegroundNotification, 
                                              object: nil)
    }
    
    deinit {
        NotificationCenter.default.removeObserver(self)
    }
    
    // MARK: - Public Methods
    
    /// Track all visible preview items for scroll management
    func trackVisibleItems(_ items: [PreviewItem], in scrollView: UIScrollView) {
        // Clear previous tracking
        let previouslyVisibleIDs = Set(visibleItems.keys)
        visibleItems.removeAll()
        
        // Calculate visible rect
        let visibleRect = CGRect(origin: scrollView.contentOffset, size: scrollView.bounds.size)
        
        // Evaluate each item
        for item in items {
            // Convert item frame to scrollView coordinates
            guard let itemSuperview = item.playerView.superview else { continue }
            let itemFrame = scrollView.convert(item.playerView.frame, from: itemSuperview)
            
            // Calculate intersection with visible area
            let intersection = visibleRect.intersection(itemFrame)
            guard !intersection.isNull else { continue }
            
            // Calculate visibility percentage
            let visibleArea = intersection.width * intersection.height
            let totalArea = itemFrame.width * itemFrame.height
            let visibilityPercentage = visibleArea / totalArea
            
            // Track the item with its visibility
            visibleItems[item.videoId] = item
            
            // Manage playback based on visibility
            if visibilityPercentage >= configuration.visibilityThreshold {
                playPreviewIfNeeded(for: item)
            } else if item.isPlayingPreview {
                pausePreview(for: item)
            }
        }
        
        // Find items that are no longer visible
        let currentVisibleIDs = Set(visibleItems.keys)
        let removedIDs = previouslyVisibleIDs.subtracting(currentVisibleIDs)
        
        // Pause previews for items no longer visible
        for videoId in removedIDs {
            if let index = playingItems.firstIndex(of: videoId) {
                playingItems.remove(at: index)
            }
        }
        
        // Preload upcoming items
        preloadUpcomingItems(near: scrollView.contentOffset.y, in: scrollView, items: items)
    }
    
    /// Prepare manager for cell reuse
    func prepareForReuse(item: PreviewItem) {
        pausePreview(for: item)
        item.prepareForReuse()
        
        if let index = playingItems.firstIndex(of: item.videoId) {
            playingItems.remove(at: index)
        }
    }
    
    /// Manually play preview for an item
    func playPreview(for item: PreviewItem) {
        guard let previewURL = item.previewURL else { return }
        
        // Check if already prepared
        if item.player == nil {
            // Check if we have a preloaded player
            if let preloadedPlayer = preloadedPlayers[item.videoId] {
                item.player = preloadedPlayer
                configurePlayerLayer(for: item)
                preloadedPlayers.removeValue(forKey: item.videoId)
            } else {
                // Create new player
                item.configurePlayer(with: previewURL)
            }
        }
        
        // Start playback
        item.player?.play()
        item.isPlayingPreview = true
        
        // Manage sound
        item.player?.isMuted = !configuration.playWithSound
        
        // Loop playback
        setupLooping(for: item)
        
        // Track as playing
        if !playingItems.contains(item.videoId) {
            playingItems.append(item.videoId)
        }
        
        // Limit concurrent playback
        enforceMaxConcurrentPreviews()
    }
    
    /// Manually pause preview for an item
    func pausePreview(for item: PreviewItem) {
        item.player?.pause()
        item.isPlayingPreview = false
        
        if let index = playingItems.firstIndex(of: item.videoId) {
            playingItems.remove(at: index)
        }
    }
    
    /// Pause all currently playing previews
    func pauseAllPreviews() {
        for videoId in playingItems {
            if let item = visibleItems[videoId] {
                item.player?.pause()
                item.isPlayingPreview = false
            }
        }
        playingItems.removeAll()
    }
    
    // MARK: - Private Methods
    
    private func playPreviewIfNeeded(for item: PreviewItem) {
        // Check network conditions
        if isWifiOnly && networkType != "wifi" {
            return
        }
        
        // Don't restart if already playing
        if item.isPlayingPreview {
            return
        }
        
        playPreview(for: item)
    }
    
    private func configurePlayerLayer(for item: PreviewItem) {
        guard let player = item.player else { return }
        
        if item.playerLayer == nil {
            let layer = AVPlayerLayer(player: player)
            layer.videoGravity = .resizeAspectFill
            layer.frame = item.playerView.bounds
            item.playerView.layer.addSublayer(layer)
            item.playerLayer = layer
        } else {
            item.playerLayer?.player = player
        }
    }
    
    private func setupLooping(for item: PreviewItem) {
        guard let player = item.player else { return }
        
        NotificationCenter.default.removeObserver(self, 
                                                name: .AVPlayerItemDidPlayToEndTime, 
                                                object: player.currentItem)
        
        NotificationCenter.default.addObserver(forName: .AVPlayerItemDidPlayToEndTime, 
                                              object: player.currentItem, 
                                              queue: .main) { [weak self] _ in
            player.seek(to: .zero)
            player.play()
        }
    }
    
    private func enforceMaxConcurrentPreviews() {
        // If we're under the limit, no need to do anything
        if playingItems.count <= configuration.maxConcurrentPreviews {
            return
        }
        
        // Keep only the most recent previews playing
        let itemsToKeepPlaying = Array(playingItems.suffix(configuration.maxConcurrentPreviews))
        let itemsToPause = playingItems.filter { !itemsToKeepPlaying.contains($0) }
        
        // Pause older previews
        for videoId in itemsToPause {
            if let item = visibleItems[videoId] {
                pausePreview(for: item)
            }
        }
    }
    
    private func preloadUpcomingItems(near yOffset: CGFloat, in scrollView: UIScrollView, items: [PreviewItem]) {
        // Skip preloading if we're on cellular and wifi-only is enabled
        if isWifiOnly && networkType != "wifi" {
            return
        }
        
        // Find items within preload range
        let preloadItems = items.filter { item in
            guard let superview = item.playerView.superview else { return false }
            let frame = scrollView.convert(item.playerView.frame, from: superview)
            
            // Check if item is ahead in the scroll direction (below current position)
            let isAhead = frame.origin.y > yOffset
            
            // Check if within preload range
            let distance = frame.origin.y - yOffset
            let isWithinRange = distance > 0 && distance < scrollView.bounds.height * CGFloat(configuration.preloadRange)
            
            // Not already visible or preloaded
            let isNotVisible = visibleItems[item.videoId] == nil
            let isNotPreloaded = preloadedPlayers[item.videoId] == nil
            let hasNoPlayer = item.player == nil
            
            return isAhead && isWithinRange && isNotVisible && isNotPreloaded && hasNoPlayer
        }
        
        // Preload items (limited to preload range)
        for item in preloadItems.prefix(configuration.preloadRange) {
            guard let previewURL = item.previewURL, preloadedPlayers[item.videoId] == nil else { continue }
            
            // Create a player but don't attach it yet
            let asset = AVAsset(url: previewURL)
            let playerItem = AVPlayerItem(asset: asset)
            let player = AVPlayer(playerItem: playerItem)
            
            // Store for later use
            preloadedPlayers[item.videoId] = player
        }
    }
    
    // MARK: - App State Handlers
    
    @objc private func handleAppDidEnterBackground() {
        pauseAllPreviews()
    }
    
    @objc private func handleAppWillEnterForeground() {
        // Optionally, resume previews when app comes back to foreground
        // or let the scroll tracking handle it on next scroll
    }
}

// MARK: - UITableViewCell Implementation Example

class VideoPreviewCell: UITableViewCell, PreviewItem {
    // MARK: - UI Elements
    
    @IBOutlet weak var previewContainerView: UIView!
    @IBOutlet weak var titleLabel: UILabel!
    @IBOutlet weak var descriptionLabel: UILabel!
    
    // MARK: - PreviewItem Protocol Properties
    
    var videoId: String = ""
    var previewURL: URL?
    var playerView: UIView { return previewContainerView }
    var playerLayer: AVPlayerLayer?
    var player: AVPlayer?
    var isPlayingPreview: Bool = false
    
    // MARK: - Cell Lifecycle
    
    override func prepareForReuse() {
        super.prepareForReuse()
        
        AutoPlayVideoManager.shared.prepareForReuse(item: self)
        
        // Clean up player
        playerLayer?.removeFromSuperlayer()
        playerLayer = nil
        player = nil
        isPlayingPreview = false
    }
    
    // MARK: - Configuration
    
    func configure(with video: VideoModel) {
        videoId = video.id
        titleLabel.text = video.title
        descriptionLabel.text = video.description
        previewURL = URL(string: video.previewUrl)
    }
    
    func configurePlayer(with url: URL) {
        let asset = AVAsset(url: url)
        let playerItem = AVPlayerItem(asset: asset)
        player = AVPlayer(playerItem: playerItem)
        
        let layer = AVPlayerLayer(player: player)
        layer.videoGravity = .resizeAspectFill
        layer.frame = previewContainerView.bounds
        previewContainerView.layer.addSublayer(layer)
        playerLayer = layer
    }
}

// MARK: - UITableViewController Implementation Example

class VideoFeedViewController: UITableViewController {
    var videos: [VideoModel] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Load videos from API
        loadVideos()
    }
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        
        // Check for visible cells and start previews
        updateVisibleCells()
    }
    
    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        
        // Pause all previews when view disappears
        AutoPlayVideoManager.shared.pauseAllPreviews()
    }
    
    // MARK: - UITableViewDataSource
    
    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return videos.count
    }
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "VideoPreviewCell", for: indexPath) as! VideoPreviewCell
        
        // Configure cell with video data
        let video = videos[indexPath.row]
        cell.configure(with: video)
        
        return cell
    }
    
    // MARK: - UIScrollViewDelegate
    
    override func scrollViewDidScroll(_ scrollView: UIScrollView) {
        updateVisibleCells()
    }
    
    // MARK: - Private Methods
    
    private func loadVideos() {
        // Load videos from API
        // This is a placeholder implementation
        NetworkManager.shared.fetchVideos { [weak self] videos in
            self?.videos = videos
            self?.tableView.reloadData()
            self?.updateVisibleCells()
        }
    }
    
    private func updateVisibleCells() {
        // Get all visible preview cells
        let visibleCells = tableView.visibleCells.compactMap { $0 as? VideoPreviewCell }
        
        // Update the AutoPlayVideoManager with currently visible cells
        AutoPlayVideoManager.shared.trackVisibleItems(visibleCells, in: tableView)
    }
}

// MARK: - Model

struct VideoModel {
    let id: String
    let title: String
    let description: String
    let previewUrl: String
    let fullVideoUrl: String
}

// MARK: - Network Manager (Simplified)

class NetworkManager {
    static let shared = NetworkManager()
    
    func fetchVideos(completion: @escaping ([VideoModel]) -> Void) {
        // Simulate network request
        DispatchQueue.main.asyncAfter(deadline: .now() + 1.0) {
            // Sample data
            let videos = [
                VideoModel(
                    id: "1",
                    title: "Amazing Sunset",
                    description: "Beautiful sunset over the mountains",
                    previewUrl: "https://example.com/previews/sunset.mp4",
                    fullVideoUrl: "https://example.com/videos/sunset.mp4"
                ),
                VideoModel(
                    id: "2",
                    title: "City Skyline",
                    description: "Panoramic view of the city skyline",
                    previewUrl: "https://example.com/previews/skyline.mp4",
                    fullVideoUrl: "https://example.com/videos/skyline.mp4"
                ),
                // Add more sample videos...
            ]
            
            completion(videos)
        }
    }
}

############### IOS END ################# 


################# Android Start ################# 

package com.yourcompany.streaming.preview;

import android.content.Context;
import android.graphics.Rect;
import android.net.ConnectivityManager;
import android.net.NetworkCapabilities;
import android.util.Log;
import android.view.View;

import androidx.annotation.NonNull;
import androidx.lifecycle.DefaultLifecycleObserver;
import androidx.lifecycle.LifecycleOwner;
import androidx.recyclerview.widget.RecyclerView;

import com.google.android.exoplayer2.ExoPlayer;
import com.google.android.exoplayer2.MediaItem;
import com.google.android.exoplayer2.Player;
import com.google.android.exoplayer2.SimpleExoPlayer;
import com.google.android.exoplayer2.source.MediaSource;
import com.google.android.exoplayer2.source.ProgressiveMediaSource;
import com.google.android.exoplayer2.upstream.DefaultDataSourceFactory;
import com.google.android.exoplayer2.upstream.DefaultHttpDataSource;
import com.google.android.exoplayer2.upstream.cache.CacheDataSource;
import com.google.android.exoplayer2.upstream.cache.SimpleCache;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

/**
 * Manager class to handle auto-playing video previews when scrolling through content.
 * Designed to work with RecyclerView or similar scrolling container.
 */
public class AutoPlayVideoManager implements DefaultLifecycleObserver {
    private static final String TAG = "AutoPlayVideoManager";

    // Singleton instance
    private static AutoPlayVideoManager instance;

    // Configuration
    private final AutoPlayConfiguration config;

    // Cache for ExoPlayer instances
    private final Map<String, ExoPlayer> playerCache = new HashMap<>();
    private final Map<String, PreviewItem> visibleItems = new HashMap<>();
    private final List<String> playingItems = new ArrayList<>();
    private final Set<String> preloadingItems = new HashSet<>();

    // Video preview cache (provided by app)
    private SimpleCache videoCache;
    private Context context;

    /**
     * Get the singleton instance
     */
    public static synchronized AutoPlayVideoManager getInstance(Context context) {
        if (instance == null) {
            instance = new AutoPlayVideoManager(context);
        }
        return instance;
    }

    private AutoPlayVideoManager(Context context) {
        this.context = context.getApplicationContext();
        this.config = new AutoPlayConfiguration();
    }

    /**
     * Set the video cache for more efficient preview loading
     */
    public void setVideoCache(SimpleCache videoCache) {
        this.videoCache = videoCache;
    }

    /**
     * Track visible items in a RecyclerView
     */
    public void trackVisibleItems(RecyclerView recyclerView) {
        // Clear previous visible items list
        Set<String> previouslyVisibleIds = new HashSet<>(visibleItems.keySet());
        visibleItems.clear();

        // Calculate visible rect
        Rect recyclerViewRect = new Rect();
        recyclerView.getGlobalVisibleRect(recyclerViewRect);

        // Track each visible item
        for (int i = 0; i < recyclerView.getChildCount(); i++) {
            View view = recyclerView.getChildAt(i);
            int position = recyclerView.getChildAdapterPosition(view);
            if (position == RecyclerView.NO_POSITION) continue;

            RecyclerView.ViewHolder viewHolder = recyclerView.findViewHolderForAdapterPosition(position);
            if (!(viewHolder instanceof PreviewItem)) continue;

            PreviewItem item = (PreviewItem) viewHolder;
            
            // Calculate visibility
            Rect itemRect = new Rect();
            item.getPlayerView().getGlobalVisibleRect(itemRect);
            
            // If view is not visible at all, skip
            if (!Rect.intersects(itemRect, recyclerViewRect)) continue;

            // Calculate intersection area (visible portion)
            Rect intersection = new Rect();
            intersection.setIntersect(recyclerViewRect, itemRect);
            float visibleArea = intersection.width() * intersection.height();
            float totalArea = item.getPlayerView().getWidth() * item.getPlayerView().getHeight();
            float visibilityPercentage = visibleArea / totalArea;

            // Track this item
            visibleItems.put(item.getVideoId(), item);

            // Manage playback based on visibility
            if (visibilityPercentage >= config.getVisibilityThreshold()) {
                playPreviewIfNeeded(item);
            } else if (item.isPlayingPreview()) {
                pausePreview(item);
            }
        }

        // Find items that are no longer visible
        Set<String> currentVisibleIds = new HashSet<>(visibleItems.keySet());
        Set<String> removedIds = new HashSet<>(previouslyVisibleIds);
        removedIds.removeAll(currentVisibleIds);

        // Pause previews for items no longer visible
        for (String videoId : removedIds) {
            playingItems.remove(videoId);
        }

        // Preload upcoming items
        preloadUpcomingItems(recyclerView);
    }

    /**
     * Preload videos that will soon be visible
     */
    private void preloadUpcomingItems(RecyclerView recyclerView) {
        // Skip preloading if on cellular and settings restrict it
        if (config.isWifiOnlyAutoplay() && !isWifiConnected()) {
            return;
        }

        // Identify the last visible position
        int lastVisiblePosition = -1;
        for (int i = 0; i < recyclerView.getChildCount(); i++) {
            View child = recyclerView.getChildAt(i);
            int position = recyclerView.getChildAdapterPosition(child);
            if (position > lastVisiblePosition) {
                lastVisiblePosition = position;
            }
        }

        // Nothing to preload if we can't identify visible positions
        if (lastVisiblePosition == -1) return;

        // Preload the next few items
        int itemCount = recyclerView.getAdapter().getItemCount();
        int preloadRange = Math.min(lastVisiblePosition + config.getPreloadRange(), itemCount - 1);

        for (int i = lastVisiblePosition + 1; i <= preloadRange; i++) {
            // Get ViewHolder for the position
            RecyclerView.ViewHolder viewHolder = recyclerView.findViewHolderForAdapterPosition(i);
            if (!(viewHolder instanceof PreviewItem)) continue;

            PreviewItem item = (PreviewItem) viewHolder;
            String videoId = item.getVideoId();

            // Skip if already preloaded or playing
            if (playerCache.containsKey(videoId) || 
                preloadingItems.contains(videoId) || 
                visibleItems.containsKey(videoId)) {
                continue;
            }

            // Add to preloading set
            preloadingItems.add(videoId);

            // Start preloading
            String previewUrl = item.getPreviewUrl();
            if (previewUrl != null && !previewUrl.isEmpty()) {
                preparePlayerForItem(item, false);
            }
        }
    }

    /**
     * Play preview for a specific item
     */
    public void playPreview(PreviewItem item) {
        // Check if we're allowed to play on cellular
        if (config.isWifiOnlyAutoplay() && !isWifiConnected()) {
            return;
        }

        // Don't do anything if already playing
        if (item.isPlayingPreview()) {
            return;
        }

        // Prepare the player if needed
        ExoPlayer player = preparePlayerForItem(item, true);
        if (player == null) return;

        // Start playback
        player.setPlayWhenReady(true);
        item.setPlayingPreview(true);

        // Manage sound
        player.setVolume(config.isPlayWithSound() ? 1.0f : 0.0f);

        // Track as playing
        if (!playingItems.contains(item.getVideoId())) {
            playingItems.add(item.getVideoId());
        }

        // Limit concurrent playback
        enforceMaxConcurrentPreviews();
    }

    /**
     * Prepare but don't play (for preloading)
     */
    private ExoPlayer preparePlayerForItem(PreviewItem item, boolean attachToView) {
        String videoId = item.getVideoId();
        String previewUrl = item.getPreviewUrl();

        if (previewUrl == null || previewUrl.isEmpty()) {
            return null;
        }

        // Use cached player if available
        ExoPlayer player = playerCache.get(videoId);
        
        // Create a new player if needed
        if (player == null) {
            player = new SimpleExoPlayer.Builder(context).build();
            
            // Configure player
            player.setRepeatMode(Player.REPEAT_MODE_ALL);
            
            // Create data source factory
            DefaultHttpDataSource.Factory httpDataSourceFactory = new DefaultHttpDataSource.Factory()
                    .setAllowCrossProtocolRedirects(true);
            
            // Use cache if available
            DefaultDataSourceFactory dataSourceFactory;
            if (videoCache != null) {
                CacheDataSource.Factory cacheDataSourceFactory = new CacheDataSource.Factory()
                        .setCache(videoCache)
                        .setUpstreamDataSourceFactory(httpDataSourceFactory);
                dataSourceFactory = new DefaultDataSourceFactory(context, null, cacheDataSourceFactory);
            } else {
                dataSourceFactory = new DefaultDataSourceFactory(context, null, httpDataSourceFactory);
            }
            
            // Create media source
            MediaSource mediaSource = new ProgressiveMediaSource.Factory(dataSourceFactory)
                    .createMediaSource(MediaItem.fromUri(previewUrl));
            
            // Set media source
            player.setMediaSource(mediaSource);
            player.prepare();
            
            // Save to cache
            playerCache.put(videoId, player);
        }
        
        // Attach player to view if requested
        if (attachToView) {
            item.setPlayer(player);
        }
        
        // Remove from preloading set
        preloadingItems.remove(videoId);
        
        return player;
    }

    /**
     * Pause preview for a specific item
     */
    public void pausePreview(PreviewItem item) {
        ExoPlayer player = playerCache.get(item.getVideoId());
        if (player != null) {
            player.setPlayWhenReady(false);
        }
        
        item.setPlayingPreview(false);
        playingItems.remove(item.getVideoId());
    }

    /**
     * Play preview if it meets the requirements
     */
    private void playPreviewIfNeeded(PreviewItem item) {
        // Check network requirements
        if (config.isWifiOnlyAutoplay() && !isWifiConnected()) {
            return;
        }

        // Don't do anything if already playing
        if (item.isPlayingPreview()) {
            return;
        }

        playPreview(item);
    }

    /**
     * Enforce the maximum number of concurrent previews
     */
    private void enforceMaxConcurrentPreviews() {
        // If we're under the limit, no need to do anything
        if (playingItems.size() <= config.getMaxConcurrentPreviews()) {
            return;
        }

        // Keep only the most recent previews playing
        int itemsToKeep = config.getMaxConcurrentPreviews();
        List<String> itemsToStop = new ArrayList<>(playingItems.subList(0, playingItems.size() - itemsToKeep));

        // Pause older previews
        for (String videoId : itemsToStop) {
            PreviewItem item = visibleItems.get(videoId);
            if (item != null) {
                pausePreview(item);
            }
        }
    }

    /**
     * Pause all currently playing previews
     */
    public void pauseAllPreviews() {
        for (String videoId : new ArrayList<>(playingItems)) {
            PreviewItem item = visibleItems.get(videoId);
            if (item != null) {
                pausePreview(item);
            }
        }
        playingItems.clear();
    }

    /**
     * Clean up resources for an item (e.g., when recycling a view)
     */
    public void prepareForReuse(PreviewItem item) {
        pausePreview(item);
        item.setPlayer(null);
        
        // Remove from playing list
        playingItems.remove(item.getVideoId());
    }

    /**
     * Release all resources (call when activity/fragment is destroyed)
     */
    public void releaseAll() {
        pauseAllPreviews();
        
        // Release all cached players
        for (ExoPlayer player : playerCache.values()) {
            player.release();
        }
        
        playerCache.clear();
        visibleItems.clear();
        playingItems.clear();
        preloadingItems.clear();
    }

    /**
     * Check if device is connected to WiFi
     */
    private boolean isWifiConnected() {
        ConnectivityManager connectivityManager = 
                (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
        
        if (connectivityManager == null) {
            return false;
        }
        
        NetworkCapabilities capabilities = 
                connectivityManager.getNetworkCapabilities(connectivityManager.getActiveNetwork());
        
        return capabilities != null && capabilities.hasTransport(NetworkCapabilities.TRANSPORT_WIFI);
    }

    // Lifecycle methods
    @Override
    public void onPause(@NonNull LifecycleOwner owner) {
        pauseAllPreviews();
    }

    @Override
    public void onDestroy(@NonNull LifecycleOwner owner) {
        releaseAll();
    }

    /**
     * Interface for items that can show video previews
     */
    public interface PreviewItem {
        String getVideoId();
        String getPreviewUrl();
        View getPlayerView();
        boolean isPlayingPreview();
        void setPlayingPreview(boolean isPlaying);
        void setPlayer(ExoPlayer player);
    }

    /**
     * Configuration for auto-play behavior
     */
    public static class AutoPlayConfiguration {
        private float visibilityThreshold = 0.7f;
        private int maxConcurrentPreviews = 2;
        private boolean wifiOnlyAutoplay = true;
        private boolean playWithSound = false;
        private int preloadRange = 3;

        public float getVisibilityThreshold() {
            return visibilityThreshold;
        }

        public void setVisibilityThreshold(float visibilityThreshold) {
            this.visibilityThreshold = visibilityThreshold;
        }

        public int getMaxConcurrentPreviews() {
            return maxConcurrentPreviews;
        }

        public void setMaxConcurrentPreviews(int maxConcurrentPreviews) {
            this.maxConcurrentPreviews = maxConcurrentPreviews;
        }

        public boolean isWifiOnlyAutoplay() {
            return wifiOnlyAutoplay;
        }

        public void setWifiOnlyAutoplay(boolean wifiOnlyAutoplay) {
            this.wifiOnlyAutoplay = wifiOnlyAutoplay;
        }

        public boolean isPlayWithSound() {
            return playWithSound;
        }

        public void setPlayWithSound(boolean playWithSound) {
            this.playWithSound = playWithSound;
        }

        public int getPreloadRange() {
            return preloadRange;
        }

        public void setPreloadRange(int preloadRange) {
            this.preloadRange = preloadRange;
        }
    }
}

// -----------------------------------------------------------------------------------
// Example implementation in a RecyclerView Adapter
// -----------------------------------------------------------------------------------

package com.yourcompany.streaming.ui;

import android.content.Context;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;

import androidx.annotation.NonNull;
import androidx.recyclerview.widget.RecyclerView;

import com.google.android.exoplayer2.ExoPlayer;
import com.google.android.exoplayer2.ui.PlayerView;
import com.yourcompany.streaming.R;
import com.yourcompany.streaming.model.VideoModel;
import com.yourcompany.streaming.preview.AutoPlayVideoManager;

import java.util.List;

public class VideoFeedAdapter extends RecyclerView.Adapter<VideoFeedAdapter.VideoViewHolder> {

    private Context context;
    private List<VideoModel> videos;
    private AutoPlayVideoManager autoPlayManager;

    public VideoFeedAdapter(Context context, List<VideoModel> videos) {
        this.context = context;
        this.videos = videos;
        this.autoPlayManager = AutoPlayVideoManager.getInstance(context);
    }

    @NonNull
    @Override
    public VideoViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(context).inflate(R.layout.item_video_preview, parent, false);
        return new VideoViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull VideoViewHolder holder, int position) {
        VideoModel video = videos.get(position);
        holder.bind(video);
    }

    @Override
    public void onViewRecycled(@NonNull VideoViewHolder holder) {
        super.onViewRecycled(holder);
        autoPlayManager.prepareForReuse(holder);
    }

    @Override
    public int getItemCount() {
        return videos.size();
    }

    class VideoViewHolder extends RecyclerView.ViewHolder implements AutoPlayVideoManager.PreviewItem {
        private TextView titleTextView;
        private TextView descriptionTextView;
        private PlayerView playerView;
        private String videoId;
        private String previewUrl;
        private boolean isPlayingPreview;

        public VideoViewHolder(@NonNull View itemView) {
            super(itemView);
            titleTextView = itemView.findViewById(R.id.text_title);
            descriptionTextView = itemView.findViewById(R.id.text_description);
            playerView = itemView.findViewById(R.id.player_view);
        }

        void bind(VideoModel video) {
            this.videoId = video.getId();
            this.previewUrl = video.getPreviewUrl();
            this.isPlayingPreview = false;

            titleTextView.setText(video.getTitle());
            descriptionTextView.setText(video.getDescription());

            // Reset player view
            playerView.setPlayer(null);
        }

        @Override
        public String getVideoId() {
            return videoId;
        }

        @Override
        public String getPreviewUrl() {
            return previewUrl;
        }

        @Override
        public View getPlayerView() {
            return playerView;
        }

        @Override
        public boolean isPlayingPreview() {
            return isPlayingPreview;
        }

        @Override
        public void setPlayingPreview(boolean isPlaying) {
            this.isPlayingPreview = isPlaying;
        }

        @Override
        public void setPlayer(ExoPlayer player) {
            playerView.setPlayer(player);
        }
    }
}

// -----------------------------------------------------------------------------------
// Example implementation in an Activity or Fragment
// -----------------------------------------------------------------------------------

package com.yourcompany.streaming.ui;

import android.os.Bundle;

import androidx.appcompat.app.AppCompatActivity;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import com.google.android.exoplayer2.database.ExoDatabaseProvider;
import com.google.android.exoplayer2.upstream.cache.LeastRecentlyUsedCacheEvictor;
import com.google.android.exoplayer2.upstream.cache.SimpleCache;
import com.yourcompany.streaming.R;
import com.yourcompany.streaming.model.VideoModel;
import com.yourcompany.streaming.preview.AutoPlayVideoManager;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

public class VideoFeedActivity extends AppCompatActivity {

    private RecyclerView recyclerView;
    private VideoFeedAdapter adapter;
    private AutoPlayVideoManager autoPlayManager;
    private SimpleCache videoCache;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_video_feed);

        // Initialize ExoPlayer cache
        initializeVideoCache();

        // Initialize AutoPlayVideoManager
        autoPlayManager = AutoPlayVideoManager.getInstance(this);
        autoPlayManager.setVideoCache(videoCache);
        getLifecycle().addObserver(autoPlayManager);

        // Set up RecyclerView
        recyclerView = findViewById(R.id.recycler_videos);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        // Load videos
        List<VideoModel> videos = loadVideos();
        adapter = new VideoFeedAdapter(this, videos);
        recyclerView.setAdapter(adapter);

        // Set up scroll listener for auto-play
        recyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
            @Override
            public void onScrollStateChanged(@NonNull RecyclerView recyclerView, int newState) {
                super.onScrollStateChanged(recyclerView, newState);
            }

            @Override
            public void onScrolled(@NonNull RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                autoPlayManager.trackVisibleItems(recyclerView);
            }
        });
    }

    private void initializeVideoCache() {
        // Create a cache with 100MB max size
        LeastRecentlyUsedCacheEvictor evictor = new LeastRecentlyUsedCacheEvictor(100 * 1024 * 1024);
        ExoDatabaseProvider databaseProvider = new ExoDatabaseProvider(this);
        File cacheDir = new File(getCacheDir(), "video-cache");
        videoCache = new SimpleCache(cacheDir, evictor, databaseProvider);
    }

    private List<VideoModel> loadVideos() {
        // This would typically come from your API
        // For this example, we'll use sample data
        List<VideoModel> videos = new ArrayList<>();
        videos.add(new VideoModel("1", "Amazing Sunset", 
                "Beautiful sunset over the mountains",
                "https://example.com/previews/sunset.mp4",
                "https://example.com/videos/sunset.mp4"));
        videos.add(new VideoModel("2", "City Skyline", 
                "Panoramic view of the city skyline",
                "https://example.com/previews/skyline.mp4",
                "https://example.com/videos/skyline.mp4"));
        // Add more videos...
        return videos;
    }

    @Override
    protected void onPause() {
        super.onPause();
        // Lifecycle observer will handle pausing videos
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // Lifecycle observer will handle releasing resources
        
        // Release video cache
        if (videoCache != null) {
            try {
                videoCache.release();
                videoCache = null;
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}

// -----------------------------------------------------------------------------------
// Video Model class
// -----------------------------------------------------------------------------------

package com.yourcompany.streaming.model;

public class VideoModel {
    private String id;
    private String title;
    private String description;
    private String previewUrl;
    private String fullVideoUrl;

    public VideoModel(String id, String title, String description, String previewUrl, String fullVideoUrl) {
        this.id = id;
        this.title = title;
        this.description = description;
        this.previewUrl = previewUrl;
        this.fullVideoUrl = fullVideoUrl;
    }

    public String getId() {
        return id;
    }

    public String getTitle() {
        return title;
    }

    public String getDescription() {
        return description;
    }

    public String getPreviewUrl() {
        return previewUrl;
    }

    public String getFullVideoUrl() {
        return fullVideoUrl;
    }
}



################# Android end ################# 


################# previewGenerationService Start ################# 

// src/services/previewGenerationService.js - Backend service for generating video previews

const path = require('path');
const ffmpeg = require('fluent-ffmpeg');
const { v4: uuidv4 } = require('uuid');
const fs = require('fs');
const { promisify } = require('util');
const config = require('../../config');
const storageService = require('./storageService');

// Promisify fs functions
const mkdir = promisify(fs.mkdir);
const writeFile = promisify(fs.writeFile);
const unlink = promisify(fs.unlink);

// Temporary directory for processing
const TEMP_DIR = path.join(process.cwd(), 'temp');

/**
 * Initialize preview generation service
 */
async function initializePreviewService() {
  try {
    // Ensure temp directory exists
    if (!fs.existsSync(TEMP_DIR)) {
      await mkdir(TEMP_DIR, { recursive: true });
    }
    
    console.log('Preview generation service initialized');
    return true;
  } catch (error) {
    console.error('Failed to initialize preview generation service:', error);
    throw error;
  }
}

/**
 * Generate optimized preview clips from a video segment
 * @param {string} segmentId - ID of the segment
 * @param {Object} options - Preview generation options
 * @returns {Promise<Array>} - Array of generated preview info
 */
async function generatePreviews(segmentId, options = {}) {
  try {
    // Get the segment details
    const segment = await storageService.getSegmentDetails(segmentId);
    
    if (!segment) {
      throw new Error(`Segment with ID ${segmentId} not found`);
    }
    
    // Determine preview settings
    const previewCount = options.previewCount || 3; // Generate multiple previews by default
    const previewDuration = options.previewDuration || 5; // 5 seconds per preview
    const previewFormats = options.formats || ['mp4']; // Default format
    
    // Create temp paths
    const tempInputPath = path.join(TEMP_DIR, `${uuidv4()}_input.${segment.format}`);
    
    try {
      // Get the segment file
      let videoBuffer;
      
      if (segment.storageType === 'local') {
        videoBuffer = fs.readFileSync(segment.filePath);
      } else {
        // For S3 storage, download to temp
        const s3Client = global.s3Client;
        const response = await s3Client.getObject({
          Bucket: config.storage.s3.bucket,
          Key: segment.s3Key
        }).promise();
        
        videoBuffer = response.Body;
      }
      
      // Write to temp file
      await writeFile(tempInputPath, videoBuffer);
      
      // Get video duration using ffprobe
      const videoDuration = await getVideoDuration(tempInputPath);
      
      // Calculate appropriate preview positions based on segment duration
      const previewPositions = calculatePreviewPositions(videoDuration, previewCount, previewDuration);
      
      // Generate each preview
      const previewPromises = [];
      for (let i = 0; i < previewPositions.length; i++) {
        const position = previewPositions[i];
        
        // Generate for each requested format
        for (const format of previewFormats) {
          previewPromises.push(
            generatePreviewClip(
              segment,
              tempInputPath,
              position.start, 
              previewDuration,
              format,
              options
            )
          );
        }
      }
      
      // Wait for all previews to be generated
      const previews = await Promise.all(previewPromises);
      
      // Filter out any failed previews
      const successfulPreviews = previews.filter(Boolean);
      
      return successfulPreviews;
    } finally {
      // Clean up temp files
      if (fs.existsSync(tempInputPath)) {
        await unlink(tempInputPath);
      }
    }
  } catch (error) {
    console.error(`Error generating previews for segment ${segmentId}:`, error);
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
function calculatePreviewPositions(duration, count, clipDuration) {
  const positions = [];
  
  // Handle case where video is shorter than desired total preview duration
  if (duration <= count * clipDuration) {
    // Just use the beginning for a single preview
    positions.push({ start: 0, position: 'beginning' });
    return positions;
  }
  
  // Always include the beginning if we have multiple positions
  positions.push({ start: 0, position: 'beginning' });
  
  // Evaluate for interesting content in the middle
  // For now, we'll use equally spaced positions, but this could be enhanced with content analysis
  const interval = (duration - clipDuration) / (count > 1 ? count : 1);
  
  for (let i = 1; i < count - 1; i++) {
    const start = Math.floor(interval * i);
    positions.push({ start, position: `middle_${i}` });
  }
  
  // Always include part near the end for the last preview (if we have more than one)
  if (count > 1) {
    // Leave enough room for the full clip duration
    const lastStart = Math.max(0, Math.floor(duration - clipDuration - 1));
    positions.push({ start: lastStart, position: 'end' });
  }
  
  return positions;
}

/**
 * Generate a single preview clip
 * @param {Object} segment - Original segment info
 * @param {string} inputPath - Path to input video
 * @param {number} startTime - Start time in seconds
 * @param {number} duration - Duration in seconds
 * @param {string} format - Output format
 * @param {Object} options - Additional options
 * @returns {Promise<Object>} - Preview info
 */
async function generatePreviewClip(segment, inputPath, startTime, duration, format, options) {
  // Generate unique ID for this preview
  const previewId = uuidv4();
  
  // Create temp output path
  const tempOutputPath = path.join(TEMP_DIR, `${previewId}_preview.${format}`);
  const tempThumbnailPath = path.join(TEMP_DIR, `${previewId}_thumbnail.jpg`);
  
  try {
    // Generate optimized preview
    await generateOptimizedPreview(
      inputPath, 
      tempOutputPath, 
      startTime, 
      duration, 
      options
    );
    
    // Generate thumbnail from the preview
    await generateThumbnail(tempOutputPath, tempThumbnailPath);
    
    // Read the generated files
    const previewBuffer = fs.readFileSync(tempOutputPath);
    const thumbnailBuffer = fs.readFileSync(tempThumbnailPath);
    
    // Determine preview type
    const previewType = 'autoplay';
    
    // Create metadata for the preview
    const metadata = {
      id: previewId,
      originalSegmentId: segment.id,
      streamId: segment.streamId,
      userId: segment.userId,
      format,
      startTime,
      duration,
      previewType,
      createdAt: new Date(),
      size: previewBuffer.length,
      hasThumbnail: true,
      position: startTime === 0 ? 'beginning' : (startTime > segment.duration * 0.7 ? 'end' : 'middle')
    };
    
    // File naming
    const previewFileName = `${segment.streamId}_${previewId}_${previewType}.${format}`;
    const metadataFileName = `${segment.streamId}_${previewId}_${previewType}.json`;
    const thumbnailFileName = `${segment.streamId}_${previewId}_thumbnail.jpg`;
    
    // Store preview files
    if (config.storage.type === 'local' || config.storage.type === 'hybrid') {
      // Store preview
      const previewPath = path.join(config.storage.local.directory, 'previews', previewFileName);
      await ensureDirectoryExists(path.dirname(previewPath));
      await writeFile(previewPath, previewBuffer);
      
      // Store metadata
      const metadataPath = path.join(config.storage.local.directory, 'previews', metadataFileName);
      await writeFile(metadataPath, JSON.stringify(metadata, null, 2));
      
      // Store thumbnail
      const thumbnailPath = path.join(config.storage.local.directory, 'previews', thumbnailFileName);
      await writeFile(thumbnailPath, thumbnailBuffer);
      
      // Update metadata with paths
      metadata.filePath = previewPath;
      metadata.thumbnailPath = thumbnailPath;
    }
    
    if (config.storage.type === 's3' || config.storage.type === 'hybrid') {
      const s3Client = global.s3Client;
      
      // Store preview
      const previewKey = `previews/${previewFileName}`;
      await s3Client.putObject({
        Bucket: config.storage.s3.bucket,
        Key: previewKey,
        Body: previewBuffer,
        ContentType: `video/${format}`
      }).promise();
      
      // Store metadata
      const metadataKey = `previews/${metadataFileName}`;
      await s3Client.putObject({
        Bucket: config.storage.s3.bucket,
        Key: metadataKey,
        Body: JSON.stringify(metadata, null, 2),
        ContentType: 'application/json'
      }).promise();
      
      // Store thumbnail
      const thumbnailKey = `previews/${thumbnailFileName}`;
      await s3Client.putObject({
        Bucket: config.storage.s3.bucket,
        Key: thumbnailKey,
        Body: thumbnailBuffer,
        ContentType: 'image/jpeg'
      }).promise();
      
      // Update metadata with S3 keys
      metadata.s3Key = previewKey;
      metadata.thumbnailKey = thumbnailKey;
    }
    
    // Register the preview
    await storageService.registerPreview(previewId, metadata);
    
    return {
      id: previewId,
      originalSegmentId: segment.id,
      streamId: segment.streamId,
      startTime,
      duration,
      format,
      previewType,
      size: metadata.size,
      position: metadata.position
    };
  } catch (error) {
    console.error(`Error generating preview clip:`, error);
    return null;
  } finally {
    // Clean up temp files
    if (fs.existsSync(tempOutputPath)) {
      await unlink(tempOutputPath);
    }
    if (fs.existsSync(tempThumbnailPath)) {
      await unlink(tempThumbnailPath);
    }
  }
}

/**
 * Generate optimized preview using FFmpeg
 * @param {string} inputPath - Input video path
 * @param {string} outputPath - Output preview path
 * @param {number} startTime - Start time in seconds
 * @param {number} duration - Duration in seconds
 * @param {Object} options - Optimization options
 * @returns {Promise<void>}
 */
function generateOptimizedPreview(inputPath, outputPath, startTime, duration, options) {
  return new Promise((resolve, reject) => {
    // Determine quality settings
    const videoBitrate = options.videoBitrate || 800; // kbps
    const audioBitrate = options.audioBitrate || 64;  // kbps
    const videoCodec = options.videoCodec || 'libx264';
    const audioCodec = options.audioCodec || 'aac';
    const width = options.width || 480;
    const height = options.height || 0; // 0 means maintain aspect ratio
    const framerate = options.framerate || 24;
    
    // Create ffmpeg command
    let command = ffmpeg(inputPath)
      .setStartTime(startTime)
      .setDuration(duration)
      .output(outputPath)
      .outputOptions([
        `-c:v ${videoCodec}`,         // Video codec
        `-c:a ${audioCodec}`,         // Audio codec
        `-b:v ${videoBitrate}k`,      // Video bitrate
        `-b:a ${audioBitrate}k`,      // Audio bitrate
        `-r ${framerate}`,            // Frame rate
        '-pix_fmt yuv420p',           // Pixel format for compatibility
        '-movflags faststart',        // Optimize for web playback
        '-profile:v baseline',        // Use baseline profile for compatibility
        '-level 3.0'                  // Compatibility level
      ]);
    
    // Add resolution settings
    if (width > 0) {
      if (height > 0) {
        command = command.size(`${width}x${height}`);
      } else {
        command = command.size(`${width}x?`);
      }
    }
    
    // Run the command
    command
      .on('end', () => {
        console.log(`Generated preview: ${outputPath}`);
        resolve();
      })
      .on('error', (err) => {
        console.error('Error generating preview:', err);
        reject(err);
      })
      .run();
  });
}

/**
 * Generate thumbnail from a video
 * @param {string} videoPath - Path to video
 * @param {string} thumbnailPath - Path for output thumbnail
 * @returns {Promise<void>}
 */
function generateThumbnail(videoPath, thumbnailPath) {
  return new Promise((resolve, reject) => {
    // Take a thumbnail from 1 second into the preview
    ffmpeg(videoPath)
      .screenshots({
        timestamps: [1],
        filename: path.basename(thumbnailPath),
        folder: path.dirname(thumbnailPath),
        size: '320x180' // 16:9 thumbnail
      })
      .on('end', () => {
        resolve();
      })
      .on('error', (err) => {
        reject(err);
      });
  });
}

/**
 * Get video duration using ffprobe
 * @param {string} videoPath - Path to video file
 * @returns {Promise<number>} - Duration in seconds
 */
function getVideoDuration(videoPath) {
  return new Promise((resolve, reject) => {
    ffmpeg.ffprobe(videoPath, (err, metadata) => {
      if (err) {
        reject(err);
        return;
      }
      
      resolve(metadata.format.duration);
    });
  });
}

/**
 * Ensure a directory exists
 * @param {string} dirPath - Directory path
 */
async function ensureDirectoryExists(dirPath) {
  if (!fs.existsSync(dirPath)) {
    await mkdir(dirPath, { recursive: true });
  }
}

/**
 * Get previews for a segment
 * @param {string} segmentId - Segment ID
 * @returns {Promise<Array>} - List of previews
 */
async function getSegmentPreviews(segmentId) {
  try {
    // This would typically query your database or storage
    const allPreviews = await storageService.listPreviews({ 
      originalSegmentId: segmentId 
    });
    
    return allPreviews.map(preview => ({
      id: preview.id,
      originalSegmentId: preview.originalSegmentId,
      streamId: preview.streamId,
      startTime: preview.startTime,
      duration: preview.duration,
      format: preview.format,
      previewType: preview.previewType,
      size: preview.size,
      position: preview.position,
      createdAt: preview.createdAt,
      hasThumbnail: preview.hasThumbnail
    }));
  } catch (error) {
    console.error(`Error getting previews for segment ${segmentId}:`, error);
    throw error;
  }
}

/**
 * Get preview URL and thumbnail URL
 * @param {string} previewId - Preview ID
 * @param {number} expiresIn - Expiration time in seconds
 * @returns {Promise<Object>} - URLs for the preview
 */
async function getPreviewUrls(previewId, expiresIn = 3600) {
  try {
    const preview = await storageService.getPreviewDetails(previewId);
    
    if (!preview) {
      throw new Error(`Preview with ID ${previewId} not found`);
    }
    
    let previewUrl;
    let thumbnailUrl;
    
    if (preview.storageType === 'local') {
      // For local storage, create a URL through the API
      previewUrl = `${config.server.baseUrl}/api/previews/${previewId}/stream`;
      thumbnailUrl = preview.hasThumbnail 
        ? `${config.server.baseUrl}/api/previews/${previewId}/thumbnail` 
        : null;
    } else if (preview.s3Key) {
      // For S3 storage, generate a pre-signed URL
      const s3Client = global.s3Client;
      
      // Preview URL
      previewUrl = await s3Client.getSignedUrlPromise('getObject', {
        Bucket: config.storage.s3.bucket,
        Key: preview.s3Key,
        Expires: expiresIn
      });
      
      // Thumbnail URL if available
      thumbnailUrl = preview.hasThumbnail && preview.thumbnailKey 
        ? await s3Client.getSignedUrlPromise('getObject', {
            Bucket: config.storage.s3.bucket,
            Key: preview.thumbnailKey,
            Expires: expiresIn
          }) 
        : null;
    } else {
      throw new Error(`No storage information for preview ${previewId}`);
    }
    
    return {
      previewUrl,
      thumbnailUrl,
      expiresIn
    };
  } catch (error) {
    console.error(`Error getting URLs for preview ${previewId}:`, error);
    throw error;
  }
}

/**
 * Delete a preview
 * @param {string} previewId - ID of the preview to delete
 * @returns {Promise<boolean>} - Success status
 */
async function deletePreview(previewId) {
  try {
    const preview = await storageService.getPreviewDetails(previewId);
    
    if (!preview) {
      throw new Error(`Preview with ID ${previewId} not found`);
    }
    
    if (preview.storageType === 'local' || preview.storageType === 'hybrid') {
      // Delete from local storage
      if (preview.filePath && fs.existsSync(preview.filePath)) {
        await unlink(preview.filePath);
      }
      
      // Delete metadata file
      const metadataPath = preview.filePath?.replace(/\.(mp4|webm)$/, '.json');
      if (metadataPath && fs.existsSync(metadataPath)) {
        await unlink(metadataPath);
      }
      
      // Delete thumbnail
      if (preview.thumbnailPath && fs.existsSync(preview.thumbnailPath)) {
        await unlink(preview.thumbnailPath);
      }
    }
    
    if (preview.storageType === 's3' || preview.storageType === 'hybrid') {
      const s3Client = global.s3Client;
      
      // Delete preview file
      if (preview.s3Key) {
        await s3Client.deleteObject({
          Bucket: config.storage.s3.bucket,
          Key: preview.s3Key
        }).promise();
      }
      
      // Delete metadata file
      const metadataKey = preview.s3Key?.replace(/\.(mp4|webm)$/, '.json');
      if (metadataKey) {
        await s3Client.deleteObject({
          Bucket: config.storage.s3.bucket,
          Key: metadataKey
        }).promise();
      }
      
      // Delete thumbnail
      if (preview.thumbnailKey) {
        await s3Client.deleteObject({
          Bucket: config.storage.s3.bucket,
          Key: preview.thumbnailKey
        }).promise();
      }
    }
    
    // Remove from storage service registry
    await storageService.removePreview(previewId);
    
    return true;
  } catch (error) {
    console.error(`Error deleting preview ${previewId}:`, error);
    throw error;
  }
}

module.exports = {
  initializePreviewService,
  generatePreviews,
  getSegmentPreviews,
  getPreviewUrls,
  deletePreview
};

################# previewGenerationService end ################# 

