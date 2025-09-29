# FFmpeg-Go Usage Guide: Practical Examples and Patterns

## Table of Contents
1. [Basic Concepts](#basic-concepts)
2. [Input and Output Operations](#input-and-output-operations)
3. [Filtering and Transformations](#filtering-and-transformations)
4. [Advanced Patterns](#advanced-patterns)
5. [Error Handling and Debugging](#error-handling-and-debugging)
6. [Performance and Best Practices](#performance-and-best-practices)

## Basic Concepts

### The Fluent API Pattern

FFmpeg-Go uses method chaining to build processing pipelines:

```go
result := ffmpeg.Input("source.mp4").        // Start with input
    Filter("scale", ffmpeg.Args{"1280:720"}). // Apply transformation
    Output("output.mp4").                     // Specify output
    OverWriteOutput().                        // Add options
    Run()                                     // Execute
```

This approach mirrors FFmpeg's conceptual flow: **Input → Process → Output → Execute**

### Understanding Streams vs Nodes

- **Stream**: Represents a data flow (video/audio stream)
- **Node**: Represents an operation (input, filter, output)

```go
// Creating streams
inputStream := ffmpeg.Input("video.mp4")     // Input stream
filteredStream := inputStream.VFlip()        // Filtered stream  
outputStream := filteredStream.Output("out.mp4") // Output stream
```

## Input and Output Operations

### 1. File-Based Operations

```go
// Basic file input
input := ffmpeg.Input("input.mp4")

// Input with options
input := ffmpeg.Input("input.mp4", ffmpeg.KwArgs{
    "ss": "00:01:30",    // Start at 1:30
    "t":  "00:00:10",    // Duration 10 seconds
    "r":  "30",          // Frame rate
})

// Multiple inputs
video := ffmpeg.Input("video.mp4")
audio := ffmpeg.Input("audio.mp3")
overlay := ffmpeg.Input("watermark.png")
```

### 2. Pipe Operations

```go
// Reading from stdin
input := ffmpeg.Input("pipe:")

// Writing to stdout
input.Output("pipe:", ffmpeg.KwArgs{"format": "mp4"})

// Custom readers/writers
var inputReader io.Reader = getVideoStream()
var outputWriter io.Writer = getDestination()

err := ffmpeg.Input("pipe:").
    WithInput(inputReader).
    Output("pipe:").
    WithOutput(outputWriter).
    Run()
```

### 3. Memory Buffer Operations

```go
// Capture output in memory
buf := bytes.NewBuffer(nil)
err := ffmpeg.Input("input.mp4").
    Filter("select", ffmpeg.Args{"gte(n,10)"}).  // Frame 10
    Output("pipe:", ffmpeg.KwArgs{
        "vframes": 1,
        "format":  "image2", 
        "vcodec":  "mjpeg",
    }).
    WithOutput(buf).
    Run()

// Use the captured data
img, err := jpeg.Decode(buf)
```

### 4. Streaming Operations

```go
// Process video frames in real-time
func processVideoStream(inputFile, outputFile string) {
    // Read process: video → raw frames
    pr, pw := io.Pipe()
    go func() {
        defer pw.Close()
        ffmpeg.Input(inputFile).
            Output("pipe:", ffmpeg.KwArgs{
                "format":  "rawvideo",
                "pix_fmt": "rgb24",
            }).
            WithOutput(pw).
            Run()
    }()
    
    // Write process: processed frames → video
    pr2, pw2 := io.Pipe()
    go func() {
        defer pw2.Close()
        processFrames(pr, pw2) // Your custom processing
    }()
    
    // Final encoding
    ffmpeg.Input("pipe:", ffmpeg.KwArgs{
        "format":  "rawvideo", 
        "pix_fmt": "rgb24",
        "s":       "1920x1080",
    }).
    WithInput(pr2).
    Output(outputFile).
    Run()
}
```

## Filtering and Transformations

### 1. Video Filters

```go
// Scale video
input.Filter("scale", ffmpeg.Args{"1280:720"})
input.Filter("scale", ffmpeg.Args{"1280:-1"}) // Maintain aspect ratio

// Crop video  
input.Filter("crop", ffmpeg.Args{"800:600:100:50"}) // w:h:x:y

// Rotate video
input.Filter("rotate", ffmpeg.Args{"PI/4"}) // 45 degrees

// Apply effects
input.Filter("blur", ffmpeg.Args{"5:5"})
input.Filter("brightness", ffmpeg.Args{"0.1"})
input.Filter("contrast", ffmpeg.Args{"1.2"})

// Built-in convenience methods
input.VFlip()    // Vertical flip
input.HFlip()    // Horizontal flip
input.Trim(ffmpeg.KwArgs{"start": "10", "end": "20"})
```

### 2. Audio Filters

```go
// Volume adjustment
input.Filter("volume", ffmpeg.Args{"0.5"})

// Audio format conversion
input.Filter("aformat", ffmpeg.Args{"sample_fmts=s16:channel_layouts=stereo"})

// Audio effects
input.Filter("highpass", ffmpeg.Args{"f=200"})
input.Filter("lowpass", ffmpeg.Args{"f=3000"})
```

### 3. Complex Filtering

```go
// Overlay watermark
main := ffmpeg.Input("video.mp4")
watermark := ffmpeg.Input("logo.png").Filter("scale", ffmpeg.Args{"64:-1"})

result := ffmpeg.Filter(
    []*ffmpeg.Stream{main, watermark},
    "overlay",
    ffmpeg.Args{"10:10"}, // Position
    ffmpeg.KwArgs{"enable": "gte(t,5)"}, // Start at 5 seconds
)
```

### 4. Stream Splitting and Merging

```go
// Split input into multiple streams
input := ffmpeg.Input("input.mp4")
split := input.Split() // Creates a split node

// Get individual streams
stream0 := split.Get("0")
stream1 := split.Get("1")

// Process each stream differently
processed0 := stream0.Filter("scale", ffmpeg.Args{"1920:-1"})
processed1 := stream1.Filter("scale", ffmpeg.Args{"1280:-1"})

// Multiple outputs
out1 := processed0.Output("hd.mp4", ffmpeg.KwArgs{"b:v": "5000k"})
out2 := processed1.Output("sd.mp4", ffmpeg.KwArgs{"b:v": "2000k"})

// Execute both outputs
err := ffmpeg.MergeOutputs(out1, out2).OverWriteOutput().Run()
```

## Advanced Patterns

### 1. Concatenation

```go
// Video concatenation with filters
clip1 := ffmpeg.Input("clip1.mp4")
clip2 := ffmpeg.Input("clip2.mp4") 
clip3 := ffmpeg.Input("clip3.mp4")

result := ffmpeg.Concat([]*ffmpeg.Stream{clip1, clip2, clip3}, ffmpeg.KwArgs{
    "v": 1, // Video streams
    "a": 1, // Audio streams  
})

result.Output("combined.mp4").Run()
```

### 2. Format Conversion

```go
// Transcode between formats
ffmpeg.Input("input.avi").
    Output("output.mp4", ffmpeg.KwArgs{
        "c:v":     "libx264",  // Video codec
        "c:a":     "aac",      // Audio codec
        "b:v":     "2000k",    // Video bitrate
        "b:a":     "128k",     // Audio bitrate
        "preset":  "medium",   // Encoding speed
        "crf":     "23",       // Quality (lower = better)
    }).
    OverWriteOutput().
    Run()
```

### 3. Live Streaming

```go
// Stream to RTMP server
ffmpeg.Input("input.mp4").
    Output("rtmp://server/live/stream", ffmpeg.KwArgs{
        "c:v":      "libx264",
        "c:a":      "aac", 
        "preset":   "veryfast",
        "tune":     "zerolatency",
        "f":        "flv",
    }).
    Run()

// Capture from camera/screen
ffmpeg.Input("0", ffmpeg.KwArgs{  // Camera device
        "f":        "dshow",       // DirectShow (Windows)
        "video_size": "1920x1080",
        "framerate": "30",
    }).
    Output("stream.mp4").
    Run()
```

### 4. Frame Extraction

```go
// Extract specific frame
func extractFrame(videoPath string, frameNumber int) (image.Image, error) {
    buf := bytes.NewBuffer(nil)
    
    err := ffmpeg.Input(videoPath).
        Filter("select", ffmpeg.Args{fmt.Sprintf("gte(n,%d)", frameNumber)}).
        Output("pipe:", ffmpeg.KwArgs{
            "vframes": 1,
            "format":  "image2",
            "vcodec":  "mjpeg",
        }).
        WithOutput(buf).
        Run()
    
    if err != nil {
        return nil, err
    }
    
    return jpeg.Decode(buf)
}

// Extract thumbnail at time position
func extractThumbnail(videoPath string, timePos string) (image.Image, error) {
    buf := bytes.NewBuffer(nil)
    
    err := ffmpeg.Input(videoPath, ffmpeg.KwArgs{"ss": timePos}).
        Output("pipe:", ffmpeg.KwArgs{
            "vframes": 1,
            "format":  "image2", 
            "vcodec":  "mjpeg",
        }).
        WithOutput(buf).
        Run()
    
    if err != nil {
        return nil, err
    }
    
    return jpeg.Decode(buf)
}
```

### 5. Progress Monitoring

```go
func convertWithProgress(input, output string) error {
    // Get total duration first
    probeData, err := ffmpeg.Probe(input)
    if err != nil {
        return err
    }
    
    totalDuration := gjson.Get(probeData, "format.duration").Float()
    
    // Create progress socket
    progressSock := createProgressSocket() // Your implementation
    
    // Start conversion with progress reporting
    err = ffmpeg.Input(input).
        Output(output, ffmpeg.KwArgs{"c:v": "libx264"}).
        GlobalArgs("-progress", "unix://"+progressSock).
        OverWriteOutput().
        Run()
    
    return err
}
```

## Error Handling and Debugging

### 1. Command Inspection

```go
// Enable command logging
ffmpeg.LogCompiledCommand = true

// Or use Silent() to disable
stream := ffmpeg.Input("input.mp4").Silent(false)

// Get generated arguments without execution
args := stream.GetArgs()
fmt.Printf("Would execute: ffmpeg %s\n", strings.Join(args, " "))
```

### 2. Error Capture

```go
// Capture stderr for debugging
var errBuf bytes.Buffer

err := ffmpeg.Input("input.mp4").
    Output("output.mp4").
    WithErrorOutput(&errBuf).
    Run()

if err != nil {
    fmt.Printf("FFmpeg error: %v\n", err)
    fmt.Printf("FFmpeg stderr: %s\n", errBuf.String())
}
```

### 3. Validation

```go
// Check if input file is valid
func validateMedia(filename string) error {
    _, err := ffmpeg.Probe(filename)
    return err
}

// Get media information
func getMediaInfo(filename string) (map[string]interface{}, error) {
    data, err := ffmpeg.Probe(filename)
    if err != nil {
        return nil, err
    }
    
    var info map[string]interface{}
    err = json.Unmarshal([]byte(data), &info)
    return info, err
}
```

## Performance and Best Practices

### 1. Resource Management

```go
// Set timeouts to prevent hanging
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
defer cancel()

stream := ffmpeg.Input("input.mp4")
stream.Context = ctx
err := stream.Output("output.mp4").Run()
```

### 2. Efficient Processing

```go
// Use hardware acceleration when available
ffmpeg.Input("input.mp4").
    Output("output.mp4", ffmpeg.KwArgs{
        "c:v":        "h264_nvenc",  // NVIDIA GPU
        "preset":     "fast",
        "b:v":        "2000k",
    }).Run()

// Optimize for specific use cases
ffmpeg.Input("input.mp4").
    Output("output.mp4", ffmpeg.KwArgs{
        "c:v":     "libx264",
        "preset":  "ultrafast",  // Fast encoding
        "tune":    "fastdecode", // Fast decoding
    }).Run()
```

### 3. Memory Optimization

```go
// Process large files in chunks
func processLargeVideo(inputPath, outputPath string) error {
    // Get video duration
    probe, err := ffmpeg.Probe(inputPath)
    if err != nil {
        return err
    }
    
    duration := gjson.Get(probe, "format.duration").Float()
    chunkSize := 60.0 // 60 seconds per chunk
    
    var chunks []*ffmpeg.Stream
    for start := 0.0; start < duration; start += chunkSize {
        end := math.Min(start+chunkSize, duration)
        
        chunk := ffmpeg.Input(inputPath, ffmpeg.KwArgs{
            "ss": fmt.Sprintf("%.2f", start),
            "t":  fmt.Sprintf("%.2f", end-start),
        })
        
        chunks = append(chunks, chunk)
    }
    
    // Concatenate processed chunks
    return ffmpeg.Concat(chunks).Output(outputPath).Run()
}
```

### 4. Parallel Processing

```go
// Process multiple files concurrently
func processFiles(inputFiles []string, outputDir string) error {
    var wg sync.WaitGroup
    errChan := make(chan error, len(inputFiles))
    
    for _, inputFile := range inputFiles {
        wg.Add(1)
        go func(input string) {
            defer wg.Done()
            
            output := filepath.Join(outputDir, 
                strings.TrimSuffix(filepath.Base(input), filepath.Ext(input))+".mp4")
                
            err := ffmpeg.Input(input).
                Output(output, ffmpeg.KwArgs{"c:v": "libx264"}).
                OverWriteOutput().
                Run()
                
            if err != nil {
                errChan <- err
            }
        }(inputFile)
    }
    
    wg.Wait()
    close(errChan)
    
    // Check for any errors
    for err := range errChan {
        if err != nil {
            return err
        }
    }
    
    return nil
}
```

This guide covers the most common usage patterns and demonstrates how FFmpeg-Go simplifies complex multimedia processing tasks while maintaining the full power of FFmpeg.