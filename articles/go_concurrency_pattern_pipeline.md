# Pipeline Pattern in Go: A Comprehensive Guide

## Overview

The pipeline pattern is a powerful concurrency pattern in Go that allows you to break down a complex operation into a series of stages, each performed by a separate goroutine. These stages are connected by channels, creating a "pipeline" through which data flows and is processed.

## Detailed Explanation

In a pipeline, each stage:
1. Receives data from upstream via inbound channels
2. Performs some operation on that data
3. Sends the result downstream via outbound channels

This pattern is particularly useful for operations that can be naturally divided into sequential steps, especially when dealing with streams of data or when you want to leverage parallelism for performance gains.

## Examples

Let's explore a few examples to illustrate the pipeline pattern in action.

### Example 1: Simple Number Processing Pipeline

```go
package main

import (
	"fmt"
)

func generator(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		for _, n := range nums {
			out <- n
		}
		close(out)
	}()
	return out
}

func square(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		for n := range in {
			out <- n * n
		}
		close(out)
	}()
	return out
}

func main() {
	// Set up the pipeline
	c := generator(1, 2, 3, 4)
	out := square(c)

	// Consume the output
	for n := range out {
		fmt.Println(n)
	}
}
```

This example demonstrates a simple two-stage pipeline:
1. The `generator` stage produces a series of numbers.
2. The `square` stage receives these numbers and squares them.

### Example 2: Text Processing Pipeline

Let's create a more practical example: a pipeline that processes text files.

```go
package main

import (
	"bufio"
	"fmt"
	"strings"
)

func lineGenerator(lines []string) <-chan string {
	out := make(chan string)
	go func() {
		for _, line := range lines {
			out <- line
		}
		close(out)
	}()
	return out
}

func toUpper(in <-chan string) <-chan string {
	out := make(chan string)
	go func() {
		for line := range in {
			out <- strings.ToUpper(line)
		}
		close(out)
	}()
	return out
}

func removeSpaces(in <-chan string) <-chan string {
	out := make(chan string)
	go func() {
		for line := range in {
			out <- strings.ReplaceAll(line, " ", "")
		}
		close(out)
	}()
	return out
}

func main() {
	// Sample input
	input := []string{
		"hello world",
		"go is awesome",
		"concurrent programming rocks",
	}

	// Set up the pipeline
	lines := lineGenerator(input)
	upper := toUpper(lines)
	noSpaces := removeSpaces(upper)

	// Consume the output
	for line := range noSpaces {
		fmt.Println(line)
	}
}
```

This pipeline has three stages:
1. `lineGenerator`: Produces lines from the input text.
2. `toUpper`: Converts each line to uppercase.
3. `removeSpaces`: Removes all spaces from each line.

## Use Cases

The pipeline pattern is particularly useful in scenarios such as:

1. **Data Processing**: When you need to apply a series of transformations to data, especially large datasets.
2. **Image Processing**: Applying multiple filters or transformations to images.
3. **Log Analysis**: Processing log files through multiple stages (parsing, filtering, aggregating).
4. **ETL (Extract, Transform, Load) Operations**: In data warehousing scenarios.
5. **Network Packet Processing**: Analyzing and transforming network packets.
6. **Compiler Design**: For various stages of compilation (lexing, parsing, code generation).

## Best Practices and Considerations

1. **Proper Channel Closure**: Ensure that each stage closes its output channel when it's done processing all input.

2. **Error Handling**: Implement robust error handling. Consider using a separate error channel or wrapping your data with error information.

3. **Backpressure**: Be mindful of the pipeline's overall throughput. If one stage is significantly slower, it can create backpressure in the system.

4. **Resource Management**: For resource-intensive operations, consider implementing worker pools within stages to manage concurrency levels.

5. **Testing**: Write unit tests for individual stages and integration tests for the entire pipeline.

6. **Monitoring and Observability**: Implement logging or metrics collection to monitor the performance and health of your pipeline.

7. **Cancellation**: Use context for graceful cancellation of the pipeline.

## Advanced Example: File Processing Pipeline with Error Handling and Cancellation

Let's create a more advanced example that incorporates error handling and cancellation:

```go
package main

import (
	"context"
	"fmt"
	"io/ioutil"
	"os"
	"path/filepath"
	"strings"
	"time"
)

type FileInfo struct {
	Path    string
	Content string
	Error   error
}

func walkFiles(ctx context.Context, root string) <-chan FileInfo {
	out := make(chan FileInfo)
	go func() {
		defer close(out)
		err := filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
			if err != nil {
				return err
			}
			if !info.Mode().IsRegular() {
				return nil
			}
			select {
			case out <- FileInfo{Path: path}:
			case <-ctx.Done():
				return ctx.Err()
			}
			return nil
		})
		if err != nil {
			out <- FileInfo{Error: err}
		}
	}()
	return out
}

func readContent(ctx context.Context, in <-chan FileInfo) <-chan FileInfo {
	out := make(chan FileInfo)
	go func() {
		defer close(out)
		for fi := range in {
			if fi.Error != nil {
				out <- fi
				continue
			}
			content, err := ioutil.ReadFile(fi.Path)
			select {
			case out <- FileInfo{
				Path:    fi.Path,
				Content: string(content),
				Error:   err,
			}:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}

func grep(ctx context.Context, in <-chan FileInfo, search string) <-chan FileInfo {
	out := make(chan FileInfo)
	go func() {
		defer close(out)
		for fi := range in {
			if fi.Error != nil {
				out <- fi
				continue
			}
			if strings.Contains(fi.Content, search) {
				select {
				case out <- fi:
				case <-ctx.Done():
					return
				}
			}
		}
	}()
	return out
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// Set up the pipeline
	files := walkFiles(ctx, ".")
	contents := readContent(ctx, files)
	matches := grep(ctx, contents, "TODO")

	// Consume the output
	for match := range matches {
		if match.Error != nil {
			fmt.Printf("Error: %v\n", match.Error)
			continue
		}
		fmt.Printf("Found match in file: %s\n", match.Path)
	}
}
```

This advanced example demonstrates:

1. **Error Handling**: Each stage propagates errors through the pipeline.
2. **Cancellation**: Uses `context` for timeout and cancellation.
3. **Practical Use Case**: Searches for "TODO" comments in files.
4. **Multiple Stages**: File walking, content reading, and text searching.

## Conclusion

The pipeline pattern in Go is a powerful tool for creating concurrent, efficient data processing systems. By breaking complex operations into stages connected by channels, we can create modular, scalable, and easy-to-reason-about code. Remember to handle errors appropriately, manage resources carefully, and consider the specific needs of your application when implementing pipelines.
