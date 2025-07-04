# Claude Go Templates

Go-specific templates for Claude Code development with Wails, CLI tools, and Go best practices.

## What's Included

### Wails Desktop Applications
- **Wails Setup**: Complete Wails desktop application initialization
- **Event Management**: Proper event system implementation
- **State Management**: Frontend-backend state synchronization

### CLI Tools
- **CLI Framework**: Command-line application structure
- **Configuration**: Config file and environment variable handling

### Web Services
- **REST API**: Go web service templates
- **Middleware**: Common middleware patterns

### Testing & Quality
- **Testing Framework**: Comprehensive testing setup
- **Performance Testing**: Benchmarking and profiling
- **Race Detection**: Concurrency testing templates

## Quick Start

```bash
# Add this repository to claude-init
claude-init repo add go-templates https://github.com/your-username/claude-go-templates

# Sync templates
claude-init sync go-templates

# List available templates
claude-init templates list --category wails

# Install a Wails template
claude-init templates install init-wails
```

## Template Categories

- **wails**: Wails desktop application templates
- **cli**: Command-line interface tools and utilities
- **web**: Web services, APIs, and HTTP applications
- **testing**: Testing frameworks and quality assurance

## Permission Sets

- **go-standard**: Standard Go development permissions
- **wails-development**: Wails-specific development permissions
- **go-testing**: Testing and quality assurance permissions

## Wails Best Practices

These templates follow Wails best practices:

- ✅ Event system cleanup with EventsOff
- ✅ Structured event naming (`module:action:target`)
- ✅ Proper context handling for graceful shutdowns
- ✅ sync.Map for concurrent data access
- ✅ Headless mode support for testing
- ✅ Error boundaries and proper error handling
- ✅ Platform-specific considerations

## Go Development Standards

- Lock-free concurrency patterns preferred
- Comprehensive error handling
- Proper resource cleanup
- Race condition prevention
- Performance monitoring
- Cross-platform compatibility

## Template Structure

```
templates/
└── category/
    ├── template-name.md      # Template implementation
    └── template-name.kdl     # Template metadata
```

## Contributing

1. Fork this repository
2. Follow Go best practices and conventions
3. Include comprehensive examples
4. Test on multiple platforms
5. Submit a pull request

### Guidelines

- Use idiomatic Go code
- Include proper error handling
- Provide testing examples
- Document platform differences
- Follow security best practices
- Include performance considerations

## License

MIT License - Use these templates to accelerate your Go development workflow.