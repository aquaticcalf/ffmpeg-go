# FFmpeg-Go Architecture: How Go Interacts with FFmpeg

## Overview

FFmpeg-Go is a Go library that provides a fluent, chainable API for building and executing FFmpeg commands. Rather than working with raw command-line strings, it allows developers to construct complex multimedia processing pipelines using Go's type system and method chaining.

## Core Architecture

### 1. Stream-Based Design

The library's foundation is built around **Streams** - objects that represent data flows in multimedia processing pipelines. Each Stream corresponds to a multimedia stream (video, audio, or data) that can be processed by FFmpeg.

```go
type Stream struct {
    Node       *Node           // The processing node this stream represents
    Label      Label           // Stream identifier within the node  
    Selector   Selector        // Stream selector (e.g., "v" for video, "a" for audio)
    Type       string          // Stream type ("FilterableStream", "OutputStream")
    FfmpegPath string          // Path to ffmpeg executable
    Context    context.Context // Execution context
}
```

### 2. Node-Based Graph Structure

Under the hood, FFmpeg-Go uses a **Directed Acyclic Graph (DAG)** structure where each **Node** represents an operation:

```go
type Node struct {
    streamSpec          []*Stream      // Input streams
    name                string         // Operation name (e.g., "input", "filter", "output")
    incomingStreamTypes sets.String    // Allowed input stream types
    outgoingStreamType  string         // Output stream type
    minInputs           int            // Minimum required inputs
    maxInputs           int            // Maximum allowed inputs  
    args                []string       // Positional arguments
    kwargs              KwArgs         // Keyword arguments
    nodeType            string         // Node classification
}
```

#### Node Types:
- **InputNode**: Represents FFmpeg input sources (`-i` flag)
- **FilterNode**: Represents FFmpeg filters (`-filter_complex`)  
- **OutputNode**: Represents FFmpeg outputs (output files)
- **GlobalNode**: Represents global FFmpeg options (`-y`, `-progress`, etc.)
- **MergeOutputsNode**: Combines multiple outputs into single command

### 3. Command Generation Pipeline

The library transforms the DAG into FFmpeg command-line arguments through these steps:

1. **Topological Sort**: Orders nodes for proper execution sequence
2. **Stream Name Allocation**: Assigns temporary stream labels for inter-filter communication
3. **Argument Assembly**: Converts nodes to their respective FFmpeg arguments
4. **Command Compilation**: Combines all arguments into final FFmpeg command

## How Go Interacts with FFmpeg

### 1. Process Execution

FFmpeg-Go executes FFmpeg as an external process using Go's `os/exec` package:

```go
func (s *Stream) Run(options ...CompilationOption) error {
    return s.Compile(options...).Run()
}

func (s *Stream) Compile(options ...CompilationOption) *exec.Cmd {
    args := s.GetArgs()  // Generate FFmpeg arguments
    cmd := exec.CommandContext(s.Context, s.FfmpegPath, args...)
    
    // Set up I/O streams
    if stdin, ok := s.Context.Value("Stdin").(io.Reader); ok {
        cmd.Stdin = stdin
    }
    if stdout, ok := s.Context.Value("Stdout").(io.Writer); ok {
        cmd.Stdout = stdout  
    }
    if stderr, ok := s.Context.Value("Stderr").(io.Writer); ok {
        cmd.Stderr = stderr
    }
    
    return cmd
}
```

### 2. Argument Generation

The `GetArgs()` method is the heart of command generation:

```go
func (s *Stream) GetArgs() []string {
    var args []string
    
    // Process nodes in topological order
    nodes := getStreamSpecNodes([]*Stream{s})
    sorted, outGoingMap, _ := TopSort(nodes)
    
    // Categorize nodes by type
    var inputNodes, outputNodes, globalNodes, filterNodes []*Node
    for _, node := range sorted {
        switch node.nodeType {
        case "InputNode":
            inputNodes = append(inputNodes, node)
        case "OutputNode": 
            outputNodes = append(outputNodes, node)
        case "GlobalNode":
            globalNodes = append(globalNodes, node)
        case "FilterNode":
            filterNodes = append(filterNodes, node)
        }
    }
    
    // Generate arguments for each category
    for _, n := range inputNodes {
        args = append(args, getInputArgs(n)...)  // -i inputs
    }
    
    filterArgs := _getFilterArg(filterNodes, outGoingMap, streamNameMap)
    if filterArgs != "" {
        args = append(args, "-filter_complex", filterArgs)  // Filters
    }
    
    for _, n := range outputNodes {
        args = append(args, _getOutputArgs(n, streamNameMap)...)  // Outputs
    }
    
    for _, n := range globalNodes {
        args = append(args, _getGlobalArgs(n)...)  // Global options
    }
    
    return args
}
```

### 3. Input Processing

Input nodes generate `-i` arguments:

```go
func getInputArgs(node *Node) []string {
    var args []string
    kwargs := node.kwargs.Copy()
    
    filename := kwargs.PopString("filename")
    format := kwargs.PopString("format")
    
    if format != "" {
        args = append(args, "-f", format)  // Input format
    }
    
    args = append(args, ConvertKwargsToCmdLineArgs(kwargs)...)
    args = append(args, "-i", filename)  // Input file
    
    return args
}
```

### 4. Filter Complex Generation

The library generates FFmpeg's `-filter_complex` syntax by:

1. **Stream Naming**: Assigning temporary labels like `[s0]`, `[s1]` for intermediate streams
2. **Filter Specification**: Building filter strings with input/output mappings
3. **Chain Assembly**: Connecting filters with semicolon separators

```go
func _getFilterSpec(node *Node, outEdges map[Label][]NodeInfo, streamNameMap map[string]string) string {
    var input, output []string
    
    // Input stream references
    for _, edge := range node.GetInComingEdges() {
        input = append(input, formatInputStreamName(streamNameMap, edge, false))
    }
    
    // Output stream labels  
    for _, edge := range outEdges {
        output = append(output, formatOutStreamName(streamNameMap, edge))
    }
    
    // Combine: [input1][input2]filter=param[output1][output2]
    return fmt.Sprintf("%s%s%s", 
        strings.Join(input, ""), 
        node.GetFilter(outEdges), 
        strings.Join(output, ""))
}
```

### 5. I/O Stream Handling

FFmpeg-Go supports various I/O patterns:

#### Pipe Input/Output
```go
// Reading from stdin
stream := ffmpeg.Input("pipe:")

// Writing to stdout  
stream.Output("pipe:")

// Custom I/O streams
stream.WithInput(reader).WithOutput(writer)
```

#### Memory Buffers
```go
buf := bytes.NewBuffer(nil)
err := ffmpeg.Input("input.mp4").
    Output("pipe:", ffmpeg.KwArgs{"format": "mp4"}).
    WithOutput(buf).
    Run()
```

### 6. Context and Cancellation

The library integrates with Go's context system for:

- **Timeouts**: Automatic process termination
- **Cancellation**: Clean shutdown on context cancellation  
- **Metadata**: Passing execution options via context values

```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

stream.Context = ctx
err := stream.Run()
```

## Example: Complete Processing Flow

Here's how a simple transcoding operation flows through the system:

```go
err := ffmpeg.Input("input.mp4").
    Output("output.mp4", ffmpeg.KwArgs{"c:v": "libx264"}).
    OverWriteOutput().
    Run()
```

### Internal Processing:

1. **Graph Construction**:
   ```
   InputNode("input.mp4") → OutputNode("output.mp4", codec=libx264)
   ```

2. **Argument Generation**:
   ```bash
   ffmpeg -i input.mp4 -c:v libx264 -y output.mp4
   ```

3. **Process Execution**:
   ```go
   cmd := exec.Command("ffmpeg", "-i", "input.mp4", "-c:v", "libx264", "-y", "output.mp4")
   err := cmd.Run()
   ```

## Advanced Features

### 1. Complex Filter Graphs

The library can generate sophisticated filter graphs:

```go
split := ffmpeg.Input("input.mp4").Split()
left := split.Get("0").Filter("crop", ffmpeg.Args{"iw/2:ih:0:0"})  
right := split.Get("1").Filter("crop", ffmpeg.Args{"iw/2:ih:iw/2:0"})
output := ffmpeg.Filter([]*ffmpeg.Stream{left, right}, "hstack", nil)
```

Generates:
```bash
ffmpeg -i input.mp4 -filter_complex "[0]split[s0][s1];[s0]crop=iw/2:ih:0:0[s2];[s1]crop=iw/2:ih:iw/2:0[s3];[s2][s3]hstack[s4]" -map "[s4]" output.mp4
```

### 2. Multi-Output Processing

```go
input := ffmpeg.Input("input.mp4").Split()
hd := input.Get("0").Output("hd.mp4", ffmpeg.KwArgs{"b:v": "5000k"})
sd := input.Get("1").Output("sd.mp4", ffmpeg.KwArgs{"b:v": "1000k"})
err := ffmpeg.MergeOutputs(hd, sd).Run()
```

### 3. Progress Monitoring

```go
err := ffmpeg.Input("input.mp4").
    Output("output.mp4").
    GlobalArgs("-progress", "unix://"+progressSocket).
    Run()
```

## Key Benefits

1. **Type Safety**: Go's type system catches errors at compile time
2. **Composability**: Fluent API enables complex pipeline construction
3. **Resource Management**: Automatic process lifecycle management
4. **Flexibility**: Support for pipes, custom I/O, and advanced FFmpeg features
5. **Debugging**: Built-in command logging and error handling

This architecture makes FFmpeg accessible to Go developers while maintaining the full power and flexibility of the underlying FFmpeg tool.