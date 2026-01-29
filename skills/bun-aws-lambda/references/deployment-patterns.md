# Deployment Patterns Reference

## Table of Contents
- [Runtime Options](#runtime-options)
- [Container Image Deployment](#container-image-deployment)
- [Custom Runtime with Lambda Layer](#custom-runtime-with-lambda-layer)
- [Build and Bundle](#build-and-bundle)
- [IaC Integration](#iac-integration)
- [Environment and Configuration](#environment-and-configuration)

## Runtime Options

### Decision Matrix

| Approach | Cold Start | Package Size | Complexity | Use Case |
|----------|------------|--------------|------------|----------|
| Container Image | ~300-500ms | Up to 10GB | Medium | Full control, large dependencies |
| Custom Runtime Layer | ~100-200ms | 50MB limit | Higher | Minimal size, fastest cold start |
| Node.js + Bun Build | ~100ms | 50MB limit | Low | Simplest, if Node.js compatible |

### Recommended: Container Image

Best balance of simplicity and performance for most Bun Lambda use cases.

## Container Image Deployment

### Directory Structure

```
my-lambda/
├── src/
│   └── handler.ts
├── Dockerfile
├── bunfig.toml (optional)
└── package.json
```

### Dockerfile

```dockerfile
# Use Bun's official image as builder
FROM oven/bun:1 AS builder

WORKDIR /app

# Copy package files
COPY package.json bun.lockb ./

# Install dependencies
RUN bun install --frozen-lockfile --production

# Copy source
COPY src/ ./src/

# Bundle to single file (optional but recommended)
RUN bun build src/handler.ts --outdir=dist --target=bun --minify

# Production image - use AWS Lambda base
FROM public.ecr.aws/lambda/provided:al2023

# Install Bun in Lambda image
RUN curl -fsSL https://bun.sh/install | bash
ENV PATH="/root/.bun/bin:${PATH}"

WORKDIR ${LAMBDA_TASK_ROOT}

# Copy bundled code or full node_modules
COPY --from=builder /app/dist/ ./
# Or if not bundling:
# COPY --from=builder /app/node_modules ./node_modules
# COPY --from=builder /app/src ./src

# Copy custom bootstrap
COPY bootstrap ${LAMBDA_RUNTIME_DIR}/bootstrap
RUN chmod +x ${LAMBDA_RUNTIME_DIR}/bootstrap

CMD ["handler.handler"]
```

### Bootstrap Script

```bash
#!/bin/bash
set -euo pipefail

# Lambda runtime API endpoint
RUNTIME_API="http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime"

# Handler format: filename.export
HANDLER_FILE="${_HANDLER%.*}"
HANDLER_FUNCTION="${_HANDLER##*.}"

# Main loop
while true; do
  # Get next invocation
  HEADERS=$(mktemp)
  EVENT=$(curl -sS -LD "$HEADERS" "${RUNTIME_API}/invocation/next")
  REQUEST_ID=$(grep -i "Lambda-Runtime-Aws-Request-Id" "$HEADERS" | tr -d '\r' | cut -d: -f2 | xargs)

  # Invoke handler with Bun
  RESPONSE=$(echo "$EVENT" | bun run --bun "${LAMBDA_TASK_ROOT}/${HANDLER_FILE}.js" "$HANDLER_FUNCTION" 2>&1) || {
    # Report error
    ERROR='{"errorMessage":"Handler error","errorType":"HandlerError"}'
    curl -sS -X POST "${RUNTIME_API}/invocation/${REQUEST_ID}/error" \
      -d "$ERROR" \
      -H "Content-Type: application/json"
    continue
  }

  # Report success
  curl -sS -X POST "${RUNTIME_API}/invocation/${REQUEST_ID}/response" \
    -d "$RESPONSE" \
    -H "Content-Type: application/json"

  rm -f "$HEADERS"
done
```

### Alternative: Integrated Bootstrap Handler

Instead of shell script, use Bun directly as the entrypoint:

```typescript
// bootstrap.ts - Custom runtime bootstrap in TypeScript
const RUNTIME_API = `http://${Bun.env.AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime`

// Import the handler module dynamically
const handlerPath = Bun.env._HANDLER ?? 'handler.handler'
const [moduleName, functionName] = handlerPath.split('.')
const handlerModule = await import(`./${moduleName}.js`)
const handler = handlerModule[functionName]

// Main event loop
while (true) {
  try {
    // Get next invocation
    const nextResponse = await fetch(`${RUNTIME_API}/invocation/next`)
    const requestId = nextResponse.headers.get('Lambda-Runtime-Aws-Request-Id')!
    const event = await nextResponse.json()

    // Create context object
    const context = {
      awsRequestId: requestId,
      functionName: Bun.env.AWS_LAMBDA_FUNCTION_NAME,
      functionVersion: Bun.env.AWS_LAMBDA_FUNCTION_VERSION,
      memoryLimitInMB: Bun.env.AWS_LAMBDA_FUNCTION_MEMORY_SIZE,
      getRemainingTimeInMillis: () => 0, // Simplified
    }

    // Invoke handler
    const result = await handler(event, context)

    // Report success
    await fetch(`${RUNTIME_API}/invocation/${requestId}/response`, {
      method: 'POST',
      body: JSON.stringify(result),
      headers: { 'Content-Type': 'application/json' },
    })
  } catch (error) {
    console.error('Runtime error:', error)
  }
}
```

Update Dockerfile CMD:

```dockerfile
CMD ["bun", "run", "bootstrap.ts"]
```

## Custom Runtime with Lambda Layer

Smaller package, faster cold starts, more complex setup.

### Layer Structure

```
bun-runtime-layer/
└── bin/
    └── bun           # Bun binary
```

### Creating the Layer

```bash
#!/bin/bash
# build-layer.sh

LAYER_DIR="layer"
mkdir -p "$LAYER_DIR/bin"

# Download Bun for Amazon Linux 2023 (x86_64)
curl -fsSL https://github.com/oven-sh/bun/releases/latest/download/bun-linux-x64.zip -o bun.zip
unzip bun.zip
mv bun-linux-x64/bun "$LAYER_DIR/bin/"
chmod +x "$LAYER_DIR/bin/bun"

# Package layer
cd "$LAYER_DIR"
zip -r ../bun-runtime-layer.zip .
```

### Function Package Structure

```
function.zip
├── bootstrap
└── handler.js (or handler.ts)
```

### Bootstrap for Layer

```bash
#!/bin/bash
set -euo pipefail

export PATH="/opt/bin:$PATH"

RUNTIME_API="http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime"
HANDLER_FILE="${_HANDLER%.*}"
HANDLER_FUNCTION="${_HANDLER##*.}"

while true; do
  HEADERS=$(mktemp)
  EVENT=$(curl -sS -LD "$HEADERS" "${RUNTIME_API}/invocation/next")
  REQUEST_ID=$(grep -i "Lambda-Runtime-Aws-Request-Id" "$HEADERS" | tr -d '\r' | cut -d: -f2 | xargs)

  RESPONSE=$(echo "$EVENT" | bun run "${LAMBDA_TASK_ROOT}/${HANDLER_FILE}.ts" "$HANDLER_FUNCTION" 2>&1) || {
    curl -sS -X POST "${RUNTIME_API}/invocation/${REQUEST_ID}/error" \
      -d '{"errorMessage":"Handler error"}' \
      -H "Content-Type: application/json"
    continue
  }

  curl -sS -X POST "${RUNTIME_API}/invocation/${REQUEST_ID}/response" \
    -d "$RESPONSE" \
    -H "Content-Type: application/json"

  rm -f "$HEADERS"
done
```

## Build and Bundle

### Bun Build Configuration

```typescript
// build.ts
await Bun.build({
  entrypoints: ['./src/handler.ts'],
  outdir: './dist',
  target: 'bun',
  minify: true,
  sourcemap: 'external',
  external: ['@aws-sdk/*'],  // Use Lambda's built-in AWS SDK
})
```

### Build Script

```json
{
  "scripts": {
    "build": "bun run build.ts",
    "build:watch": "bun run --watch build.ts"
  }
}
```

### Bundle for Node.js Compatibility

If targeting Node.js runtime with Bun-built code:

```typescript
await Bun.build({
  entrypoints: ['./src/handler.ts'],
  outdir: './dist',
  target: 'node',
  format: 'esm',
  minify: true,
})
```

## IaC Integration

### CDK (TypeScript)

```typescript
import * as cdk from 'aws-cdk-lib'
import * as lambda from 'aws-cdk-lib/aws-lambda'

// Container image function
const bunLambda = new lambda.DockerImageFunction(this, 'BunLambda', {
  code: lambda.DockerImageCode.fromImageAsset('./lambda'),
  architecture: lambda.Architecture.X86_64,
  memorySize: 512,
  timeout: cdk.Duration.seconds(30),
  environment: {
    NODE_ENV: 'production',
  },
})

// With custom runtime layer
const bunLayer = new lambda.LayerVersion(this, 'BunLayer', {
  code: lambda.Code.fromAsset('./bun-runtime-layer.zip'),
  compatibleRuntimes: [lambda.Runtime.PROVIDED_AL2023],
  compatibleArchitectures: [lambda.Architecture.X86_64],
})

const bunFunctionWithLayer = new lambda.Function(this, 'BunFunctionWithLayer', {
  runtime: lambda.Runtime.PROVIDED_AL2023,
  handler: 'handler.handler',
  code: lambda.Code.fromAsset('./dist'),
  layers: [bunLayer],
  memorySize: 512,
  timeout: cdk.Duration.seconds(30),
})
```

### SAM Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  BunFunction:
    Type: AWS::Serverless::Function
    Properties:
      PackageType: Image
      MemorySize: 512
      Timeout: 30
      Architectures:
        - x86_64
      Events:
        Api:
          Type: HttpApi
          Properties:
            Path: /{proxy+}
            Method: ANY
    Metadata:
      Dockerfile: Dockerfile
      DockerContext: ./lambda
      DockerTag: latest
```

### Terraform

```hcl
resource "aws_lambda_function" "bun_lambda" {
  function_name = "bun-lambda"
  package_type  = "Image"
  image_uri     = "${aws_ecr_repository.lambda.repository_url}:latest"
  role          = aws_iam_role.lambda.arn
  memory_size   = 512
  timeout       = 30

  environment {
    variables = {
      NODE_ENV = "production"
    }
  }
}
```

### SST v3

```typescript
// sst.config.ts
export default $config({
  app(input) {
    return {
      name: 'my-app',
      removal: 'remove',
    }
  },
  async run() {
    // Using container
    const bunLambda = new sst.aws.Function('BunLambda', {
      handler: 'packages/functions/src/handler.handler',
      runtime: 'container',
      container: {
        dockerfile: 'packages/functions/Dockerfile',
      },
      memory: '512 MB',
      timeout: '30 seconds',
    })

    // API Gateway integration
    const api = new sst.aws.ApiGatewayV2('Api')
    api.route('ANY /{proxy+}', bunLambda)

    return { apiUrl: api.url }
  },
})
```

## Environment and Configuration

### Lambda Configuration Recommendations

| Setting | Recommended | Notes |
|---------|-------------|-------|
| Memory | 512MB-1024MB | More memory = more CPU |
| Timeout | 30s (API), 900s (async) | Match use case |
| Architecture | x86_64 | ARM64 support varies |
| Provisioned Concurrency | Consider for latency-critical | Eliminates cold starts |

### Environment Variables

```typescript
// Access in handler
const config = {
  databaseUrl: Bun.env.DATABASE_URL,
  apiKey: Bun.env.API_KEY,
  stage: Bun.env.STAGE ?? 'dev',
  region: Bun.env.AWS_REGION,
}
```

### Cold Start Optimization

1. **Minimize dependencies**: Bundle with Bun, tree-shake unused code
2. **Use Bun's fast startup**: Bun starts faster than Node.js
3. **Lazy initialization**: Initialize heavy resources on first request
4. **Keep handler warm**: Use provisioned concurrency or scheduled pings
5. **Optimize imports**: Use dynamic imports for rarely-used code paths

```typescript
// Lazy initialization pattern
let dbClient: DatabaseClient | null = null

async function getDbClient(): Promise<DatabaseClient> {
  if (!dbClient) {
    const { DatabaseClient } = await import('./db-client')
    dbClient = new DatabaseClient(Bun.env.DATABASE_URL!)
    await dbClient.connect()
  }
  return dbClient
}

export async function handler(event: APIGatewayProxyEventV2) {
  const db = await getDbClient()
  // Use db...
}
```
