# Go Wails Desktop App Setup

Initialize a complete Wails desktop application project with Go backend and modern web frontend.

## Usage

```bash
cd $ARGUMENTS  # Navigate to target directory
```

## Prerequisites

- Go 1.21+ installed
- Node.js 16+ installed
- Platform-specific dependencies:
  - **Linux**: `webkit2gtk-4.0-dev`, `build-essential`
  - **Windows**: WebView2 runtime (usually pre-installed)
  - **macOS**: Xcode command line tools

## Installation

```bash
# Install Wails CLI
go install github.com/wailsapp/wails/v2/cmd/wails@latest

# Verify installation
wails doctor
```

## Project Initialization

```bash
# Create new project with template
wails init -n my-wails-app -t react-ts

# Available templates:
# - react, react-ts
# - vue, vue-ts  
# - svelte, svelte-ts
# - preact, preact-ts
# - lit, lit-ts
# - vanilla, vanilla-ts
```

## Project Structure

```
my-wails-app/
   app/
      app.go              # Main application logic
      menu.go             # Application menu
   frontend/
      dist/               # Built frontend assets
      src/
         main.tsx        # Frontend entry point
         components/
   build/                  # Build configuration
   embed.go                # Asset embedding
   main.go                 # Application entry point
   wails.json              # Wails configuration
   go.mod
```

## Backend Implementation

### app/app.go
```go
package app

import (
    "context"
    "fmt"
    "os"
    "sync"
    "time"
)

// App struct
type App struct {
    ctx           context.Context
    // Use sync.Map for concurrent access
    data          *sync.Map
    // Background services
    ticker        *time.Ticker
    stopChan      chan bool
    // Configuration
    headlessMode  bool
    silentMode    bool
}

// NewApp creates a new App application struct
func NewApp() *App {
    return &App{
        data:     &sync.Map{},
        stopChan: make(chan bool),
    }
}

// Startup is called when the app starts. The context here
// can be used to detect when the app is closing.
func (a *App) Startup(ctx context.Context) {
    a.ctx = ctx
    a.startBackgroundServices()
}

// OnDomReady is called after front-end resources have been loaded
func (a *App) OnDomReady(ctx context.Context) {
    // Initialize frontend-specific resources
}

// OnBeforeClose is called when the application is about to quit,
// either by clicking the window close button or calling runtime.Quit.
func (a *App) OnBeforeClose(ctx context.Context) (prevent bool) {
    // Cleanup resources
    a.stopBackgroundServices()
    return false
}

// OnShutdown is called when the application is terminating
func (a *App) OnShutdown(ctx context.Context) {
    // Final cleanup
}

// Backend methods accessible from frontend
func (a *App) GetData(key string) (interface{}, bool) {
    return a.data.Load(key)
}

func (a *App) SetData(key string, value interface{}) {
    a.data.Store(key, value)
}

func (a *App) ProcessCommand(command string) (string, error) {
    // Implement command processing logic
    return fmt.Sprintf("Processed: %s", command), nil
}

// Background services
func (a *App) startBackgroundServices() {
    a.ticker = time.NewTicker(5 * time.Second)
    go func() {
        for {
            select {
            case <-a.ticker.C:
                // Background task
                a.performHealthCheck()
            case <-a.stopChan:
                return
            case <-a.ctx.Done():
                return
            }
        }
    }()
}

func (a *App) stopBackgroundServices() {
    if a.ticker != nil {
        a.ticker.Stop()
    }
    close(a.stopChan)
}

func (a *App) performHealthCheck() {
    // Emit events to frontend
    if !a.headlessMode {
        runtime.EventsEmit(a.ctx, "health-check", map[string]interface{}{
            "timestamp": time.Now(),
            "status":    "healthy",
        })
    }
}
```

## Development Commands

```bash
# Development mode with live reload
wails dev

# Build for production
wails build

# Build with debug info
wails build -debug

# Generate bindings
wails generate module

# Check environment
wails doctor
```

## Wails Best Practices

### 1. Event System Management
- Always clean up event listeners with EventsOff in useEffect cleanup
- Use structured event naming: `module:action:target`
- Check headlessMode before emitting events
- Never emit events from goroutines without proper context checking

### 2. State Management
- Use sync.Map for concurrent data access instead of mutexes
- Implement proper context handling for graceful shutdowns
- Use Zustand for frontend global state management
- Synchronize backend state changes through events, not polling

### 3. Testing Requirements
- Test all backend methods in headless mode for unit testing
- Mock Wails runtime functions in frontend tests
- Use EventCapture pattern for integration testing
- Test event cleanup to prevent memory leaks
- Include platform-specific test scenarios

### 4. Build & Development
- Always run `wails doctor` before troubleshooting build issues
- Use `wails dev` for development with live reload capabilities
- Test both GUI and headless modes before any release
- Embed assets properly with //go:embed directives
- Handle graceful shutdown in OnBeforeClose lifecycle method

### 5. Error Handling Standards
- Implement error boundaries for all major UI sections
- Return proper error types from all backend methods
- Log errors with context (user action, timestamp, application state)
- Provide user-friendly error messages with recovery options
- Never let errors crash the application silently

### 6. Performance Guidelines
- Use sync.Map for concurrent data access instead of mutexes
- Implement backpressure for high-frequency events
- Lazy load components with React.Suspense
- Debounce user input that triggers backend calls
- Monitor and limit concurrent goroutines properly

### 7. Platform Considerations
- Test on all target platforms (Windows, macOS, Linux)
- Handle platform-specific paths and commands with runtime.GOOS
- Use build tags for platform-specific code
- Consider WebView differences between platforms
- Test window management (minimize, maximize, close) behaviors

## Common Gotchas & Solutions

### 1. Context Management and Lifecycle
```go
// GOTCHA: Not properly handling context cancellation
// SOLUTION: Always check context in long-running operations
func (a *App) LongRunningOperation(ctx context.Context) error {
    for i := 0; i < 1000; i++ {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // Do work
            time.Sleep(100 * time.Millisecond)
        }
    }
    return nil
}
```

### 2. Event System Memory Leaks
```tsx
// GOTCHA: Not cleaning up event listeners
// SOLUTION: Always clean up in useEffect
useEffect(() => {
    const handleData = (data: any) => {
        setAppData(data);
    };

    EventsOn('data-update', handleData);

    // Cleanup function
    return () => {
        EventsOff('data-update');
    };
}, []);
```

### 3. Frontend-Backend Communication
```go
// GOTCHA: Not handling errors in backend methods
// SOLUTION: Always return proper error types
func (a *App) RiskyOperation() (string, error) {
    result, err := doSomethingRisky()
    if err != nil {
        return "", fmt.Errorf("operation failed: %w", err)
    }
    return result, nil
}
```

## Testing Framework

### Backend Testing (Headless Mode)
```go
// Test backend methods in headless mode
func TestAppHeadless(t *testing.T) {
    app := NewApp()
    app.headlessMode = true
    
    ctx := context.Background()
    app.Startup(ctx)
    
    // Test backend methods
    result, err := app.ProcessCommand("test")
    assert.NoError(t, err)
    assert.Equal(t, "Processed: test", result)
    
    // Cleanup
    app.OnShutdown(ctx)
}
```

### Frontend Testing with Mocks
```tsx
// Mock Wails runtime
jest.mock('../wailsjs/runtime/runtime', () => ({
    EventsOn: jest.fn(),
    EventsOff: jest.fn(),
}));

test('handles health check events', async () => {
    const mockEventsOn = EventsOn as jest.MockedFunction<typeof EventsOn>;
    
    let eventHandler: (data: any) => void;
    mockEventsOn.mockImplementation((eventName, handler) => {
        if (eventName === 'health-check') {
            eventHandler = handler;
        }
    });
    
    render(<App />);
    
    // Simulate event emission
    const healthData = { timestamp: '2025-01-01', status: 'healthy' };
    eventHandler!(healthData);
    
    await waitFor(() => {
        expect(screen.getByText(/Status: healthy/)).toBeInTheDocument();
    });
});
```

## Configuration Example (wails.json)

```json
{
    "$schema": "https://wails.io/schemas/config.v2.json",
    "name": "my-wails-app",
    "outputfilename": "my-wails-app",
    "frontend:install": "npm install",
    "frontend:build": "npm run build",
    "frontend:dev:watcher": "npm run dev",
    "frontend:dev:serverUrl": "auto",
    "author": {
        "name": "Your Name",
        "email": "your.email@example.com"
    },
    "info": {
        "productName": "My Wails App",
        "productVersion": "1.0.0",
        "copyright": "Copyright © 2025",
        "comments": "Built with Wails"
    }
}
```

## CLAUDE.md Integration

Add to project CLAUDE.md file:

```markdown
# Wails Project Guidelines

## Event System Rules
- ALWAYS clean up event listeners with EventsOff in useEffect cleanup functions
- Use structured event naming: `module:action:target` (e.g., `auth:status:changed`)  
- Emit events conditionally - check headlessMode before runtime.EventsEmit calls
- Never emit events from goroutines without proper context checking

## State Management Standards
- Use Zustand for global state management in React applications
- Keep Wails-specific logic isolated in custom hooks (like useWailsSync)
- Avoid prop drilling - use stores for data that crosses component boundaries
- Synchronize backend state changes through events, not polling

## Testing Requirements
- Test all backend methods in headless mode for unit testing
- Mock Wails runtime functions in frontend tests (EventsOn, EventsOff, etc.)
- Use EventCapture pattern for integration testing of event flows
- Test event cleanup to prevent memory leaks
- Include platform-specific test scenarios (Windows, macOS, Linux)

## Build & Development Workflow
- Always run `wails doctor` before troubleshooting build issues
- Use `wails dev` for development with live reload capabilities
- Test both GUI and headless modes before any release
- Embed assets properly with //go:embed directives
- Handle graceful shutdown in OnBeforeClose lifecycle method

## Error Handling Standards
- Implement error boundaries for all major UI sections
- Return proper error types from all backend methods
- Log errors with context (user action, timestamp, application state)
- Provide user-friendly error messages with recovery options
- Never let errors crash the application silently

## Performance Guidelines
- Use sync.Map for concurrent data access instead of mutexes
- Implement backpressure for high-frequency events
- Lazy load components with React.Suspense
- Debounce user input that triggers backend calls
- Monitor and limit concurrent goroutines properly

## Platform Considerations
- Test on all target platforms (Windows, macOS, Linux)
- Handle platform-specific paths and commands with runtime.GOOS
- Use build tags for platform-specific code
- Consider WebView differences between platforms
- Test window management (minimize, maximize, close) behaviors
```