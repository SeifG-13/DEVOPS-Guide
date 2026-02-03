# CMD vs ENTRYPOINT

## Overview

Both CMD and ENTRYPOINT define what command runs when a container starts, but they behave differently.

```
┌─────────────────────────────────────────────────────────────────┐
│                    CMD vs ENTRYPOINT                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   CMD                              ENTRYPOINT                    │
│   ┌────────────────────┐          ┌────────────────────┐       │
│   │ Default command    │          │ Fixed command      │       │
│   │ Can be overridden  │          │ Cannot be easily   │       │
│   │ by docker run      │          │ overridden         │       │
│   └────────────────────┘          └────────────────────┘       │
│                                                                  │
│   docker run image cmd            docker run image              │
│        └─── replaces CMD                └─── args to ENTRYPOINT │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## CMD Instruction

CMD provides default arguments for the container.

### Three Forms

```dockerfile
# Exec form (preferred)
CMD ["executable", "param1", "param2"]

# Exec form as parameters to ENTRYPOINT
CMD ["param1", "param2"]

# Shell form
CMD command param1 param2
```

### CMD Examples

```dockerfile
# Exec form
FROM ubuntu
CMD ["echo", "Hello World"]

# Shell form
FROM ubuntu
CMD echo "Hello World"

# Shell form runs through shell
# Equivalent to: /bin/sh -c "echo Hello World"
```

### Overriding CMD

```bash
# Dockerfile
# CMD ["echo", "Hello"]

# Default behavior
docker run myimage
# Output: Hello

# Override CMD
docker run myimage echo "Goodbye"
# Output: Goodbye

docker run myimage /bin/bash
# Starts bash shell
```

## ENTRYPOINT Instruction

ENTRYPOINT configures the container as an executable.

### Two Forms

```dockerfile
# Exec form (preferred)
ENTRYPOINT ["executable", "param1"]

# Shell form
ENTRYPOINT command param1
```

### ENTRYPOINT Examples

```dockerfile
# Exec form
FROM ubuntu
ENTRYPOINT ["echo", "Hello"]

# Shell form
FROM ubuntu
ENTRYPOINT echo "Hello"
```

### Arguments to ENTRYPOINT

```bash
# Dockerfile
# ENTRYPOINT ["echo", "Hello"]

# Default behavior
docker run myimage
# Output: Hello

# Add arguments
docker run myimage World
# Output: Hello World

# Arguments are appended to ENTRYPOINT
```

## Combining CMD and ENTRYPOINT

The most powerful pattern: ENTRYPOINT for the command, CMD for default arguments.

```dockerfile
FROM ubuntu

ENTRYPOINT ["echo"]
CMD ["Hello World"]
```

```bash
# Default behavior
docker run myimage
# Output: Hello World
# Executes: echo Hello World

# Override default arguments
docker run myimage "Goodbye World"
# Output: Goodbye World
# Executes: echo Goodbye World
```

### Practical Example: curl

```dockerfile
FROM alpine
RUN apk add --no-cache curl
ENTRYPOINT ["curl"]
CMD ["--help"]
```

```bash
# Show help (default)
docker run mycurl
# Output: curl help

# Fetch URL
docker run mycurl https://example.com
# Fetches example.com

# With options
docker run mycurl -I https://example.com
# Shows headers
```

## Exec Form vs Shell Form

### Exec Form

```dockerfile
# Runs directly without shell
ENTRYPOINT ["nginx", "-g", "daemon off;"]
CMD ["echo", "Hello"]

# PID 1 is the actual process
# Signals are received correctly
# No shell variable expansion
```

### Shell Form

```dockerfile
# Runs through /bin/sh -c
ENTRYPOINT nginx -g "daemon off;"
CMD echo "Hello"

# PID 1 is /bin/sh
# Command is child of shell
# Shell variable expansion works
```

### Signal Handling Comparison

```
Exec Form:                    Shell Form:
┌─────────────┐              ┌─────────────┐
│ Container   │              │ Container   │
├─────────────┤              ├─────────────┤
│ PID 1: nginx│◄── Signals   │ PID 1: sh   │◄── Signals
└─────────────┘              │  └─ nginx   │
                             └─────────────┘
                             (signals may not reach nginx)
```

```dockerfile
# Good - exec form
ENTRYPOINT ["nginx", "-g", "daemon off;"]

# Bad - shell form (signals don't reach nginx)
ENTRYPOINT nginx -g "daemon off;"
```

## Override Behavior Summary

| Dockerfile | docker run | Result |
|------------|-----------|--------|
| CMD ["a"] | - | a |
| CMD ["a"] | b | b |
| ENTRYPOINT ["a"] | - | a |
| ENTRYPOINT ["a"] | b | a b |
| ENTRYPOINT ["a"] CMD ["b"] | - | a b |
| ENTRYPOINT ["a"] CMD ["b"] | c | a c |

## Overriding ENTRYPOINT

```bash
# Override ENTRYPOINT with --entrypoint
docker run --entrypoint /bin/bash myimage

# Override both
docker run --entrypoint /bin/sh myimage -c "echo Hello"
```

## Use Cases

### When to Use CMD

```dockerfile
# Default command that should be easily overridable
FROM python:3.11
WORKDIR /app
COPY . .
CMD ["python", "app.py"]
# Users can: docker run myimage python other_script.py
```

### When to Use ENTRYPOINT

```dockerfile
# Container should always run specific executable
FROM alpine
RUN apk add --no-cache curl
ENTRYPOINT ["curl"]
CMD ["-s"]
# Container acts as curl command
```

### When to Use Both

```dockerfile
# Fixed command with default arguments
FROM postgres:15
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["postgres"]
# Entry point script handles initialization
# CMD provides default command to run
```

## Wrapper Scripts

Common pattern using both instructions with a wrapper script.

### entrypoint.sh

```bash
#!/bin/bash
set -e

# Initialization tasks
echo "Starting application..."

# Handle signals
trap 'echo "Shutting down..."; exit 0' SIGTERM SIGINT

# Run the main command
exec "$@"
```

### Dockerfile

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["node", "server.js"]
```

### Benefits

```bash
# Default behavior
docker run myapp
# Runs entrypoint.sh node server.js

# Custom command
docker run myapp npm test
# Runs entrypoint.sh npm test

# Shell access
docker run -it myapp /bin/sh
# Runs entrypoint.sh /bin/sh
```

## Common Patterns

### Database Container

```dockerfile
FROM postgres:15
COPY init.sql /docker-entrypoint-initdb.d/
# Uses postgres ENTRYPOINT for initialization
# CMD ["postgres"] is inherited
```

### Web Server

```dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY dist/ /usr/share/nginx/html/
# Inherits ENTRYPOINT and CMD from nginx image
# ENTRYPOINT ["/docker-entrypoint.sh"]
# CMD ["nginx", "-g", "daemon off;"]
```

### CLI Tool

```dockerfile
FROM alpine:3.18
RUN apk add --no-cache aws-cli
ENTRYPOINT ["aws"]
CMD ["help"]
# docker run myaws s3 ls
```

## Quick Reference

| Aspect | CMD | ENTRYPOINT |
|--------|-----|------------|
| Purpose | Default arguments | Main command |
| Override | `docker run image cmd` | `--entrypoint` |
| Multiple | Last one wins | Last one wins |
| With both | CMD args go to ENTRYPOINT | Receives CMD as args |
| Preferred form | Exec form | Exec form |

### Best Practices

1. **Use exec form** for both CMD and ENTRYPOINT
2. **ENTRYPOINT** for the main executable
3. **CMD** for default arguments
4. **Wrapper scripts** for initialization logic
5. **Signal handling** requires exec form

---

**Previous:** [06-dockerfile.md](06-dockerfile.md) | **Next:** [08-environment-variables.md](08-environment-variables.md)
