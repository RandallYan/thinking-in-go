# 6 Common Operations with Go Modules

Go Modules is the official dependency management tool for the Go programming language. This article will detail six common operations with Go Modules, including adding dependencies, upgrading/downgrading dependency versions, handling dependencies with major version numbers greater than 1, upgrading to incompatible versions, removing dependencies, and using the vendor mechanism. By mastering these operations, you'll be better equipped to manage dependencies in your Go projects.

## 0. Adding a Dependency to the Current Module

To add a new dependency to your project, you can use the `go get` command:

```bash
go get github.com/example/package
```

This command will download the latest version of the package and add it to your `go.mod` file.

Alternatively, you can directly import the package in your code and then run:

```bash
go mod tidy
```

The `go mod tidy` command will automatically analyze your code, download necessary dependencies, and update the `go.mod` and `go.sum` files.

Practical example:

```go
package main

import (
    "fmt"
    "github.com/fatih/color"
)

func main() {
    color.Blue("This is a blue message")
}
```

After running `go mod tidy`, the `go.mod` file will be automatically updated:

```
module myproject

go 1.16

require github.com/fatih/color v1.13.0
```

## 1. Upgrading/Downgrading Dependency Versions

To upgrade a dependency to the latest version, use:

```bash
go get -u github.com/example/package
```

To downgrade to a specific version, specify the version number:

```bash
go get github.com/example/package@v1.2.3
```

You can also use comparison operators:

```bash
go get github.com/example/package@'<v1.2.3'
```

This will install the latest version that's less than v1.2.3.

## 2. Adding a Dependency with Major Version Number Greater than 1

Go's module versioning follows semantic versioning principles. When the major version number is greater than 1, the import path needs to include the major version number:

```go
import "github.com/example/package/v2"
```

Then run `go mod tidy` to download and update the dependency.

Practical example:

```go
package main

import (
    "fmt"
    "gopkg.in/yaml.v2"
)

func main() {
    data := []byte(`
a: Easy!
b:
  c: 2
  d: [3, 4]
`)
    
    m := make(map[interface{}]interface{})
    
    err := yaml.Unmarshal(data, &m)
    if err != nil {
        fmt.Printf("error: %v", err)
    }
    fmt.Printf("--- m:\n%v\n\n", m)
}
```

## 3. Upgrading a Dependency to an Incompatible Version

When you need to upgrade to an incompatible version (usually a change in the major version number), you need to change the import path:

```go
// From
import "github.com/example/package"
// To
import "github.com/example/package/v2"
```

Then run `go mod tidy` to update the dependency.

Note: This may require corresponding modifications to your code, as the new version might have breaking changes.

## 4. Removing a Dependency

To remove a dependency that's no longer in use, first delete all imports of that package from your code, then run:

```bash
go mod tidy
```

This command will automatically remove unused dependencies from the `go.mod` and `go.sum` files.

You can also manually edit the `go.mod` file, delete the line for the unwanted dependency, and then run `go mod tidy`.

## 5. Special Case: Using Vendor

Vendor is a mechanism for storing project dependencies directly in the project directory. To use vendor, run:

```bash
go mod vendor
```

This will create a `vendor` directory containing all dependency packages.

To build the project using vendor, use:

```bash
go build -mod=vendor
```

The advantage of using vendor is ensuring build reproducibility, while the disadvantage is increased repository size.

## Summary

By mastering these common operations with Go Modules, you can more effectively manage dependencies in your Go projects. Here are some best practices:

1. Run `go mod tidy` frequently to keep dependencies clean and up-to-date.
2. Ensure that both `go.mod` and `go.sum` files are updated and committed before pushing your code.
3. Use semantic versioning to manage your own packages.
4. Be cautious when upgrading dependencies, especially with major version changes, as they may introduce breaking changes.
5. Consider using the `go mod why` command to understand why a specific dependency is needed.

By following these operations and best practices, you'll be better equipped to manage dependencies in Go projects, improving development efficiency and code quality.
