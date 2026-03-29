# Media Cache Helper - Production-Ready Android Library

Instagram jaisa social media app ke liye optimized media caching library. Image, Video, SVG sabhi ke liye work karti hai.

## Features


- **Two-tier Caching**: Memory (LRU) + Disk cache
- **Main Thread Callbacks**: ALL callbacks delivered on UI thread via Handler
- **Duplicate URL Handling**: Same URL requests queued - no duplicate downloads
- **Progress Tracking**: Real-time download progress
- **Context-based**: Proper Android cache directory (`context.getCacheDir()`)
- **Java 7 Compatible**: No lambdas, no streams, no try-with-resources
- **No Dependencies**: Pure Android SDK, no external libraries

---

## Setup

### Installation

Add JitPack to your project-level `build.gradle` file:

```gradle
allprojects {
    repositories {
        ... 
        maven { url 'https://jitpack.io' }
    }
}
```

Add the dependency to your app-level `build.gradle` file:

```gradle
dependencies {
    implementation 'com.github.Omdeepakumar:android-media-cache-library:1.0.0'
}
```

### 2. Add Internet Permission

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

---

## Quick Start

### 1. Initialize in Application Class

```java
// MyApplication.java
import android.app.Application;
import com.media.cache.MediaCacheHelper;

public class MyApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        // Initialize ONCE - pass application context
        MediaCacheHelper.init(this);
    }
}
```

### 2. Update AndroidManifest.xml

```xml
<application
    android:name=".MyApplication"
    ... >
```

### 3. Load Image in Activity

```java
// MainActivity.java
import android.os.Bundle;
import android.widget.ImageView;
import android.graphics.BitmapFactory;
import android.app.Activity;
import com.media.cache.MediaCacheHelper;
import com.media.cache.MediaType;
import com.media.cache.CacheCallback;

public class MainActivity extends Activity {

    private ImageView imageView;
    private MediaCacheHelper cache;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        imageView = findViewById(R.id.imageView);
        cache = MediaCacheHelper.getInstance();

        // Load image
        String imageUrl = "https://example.com/photo.jpg";
        cache.loadMedia(imageUrl, MediaType.IMAGE, new CacheCallback.SimpleCallback() {

            @Override
            public void onSuccess(String url, byte[] data, boolean fromCache) {
                // CRITICAL: This ALWAYS runs on MAIN THREAD
                // No need for runOnUiThread() or Handler!
                Bitmap bitmap = BitmapFactory.decodeByteArray(data, 0, data.length);
                imageView.setImageBitmap(bitmap);
            }

            @Override
            public void onError(String url, String error) {
                // Handle error - ALWAYS on main thread
                imageView.setImageResource(R.drawable.error_placeholder);
            }
        });
    }
}
```

---

## Configuration

### Configure Cache (Anywhere in App)

```java
MediaCacheHelper cache = MediaCacheHelper.getInstance();

// Memory cache: 100 items, 5 minute expiry
cache.setMemoryCacheSize(100);
cache.setMemoryCacheMaxAge(5 * 60 * 1000L);

// Disk cache: 500 MB, 30 day expiry
cache.setDiskCacheSize(500 * 1024 * 1024L);
cache.setDiskCacheMaxAge(30L * 24 * 60 * 60 * 1000);

// Network timeouts
cache.setConnectionTimeout(15000);  // 15 seconds
cache.setReadTimeout(30000);        // 30 seconds

// Disable/enable caches
cache.setMemoryCacheEnabled(true);
cache.setDiskCacheEnabled(true);
```

**Important**: Configuration methods do NOT clear existing cache data!

---

## API Reference

### Loading Methods

```java
// Async loading (recommended)
void loadMedia(String url, MediaType type, CacheCallback callback)

// Sync loading (blocking - AVOID on main thread!)
CacheEntry loadMediaSync(String url, MediaType type)
```

### Cache Management

```java
// Check if cached
boolean isCached(String url)

// Get cached data
byte[] getCachedData(String url)

// Remove specific URL
boolean remove(String url)

// Clear caches
void clearMemoryCache()
void clearDiskCache()
void clearAllCaches()

// Remove expired entries
int removeExpired()

// Get statistics
CacheStats getStats()

// Shutdown executor
void shutdown()
```

### Callback Interface

```java
public interface CacheCallback {
    // Called when media loaded (ALWAYS on main thread)
    void onSuccess(String url, byte[] data, boolean fromCache);

    // Called on error (ALWAYS on main thread)
    void onError(String url, String error);

    // Called during download
    void onProgress(String url, long bytesLoaded, long totalBytes);
}

// Simple callback - progress is optional
public static abstract class SimpleCallback implements CacheCallback {
    // Override success and error
}
```

---

## Android Lifecycle Integration

### Complete Activity Example

```java
public class ProfileActivity extends AppCompatActivity {

    private ImageView profileImage;
    private ImageView coverImage;
    private MediaCacheHelper cache;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_profile);

        profileImage = findViewById(R.id.profileImage);
        coverImage = findViewById(R.id.coverImage);
        cache = MediaCacheHelper.getInstance();

        // Load profile image
        loadProfileImage();

        // Load cover image
        loadCoverImage();
    }

    private void loadProfileImage() {
        String url = "https://socialapp.com/users/123/profile.jpg";
        cache.loadMedia(url, MediaType.IMAGE, new CacheCallback.SimpleCallback() {

            @Override
            public void onSuccess(String url, byte[] data, boolean fromCache) {
                Bitmap bitmap = BitmapFactory.decodeByteArray(data, 0, data.length);
                profileImage.setImageBitmap(bitmap);
                Log.d("Cache", "Profile: " + (fromCache ? "from cache" : "from network"));
            }

            @Override
            public void onError(String url, String error) {
                profileImage.setImageResource(R.drawable.default_avatar);
                Log.e("Cache", "Profile error: " + error);
            }
        });
    }

    private void loadCoverImage() {
        String url = "https://socialapp.com/users/123/cover.jpg";
        cache.loadMedia(url, MediaType.IMAGE, new CacheCallback.SimpleCallback() {

            @Override
            public void onSuccess(String url, byte[] data, boolean fromCache) {
                Bitmap bitmap = BitmapFactory.decodeByteArray(data, 0, data.length);
                coverImage.setImageBitmap(bitmap);
            }

            @Override
            public void onError(String url, String error) {
                coverImage.setImageResource(R.drawable.default_cover);
            }

            @Override
            public void onProgress(String url, long bytesLoaded, long totalBytes) {
                if (totalBytes > 0) {
                    int progress = (int) ((bytesLoaded * 100) / totalBytes);
                    progressBar.setProgress(progress);
                }
            }
        });
    }

    // Clear memory cache on low memory warning
    @Override
    public void onLowMemory() {
        super.onLowMemory();
        cache.clearMemoryCache();
        Log.d("Cache", "Memory cache cleared");
    }

    // Optional: Clear all on destroy
    @Override
    protected void onDestroy() {
        super.onDestroy();
        // Keep cache for app lifetime - only clear on exit
        // cache.clearAllCaches();
    }
}
```

### Video Player Integration

```java
public class VideoPlayerActivity extends AppCompatActivity {

    private VideoView videoView;
    private MediaCacheHelper cache;
    private ProgressDialog progressDialog;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_video_player);

        videoView = findViewById(R.id.videoView);
        cache = MediaCacheHelper.getInstance();
        progressDialog = new ProgressDialog(this);

        String videoUrl = "https://socialapp.com/videos/456.mp4";
        loadVideo(videoUrl);
    }

    private void loadVideo(String url) {
        progressDialog.setMessage("Loading video...");
        progressDialog.show();

        cache.loadMedia(url, MediaType.VIDEO, new CacheCallback.SimpleCallback() {

            @Override
            public void onSuccess(String url, byte[] data, boolean fromCache) {
                progressDialog.dismiss();

                // Play from byte array
                try {
                    File tempFile = File.createTempFile("video", ".mp4", getCacheDir());
                    FileOutputStream fos = new FileOutputStream(tempFile);
                    fos.write(data);
                    fos.close();

                    videoView.setVideoPath(tempFile.getAbsolutePath());
                    videoView.start();
                } catch (IOException e) {
                    Toast.makeText(VideoPlayerActivity.this,
                        "Error playing video", Toast.LENGTH_SHORT).show();
                }
            }

            @Override
            public void onError(String url, String error) {
                progressDialog.dismiss();
                Toast.makeText(VideoPlayerActivity.this,
                    "Failed to load video: " + error, Toast.LENGTH_SHORT).show();
            }

            @Override
            public void onProgress(String url, long bytesLoaded, long totalBytes) {
                if (totalBytes > 0) {
                    int progress = (int) ((bytesLoaded * 100) / totalBytes);
                    progressDialog.setMessage("Downloading: " + progress + "%");
                }
            }
        });
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (videoView != null) {
            videoView.stopPlayback();
        }
    }
}
```

### Feed/RecyclerView Example

```java
public class FeedAdapter extends RecyclerView.Adapter<FeedAdapter.ViewHolder> {

    private List<FeedItem> items;
    private MediaCacheHelper cache;

    public FeedAdapter(List<FeedItem> items) {
        this.items = items;
        this.cache = MediaCacheHelper.getInstance();
    }

    @Override
    public void onBindViewHolder(ViewHolder holder, int position) {
        FeedItem item = items.get(position);

        // Reset image before loading new one
        holder.imageView.setImageResource(R.drawable.placeholder);

        cache.loadMedia(item.getImageUrl(), MediaType.IMAGE,
            new CacheCallback.SimpleCallback() {

            @Override
            public void onSuccess(String url, byte[] data, boolean fromCache) {
                // Use Glide/Picasso to load into ImageView
                // Or decode directly
                Bitmap bitmap = BitmapFactory.decodeByteArray(data, 0, data.length);
                holder.imageView.setImageBitmap(bitmap);
            }

            @Override
            public void onError(String url, String error) {
                holder.imageView.setImageResource(R.drawable.error_image);
            }
        });
    }

    // ViewHolder class...
}
```

---

## Periodic Cleanup

```java
// In Application class or a utility class
public class CacheManager {

    private static ScheduledExecutorService scheduler;

    public static void startPeriodicCleanup() {
        scheduler = Executors.newSingleThreadScheduledExecutor();
        scheduler.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                if (MediaCacheHelper.isInitialized()) {
                    int removed = MediaCacheHelper.getInstance().removeExpired();
                    Log.d("Cache", "Removed " + removed + " expired entries");

                    CacheStats stats = MediaCacheHelper.getInstance().getStats();
                    Log.d("Cache", stats.toString());
                }
            }
        }, 1, 1, TimeUnit.HOURS);  // Every hour
    }

    public static void stopPeriodicCleanup() {
        if (scheduler != null) {
            scheduler.shutdown();
        }
    }
}
```

---

## Memory Management

### Handling Low Memory

```java
public class MyApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        MediaCacheHelper.init(this);

        // Configure for memory-constrained devices
        MediaCacheHelper cache = MediaCacheHelper.getInstance();
        cache.setMemoryCacheSize(20);  // Smaller memory cache
    }

    @Override
    public void onLowMemory() {
        super.onLowMemory();
        // Clear memory cache immediately
        MediaCacheHelper.getInstance().clearMemoryCache();
        Log.w("App", "Low memory - cleared cache");
    }

    @Override
    public void onTrimMemory(int level) {
        super.onTrimMemory(level);
        if (level >= TRIM_MEMORY_MODERATE) {
            MediaCacheHelper.getInstance().clearMemoryCache();
        }
    }
}
```

---

## File Structure

```
com/media/cache/
├── MediaType.java           # Enum: IMAGE, VIDEO
├── CacheCallback.java       # Callback interface
├── CacheEntry.java          # Cache entry wrapper
├── MemoryCache.java         # LRU in-memory cache
├── DiskCache.java           # Persistent disk cache
└── MediaCacheHelper.java     # Main singleton class
```

---

## How It Works

```
loadMedia(url)
    │
    ▼
┌─────────────────┐
│  Check Memory   │  ← Fast, synchronized check
│  (LRU Cache)    │
└────────┬────────┘
         │ Found
         ▼
    Return data ← (onSuccess on Main Thread)

         │ Not Found
         ▼
┌─────────────────┐
│  Check Disk     │  ← Synchronous read
│  Cache          │
└────────┬────────┘
         │ Found
         ▼
    Save to Memory │ Return data ← (onSuccess on Main Thread)

         │ Not Found
         ▼
┌─────────────────┐
│  Check Pending  │  ← Same URL loading?
│  Callbacks      │
└────────┬────────┘
         │ Already loading
         ▼
    Add to pending list ← Done, callback will fire later

         │ New request
         ▼
┌─────────────────┐
│  Download from  │  ← Background thread
│  Network        │
└────────┬────────┘
         │
         ▼
    Save to Both Caches │ Notify ALL callbacks ← (on Main Thread)
```

---

## Important Notes

### 1. Callbacks Always on Main Thread

```java
// WRONG - thinking callbacks might be on background
cache.loadMedia(url, type, new SimpleCallback() {
    @Override
    public void onSuccess(String url, byte[] data, boolean fromCache) {
        // SAFE - This IS on main thread!
        imageView.setImageBitmap(bitmap);
    }
});
```

### 2. Context Leak Prevention

```java
// WRONG - passing Activity context
MediaCacheHelper.init(this);  // Activity context!

// RIGHT - always use Application context
MediaCacheHelper.init(getApplicationContext());
// Or in Application class
MediaCacheHelper.init(this);
```

### 3. Duplicate URL Handling

```java
// If multiple views load same URL simultaneously:
cache.loadMedia(url, type, callback1);  // First request starts download
cache.loadMedia(url, type, callback2);  // Added to pending list
cache.loadMedia(url, type, callback3);  // Added to pending list
// Only ONE network request, all 3 callbacks fire when done
```

### 4. SVG Support

```java
// For SVG images, use AndroidSVG library:
cache.loadMedia(url, MediaType.IMAGE, new SimpleCallback() {
    @Override
    public void onSuccess(String url, byte[] data, boolean fromCache) {
        SVG svg = SVG.getFromInputStream(
            new ByteArrayInputStream(data));
        PictureDrawable drawable = new PictureDrawable(svg.renderToPicture());
        imageView.setImageDrawable(drawable);
    }
});
```

---

## License

Free to use. No warranties.

## Changelog

### v2.0 - Production Ready
- Fixed: Callbacks now ALWAYS on main thread
- Fixed: Duplicate URL handling with callback queuing
- Fixed: Context-based disk cache directory
- Fixed: Configuration methods preserve cache data
- Added: Progress tracking for downloads
- Added: SVG support
- Added: Android lifecycle integration examples
