# Node.js to Bun Migration Reference

## Table of Contents
- [API Compatibility](#api-compatibility)
- [Common Migration Patterns](#common-migration-patterns)
- [AWS SDK Usage](#aws-sdk-usage)
- [HTTP Clients](#http-clients)
- [File System Operations](#file-system-operations)
- [Environment Variables](#environment-variables)
- [Module System](#module-system)

## API Compatibility

### Bun-Native APIs (Preferred)

| Node.js | Bun Native | Notes |
|---------|-----------|-------|
| `fetch` (node-fetch) | `fetch` | Built-in, no import needed |
| `fs.readFile` | `Bun.file().text()` | Faster, simpler API |
| `fs.writeFile` | `Bun.write()` | Faster, simpler API |
| `crypto` | `Bun.hash()`, `Bun.CryptoHasher` | Faster crypto |
| `process.env` | `Bun.env` | Same usage |
| `Buffer` | `Buffer` | Fully compatible |

### Fully Compatible (No Changes Needed)

- `console.log/error/warn`
- `JSON.parse/stringify`
- `setTimeout/setInterval`
- `Promise`, `async/await`
- Most `util` functions
- `path` module
- `url` module
- `querystring` module

## Common Migration Patterns

### Callback to Promise/Async

Node.js (callback style):
```typescript
import { readFile } from 'fs'

export function handler(event, context, callback) {
  readFile('config.json', 'utf8', (err, data) => {
    if (err) {
      callback(err)
      return
    }
    const config = JSON.parse(data)
    callback(null, { statusCode: 200, body: JSON.stringify(config) })
  })
}
```

Bun (async/await):
```typescript
export async function handler(event: APIGatewayProxyEventV2) {
  const config = await Bun.file('config.json').json()
  return {
    statusCode: 200,
    body: JSON.stringify(config),
  }
}
```

### HTTP Requests

Node.js (axios):
```typescript
import axios from 'axios'

export async function handler(event) {
  const response = await axios.get('https://api.example.com/data', {
    headers: { Authorization: `Bearer ${process.env.API_TOKEN}` },
  })
  return { statusCode: 200, body: JSON.stringify(response.data) }
}
```

Bun (native fetch):
```typescript
export async function handler(event: APIGatewayProxyEventV2) {
  const response = await fetch('https://api.example.com/data', {
    headers: { Authorization: `Bearer ${Bun.env.API_TOKEN}` },
  })
  const data = await response.json()
  return { statusCode: 200, body: JSON.stringify(data) }
}
```

### File Operations

Node.js:
```typescript
import { readFileSync, writeFileSync } from 'fs'

export function handler(event) {
  const data = readFileSync('/tmp/data.json', 'utf8')
  const parsed = JSON.parse(data)
  parsed.updated = Date.now()
  writeFileSync('/tmp/data.json', JSON.stringify(parsed))
  return { statusCode: 200, body: 'OK' }
}
```

Bun:
```typescript
export async function handler(event: APIGatewayProxyEventV2) {
  const parsed = await Bun.file('/tmp/data.json').json()
  parsed.updated = Date.now()
  await Bun.write('/tmp/data.json', JSON.stringify(parsed))
  return { statusCode: 200, body: 'OK' }
}
```

### Hashing

Node.js:
```typescript
import { createHash } from 'crypto'

export function handler(event) {
  const hash = createHash('sha256')
    .update(event.body)
    .digest('hex')
  return { statusCode: 200, body: hash }
}
```

Bun:
```typescript
export function handler(event: APIGatewayProxyEventV2) {
  const hash = Bun.hash(event.body ?? '', 'sha256').toString(16)
  // Or for more control:
  const hasher = new Bun.CryptoHasher('sha256')
  hasher.update(event.body ?? '')
  const hexHash = hasher.digest('hex')
  return { statusCode: 200, body: hexHash }
}
```

## AWS SDK Usage

### SDK v3 (Recommended)

Works identically in both Node.js and Bun:

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient, GetCommand, PutCommand } from '@aws-sdk/lib-dynamodb'

const client = new DynamoDBClient({})
const docClient = DynamoDBDocumentClient.from(client)

export async function handler(event: APIGatewayProxyEventV2) {
  const { Item } = await docClient.send(
    new GetCommand({
      TableName: Bun.env.TABLE_NAME,
      Key: { pk: event.pathParameters?.id },
    })
  )

  return {
    statusCode: Item ? 200 : 404,
    body: JSON.stringify(Item ?? { error: 'Not found' }),
  }
}
```

### SDK v2 to v3 Migration

SDK v2 (deprecated):
```typescript
import AWS from 'aws-sdk'
const dynamodb = new AWS.DynamoDB.DocumentClient()

export async function handler(event) {
  const result = await dynamodb.get({
    TableName: process.env.TABLE_NAME,
    Key: { pk: event.pathParameters.id },
  }).promise()
  return { statusCode: 200, body: JSON.stringify(result.Item) }
}
```

SDK v3 (Bun compatible):
```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient, GetCommand } from '@aws-sdk/lib-dynamodb'

const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}))

export async function handler(event: APIGatewayProxyEventV2) {
  const { Item } = await docClient.send(
    new GetCommand({
      TableName: Bun.env.TABLE_NAME,
      Key: { pk: event.pathParameters?.id },
    })
  )
  return { statusCode: 200, body: JSON.stringify(Item) }
}
```

## HTTP Clients

### Replacing axios/got/node-fetch

All can be replaced with native `fetch`:

```typescript
// GET with query params
const url = new URL('https://api.example.com/search')
url.searchParams.set('q', query)
const response = await fetch(url)

// POST with JSON
const response = await fetch('https://api.example.com/data', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(payload),
})

// With timeout (Bun-specific)
const controller = new AbortController()
const timeoutId = setTimeout(() => controller.abort(), 5000)

try {
  const response = await fetch(url, { signal: controller.signal })
  clearTimeout(timeoutId)
  return await response.json()
} catch (error) {
  if (error.name === 'AbortError') {
    throw new Error('Request timeout')
  }
  throw error
}
```

## File System Operations

### Bun File API

```typescript
// Read text
const text = await Bun.file('path/to/file.txt').text()

// Read JSON
const json = await Bun.file('path/to/file.json').json()

// Read binary
const bytes = await Bun.file('path/to/file.bin').arrayBuffer()

// Write text/JSON
await Bun.write('path/to/output.json', JSON.stringify(data))

// Write binary
await Bun.write('path/to/output.bin', new Uint8Array([1, 2, 3]))

// Check if file exists
const file = Bun.file('path/to/file.txt')
const exists = await file.exists()

// Get file size
const size = file.size
```

### When to Use Node.js fs

For operations Bun.file doesn't cover:

```typescript
import { mkdir, readdir, unlink, stat } from 'fs/promises'

// Create directory
await mkdir('/tmp/cache', { recursive: true })

// List directory
const files = await readdir('/tmp/cache')

// Delete file
await unlink('/tmp/cache/old-file.json')

// Get file stats
const stats = await stat('/tmp/cache/file.json')
```

## Environment Variables

### Access Patterns

```typescript
// Bun.env (preferred in Bun)
const dbUrl = Bun.env.DATABASE_URL

// process.env (Node.js compatible, also works)
const dbUrl = process.env.DATABASE_URL

// With defaults
const stage = Bun.env.STAGE ?? 'dev'
const port = parseInt(Bun.env.PORT ?? '3000', 10)

// Required env check
function requireEnv(key: string): string {
  const value = Bun.env[key]
  if (!value) {
    throw new Error(`Missing required environment variable: ${key}`)
  }
  return value
}

const apiKey = requireEnv('API_KEY')
```

## Module System

### ESM (Recommended)

```typescript
// Named imports
import { DynamoDBClient } from '@aws-sdk/client-dynamodb'

// Default import
import Stripe from 'stripe'

// Type-only import
import type { APIGatewayProxyEventV2 } from 'aws-lambda'

// Dynamic import (lazy loading)
const { heavyModule } = await import('./heavy-module')
```

### CommonJS Compatibility

Bun supports CommonJS, but ESM is preferred:

```typescript
// CommonJS (works but not recommended)
const { readFile } = require('fs')

// ESM equivalent (preferred)
import { readFile } from 'fs'
```

### Package.json Type

```json
{
  "type": "module",
  "exports": {
    ".": "./dist/handler.js"
  }
}
```

## Full Migration Example

### Before (Node.js)

```typescript
const AWS = require('aws-sdk')
const axios = require('axios')
const { readFileSync } = require('fs')
const { createHash } = require('crypto')

const dynamodb = new AWS.DynamoDB.DocumentClient()

exports.handler = async (event, context) => {
  try {
    const config = JSON.parse(readFileSync('config.json', 'utf8'))

    const response = await axios.get(config.apiUrl, {
      headers: { 'X-API-Key': process.env.API_KEY }
    })

    const hash = createHash('md5').update(JSON.stringify(response.data)).digest('hex')

    await dynamodb.put({
      TableName: process.env.TABLE_NAME,
      Item: {
        pk: event.pathParameters.id,
        data: response.data,
        hash: hash,
        timestamp: Date.now()
      }
    }).promise()

    return {
      statusCode: 200,
      body: JSON.stringify({ success: true, hash })
    }
  } catch (error) {
    console.error('Error:', error)
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal server error' })
    }
  }
}
```

### After (Bun)

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb'
import type { APIGatewayProxyEventV2, APIGatewayProxyResultV2 } from 'aws-lambda'

const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}))

interface Config {
  apiUrl: string
}

export async function handler(event: APIGatewayProxyEventV2): Promise<APIGatewayProxyResultV2> {
  try {
    const config: Config = await Bun.file('config.json').json()

    const response = await fetch(config.apiUrl, {
      headers: { 'X-API-Key': Bun.env.API_KEY! },
    })
    const data = await response.json()

    const hash = Bun.hash(JSON.stringify(data), 'md5').toString(16)

    await docClient.send(
      new PutCommand({
        TableName: Bun.env.TABLE_NAME!,
        Item: {
          pk: event.pathParameters?.id,
          data,
          hash,
          timestamp: Date.now(),
        },
      })
    )

    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ success: true, hash }),
    }
  } catch (error) {
    console.error('Error:', error)
    return {
      statusCode: 500,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ error: 'Internal server error' }),
    }
  }
}
```
