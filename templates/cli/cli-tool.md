# Go CLI Tool Development

Create a robust command-line interface tool using Go with proper structure, configuration, and best practices.

## Usage

```bash
cd $ARGUMENTS  # Navigate to target directory
```

## Project Setup

```bash
# Initialize Go module
go mod init <module-name>

# Install common CLI dependencies
go get github.com/spf13/cobra@latest
go get github.com/spf13/viper@latest
go get github.com/spf13/pflag@latest
```

## Project Structure

```
cli-tool/
├── cmd/
│   ├── root.go          # Root command
│   ├── serve.go         # Subcommands
│   └── version.go       # Version command
├── pkg/
│   ├── config/          # Configuration handling
│   ├── logger/          # Logging utilities
│   └── utils/           # Utility functions
├── internal/
│   ├── api/             # Internal API logic
│   └── storage/         # Data storage
├── configs/
│   └── config.yaml      # Default configuration
├── .goreleaser.yml      # Release configuration
├── Makefile            # Build commands
├── go.mod
├── go.sum
└── main.go             # Application entry point
```

## Main Entry Point (main.go)

```go
package main

import (
    "os"
    "github.com/your-org/cli-tool/cmd"
)

func main() {
    if err := cmd.Execute(); err != nil {
        os.Exit(1)
    }
}
```

## Root Command (cmd/root.go)

```go
package cmd

import (
    "fmt"
    "os"
    "github.com/spf13/cobra"
    "github.com/spf13/viper"
)

var (
    cfgFile string
    verbose bool
)

// rootCmd represents the base command when called without any subcommands
var rootCmd = &cobra.Command{
    Use:   "cli-tool",
    Short: "A brief description of your application",
    Long:  `A longer description of your CLI tool...`,
    PersistentPreRun: func(cmd *cobra.Command, args []string) {
        initConfig()
        if verbose {
            fmt.Println("Verbose mode enabled")
        }
    },
}

// Execute adds all child commands to the root command and sets flags appropriately.
func Execute() error {
    return rootCmd.Execute()
}

func init() {
    cobra.OnInitialize(initConfig)

    // Global flags
    rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cli-tool.yaml)")
    rootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "verbose output")
    
    // Bind flags to viper
    viper.BindPFlag("verbose", rootCmd.PersistentFlags().Lookup("verbose"))
}

// initConfig reads in config file and ENV variables if set.
func initConfig() {
    if cfgFile != "" {
        viper.SetConfigFile(cfgFile)
    } else {
        home, err := os.UserHomeDir()
        cobra.CheckErr(err)

        viper.AddConfigPath(home)
        viper.AddConfigPath(".")
        viper.SetConfigType("yaml")
        viper.SetConfigName(".cli-tool")
    }

    viper.AutomaticEnv()

    if err := viper.ReadInConfig(); err == nil && verbose {
        fmt.Fprintln(os.Stderr, "Using config file:", viper.ConfigFileUsed())
    }
}
```

## Subcommand Example (cmd/serve.go)

```go
package cmd

import (
    "fmt"
    "net/http"
    "time"
    
    "github.com/spf13/cobra"
    "github.com/spf13/viper"
)

var (
    port int
    host string
)

var serveCmd = &cobra.Command{
    Use:   "serve",
    Short: "Start the server",
    Long:  `Start the HTTP server on the specified host and port.`,
    RunE: func(cmd *cobra.Command, args []string) error {
        return startServer()
    },
}

func init() {
    rootCmd.AddCommand(serveCmd)
    
    serveCmd.Flags().IntVarP(&port, "port", "p", 8080, "Port to listen on")
    serveCmd.Flags().StringVar(&host, "host", "localhost", "Host to listen on")
    
    viper.BindPFlag("server.port", serveCmd.Flags().Lookup("port"))
    viper.BindPFlag("server.host", serveCmd.Flags().Lookup("host"))
}

func startServer() error {
    addr := fmt.Sprintf("%s:%d", 
        viper.GetString("server.host"), 
        viper.GetInt("server.port"))
    
    fmt.Printf("Starting server on %s\n", addr)
    
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello from CLI tool!")
    })
    
    server := &http.Server{
        Addr:           addr,
        Handler:        mux,
        ReadTimeout:    10 * time.Second,
        WriteTimeout:   10 * time.Second,
        MaxHeaderBytes: 1 << 20,
    }
    
    return server.ListenAndServe()
}
```

## Version Command (cmd/version.go)

```go
package cmd

import (
    "fmt"
    "runtime"
    
    "github.com/spf13/cobra"
)

var (
    Version   = "dev"
    GitCommit = "unknown"
    BuildDate = "unknown"
)

var versionCmd = &cobra.Command{
    Use:   "version",
    Short: "Print version information",
    Run: func(cmd *cobra.Command, args []string) {
        fmt.Printf("Version:    %s\n", Version)
        fmt.Printf("Git commit: %s\n", GitCommit)
        fmt.Printf("Built:      %s\n", BuildDate)
        fmt.Printf("Go version: %s\n", runtime.Version())
        fmt.Printf("OS/Arch:    %s/%s\n", runtime.GOOS, runtime.GOARCH)
    },
}

func init() {
    rootCmd.AddCommand(versionCmd)
}
```

## Configuration Package (pkg/config/config.go)

```go
package config

import (
    "github.com/spf13/viper"
)

type Config struct {
    Server ServerConfig `mapstructure:"server"`
    Log    LogConfig    `mapstructure:"log"`
    DB     DBConfig     `mapstructure:"database"`
}

type ServerConfig struct {
    Host string `mapstructure:"host"`
    Port int    `mapstructure:"port"`
}

type LogConfig struct {
    Level  string `mapstructure:"level"`
    Format string `mapstructure:"format"`
}

type DBConfig struct {
    Host     string `mapstructure:"host"`
    Port     int    `mapstructure:"port"`
    Database string `mapstructure:"database"`
    Username string `mapstructure:"username"`
    Password string `mapstructure:"password"`
}

func Load() (*Config, error) {
    var config Config
    
    // Set defaults
    viper.SetDefault("server.host", "localhost")
    viper.SetDefault("server.port", 8080)
    viper.SetDefault("log.level", "info")
    viper.SetDefault("log.format", "text")
    
    if err := viper.Unmarshal(&config); err != nil {
        return nil, err
    }
    
    return &config, nil
}
```

## Logger Package (pkg/logger/logger.go)

```go
package logger

import (
    "os"
    "github.com/sirupsen/logrus"
)

type Logger struct {
    *logrus.Logger
}

func New(level, format string) *Logger {
    log := logrus.New()
    
    // Set log level
    switch level {
    case "debug":
        log.SetLevel(logrus.DebugLevel)
    case "info":
        log.SetLevel(logrus.InfoLevel)
    case "warn":
        log.SetLevel(logrus.WarnLevel)
    case "error":
        log.SetLevel(logrus.ErrorLevel)
    default:
        log.SetLevel(logrus.InfoLevel)
    }
    
    // Set format
    if format == "json" {
        log.SetFormatter(&logrus.JSONFormatter{})
    } else {
        log.SetFormatter(&logrus.TextFormatter{
            FullTimestamp: true,
        })
    }
    
    log.SetOutput(os.Stdout)
    
    return &Logger{log}
}
```

## Makefile

```makefile
# Build variables
BINARY_NAME=cli-tool
VERSION?=dev
GIT_COMMIT?=$(shell git rev-parse --short HEAD)
BUILD_DATE?=$(shell date -u '+%Y-%m-%d_%H:%M:%S')

# Go parameters
GOCMD=go
GOBUILD=$(GOCMD) build
GOCLEAN=$(GOCMD) clean
GOTEST=$(GOCMD) test
GOGET=$(GOCMD) get
GOMOD=$(GOCMD) mod

# Build flags
LDFLAGS=-ldflags "-X github.com/your-org/cli-tool/cmd.Version=$(VERSION) -X github.com/your-org/cli-tool/cmd.GitCommit=$(GIT_COMMIT) -X github.com/your-org/cli-tool/cmd.BuildDate=$(BUILD_DATE)"

.PHONY: all build clean test deps lint

all: test build

build:
	$(GOBUILD) $(LDFLAGS) -o $(BINARY_NAME) .

build-linux:
	GOOS=linux GOARCH=amd64 $(GOBUILD) $(LDFLAGS) -o $(BINARY_NAME)-linux .

build-windows:
	GOOS=windows GOARCH=amd64 $(GOBUILD) $(LDFLAGS) -o $(BINARY_NAME)-windows.exe .

build-darwin:
	GOOS=darwin GOARCH=amd64 $(GOBUILD) $(LDFLAGS) -o $(BINARY_NAME)-darwin .

clean:
	$(GOCLEAN)
	rm -f $(BINARY_NAME)*

test:
	$(GOTEST) -v ./...

test-coverage:
	$(GOTEST) -coverprofile=coverage.out ./...
	$(GOCMD) tool cover -html=coverage.out

deps:
	$(GOMOD) download
	$(GOMOD) tidy

lint:
	golangci-lint run

install:
	$(GOCMD) install $(LDFLAGS) .

run:
	$(GOBUILD) $(LDFLAGS) -o $(BINARY_NAME) . && ./$(BINARY_NAME)
```

## Configuration File (configs/config.yaml)

```yaml
server:
  host: "localhost"
  port: 8080

log:
  level: "info"
  format: "text"

database:
  host: "localhost"
  port: 5432
  database: "cli_tool"
  username: "user"
  password: "password"
```

## Release Configuration (.goreleaser.yml)

```yaml
before:
  hooks:
    - go mod tidy

builds:
  - env:
      - CGO_ENABLED=0
    goos:
      - linux
      - windows
      - darwin
    goarch:
      - amd64
      - arm64
    ldflags:
      - -s -w
      - -X github.com/your-org/cli-tool/cmd.Version={{.Version}}
      - -X github.com/your-org/cli-tool/cmd.GitCommit={{.ShortCommit}}
      - -X github.com/your-org/cli-tool/cmd.BuildDate={{.Date}}

archives:
  - replacements:
      darwin: Darwin
      linux: Linux
      windows: Windows
      386: i386
      amd64: x86_64

checksum:
  name_template: 'checksums.txt'

snapshot:
  name_template: "{{ incpatch .Version }}-next"

changelog:
  sort: asc
  filters:
    exclude:
      - '^docs:'
      - '^test:'
```

## Testing Example

```go
package cmd

import (
    "bytes"
    "testing"
    
    "github.com/spf13/cobra"
    "github.com/stretchr/testify/assert"
)

func TestRootCmd(t *testing.T) {
    cmd := &cobra.Command{
        Use: "test",
        Run: func(cmd *cobra.Command, args []string) {
            cmd.Print("test output")
        },
    }
    
    b := bytes.NewBufferString("")
    cmd.SetOut(b)
    cmd.Execute()
    
    assert.Equal(t, "test output", b.String())
}

func TestVersionCmd(t *testing.T) {
    b := bytes.NewBufferString("")
    versionCmd.SetOut(b)
    versionCmd.Execute()
    
    output := b.String()
    assert.Contains(t, output, "Version:")
    assert.Contains(t, output, "Git commit:")
    assert.Contains(t, output, "Built:")
}
```

## Best Practices

1. **Configuration Management**
   - Use Viper for flexible configuration
   - Support config files, environment variables, and flags
   - Provide sensible defaults

2. **Error Handling**
   - Return errors from functions, don't panic
   - Use wrapped errors for context
   - Provide helpful error messages

3. **Logging**
   - Use structured logging (JSON for production)
   - Include relevant context in log messages
   - Make log level configurable

4. **Testing**
   - Write unit tests for all functions
   - Test CLI commands with captured output
   - Use table-driven tests for multiple scenarios

5. **Building and Distribution**
   - Use GoReleaser for cross-platform builds
   - Include version information in binaries
   - Provide checksums for releases

6. **Documentation**
   - Use Cobra's help generation
   - Include examples in command descriptions
   - Maintain a comprehensive README

## Development Commands

```bash
# Install dependencies
make deps

# Run tests
make test

# Build binary
make build

# Run with hot reload (using air)
air

# Cross-compile for all platforms
make build-linux build-windows build-darwin

# Release (using goreleaser)
goreleaser release --rm-dist
```

## Common CLI Patterns

### Interactive Prompts
```go
import "github.com/manifoldco/promptui"

prompt := promptui.Prompt{
    Label: "Enter your name",
}

result, err := prompt.Run()
if err != nil {
    return err
}
```

### Progress Bars
```go
import "github.com/cheggaaa/pb/v3"

bar := pb.StartNew(100)
for i := 0; i < 100; i++ {
    bar.Increment()
    time.Sleep(time.Millisecond * 10)
}
bar.Finish()
```

### Colored Output
```go
import "github.com/fatih/color"

color.Red("Error: Something went wrong")
color.Green("Success: Operation completed")
color.Yellow("Warning: This is a warning")
```

This template provides a solid foundation for building robust CLI tools in Go with proper structure, configuration management, and best practices.