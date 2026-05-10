# S5Fixture

## Overview
Fixture for S5 env-var detection tests. Configuration section documents
`DOCUMENTED_VAR` only; `UNDOCUMENTED_VAR` should fire S5.

## Architecture
Single-component test fixture.

## Features
- Test feature 1

## Configuration
- `DOCUMENTED_VAR` — documented test variable (default `none`)

## API Endpoints
### REST API
- `GET /test` — returns 200

## Project Structure
```
S5Fixture/
  src/
```

## Development
### Prerequisites
- bash
### Getting Started
- Run the tests.

## Deployment
- Image: `s5fixture:latest`

## Ports
| Service | Host | Container |
|---|---|---|
| HTTP | 9998 | 8080 |

## See Also
- (none)
