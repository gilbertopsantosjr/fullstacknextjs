# Event Sources Reference

## Table of Contents
- [API Gateway HTTP API (v2)](#api-gateway-http-api-v2)
- [API Gateway REST API (v1)](#api-gateway-rest-api-v1)
- [Application Load Balancer (ALB)](#application-load-balancer-alb)
- [SQS](#sqs)
- [SNS](#sns)
- [EventBridge (CloudWatch Events)](#eventbridge-cloudwatch-events)
- [S3](#s3)
- [DynamoDB Streams](#dynamodb-streams)
- [Scheduled Events (Cron)](#scheduled-events-cron)

## API Gateway HTTP API (v2)

Most common for REST APIs. Simpler, cheaper, faster than REST API v1.

### Event Structure

```typescript
interface APIGatewayProxyEventV2 {
  version: '2.0'
  routeKey: string                    // e.g., 'GET /users/{id}'
  rawPath: string                     // e.g., '/users/123'
  rawQueryString: string              // e.g., 'limit=10&offset=0'
  headers: Record<string, string>
  queryStringParameters?: Record<string, string>
  pathParameters?: Record<string, string>
  body?: string
  isBase64Encoded: boolean
  requestContext: {
    accountId: string
    apiId: string
    domainName: string
    domainPrefix: string
    http: {
      method: string
      path: string
      protocol: string
      sourceIp: string
      userAgent: string
    }
    requestId: string
    routeKey: string
    stage: string
    time: string
    timeEpoch: number
  }
}
```

### Response Structure

```typescript
interface APIGatewayProxyResultV2 {
  statusCode: number
  headers?: Record<string, string>
  body?: string
  isBase64Encoded?: boolean
  cookies?: string[]
}
```

### Handler Example

```typescript
import type { APIGatewayProxyEventV2, APIGatewayProxyResultV2 } from 'aws-lambda'

export async function handler(event: APIGatewayProxyEventV2): Promise<APIGatewayProxyResultV2> {
  const { pathParameters, body, queryStringParameters } = event
  const userId = pathParameters?.id

  try {
    // Parse JSON body if present
    const payload = body ? JSON.parse(body) : undefined

    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ userId, received: payload }),
    }
  } catch (error) {
    console.error('Handler error:', error)
    return {
      statusCode: 500,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ error: 'Internal server error' }),
    }
  }
}
```

## API Gateway REST API (v1)

Legacy format, more features (request validation, WAF integration).

### Event Structure

```typescript
interface APIGatewayProxyEvent {
  resource: string                    // e.g., '/users/{id}'
  path: string                        // e.g., '/users/123'
  httpMethod: string                  // e.g., 'GET'
  headers: Record<string, string> | null
  multiValueHeaders: Record<string, string[]> | null
  queryStringParameters: Record<string, string> | null
  multiValueQueryStringParameters: Record<string, string[]> | null
  pathParameters: Record<string, string> | null
  stageVariables: Record<string, string> | null
  body: string | null
  isBase64Encoded: boolean
  requestContext: {
    accountId: string
    apiId: string
    authorizer?: Record<string, any>
    httpMethod: string
    identity: {
      sourceIp: string
      userAgent: string
    }
    path: string
    protocol: string
    requestId: string
    requestTime: string
    requestTimeEpoch: number
    resourceId: string
    resourcePath: string
    stage: string
  }
}
```

### Response Structure

```typescript
interface APIGatewayProxyResult {
  statusCode: number
  headers?: Record<string, string>
  multiValueHeaders?: Record<string, string[]>
  body: string
  isBase64Encoded?: boolean
}
```

### Handler Example

```typescript
import type { APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda'

export async function handler(event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> {
  const { httpMethod, pathParameters, body } = event

  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      method: httpMethod,
      userId: pathParameters?.id,
    }),
  }
}
```

## Application Load Balancer (ALB)

### Event Structure

```typescript
interface ALBEvent {
  requestContext: {
    elb: {
      targetGroupArn: string
    }
  }
  httpMethod: string
  path: string
  queryStringParameters: Record<string, string> | null
  headers: Record<string, string>
  body: string | null
  isBase64Encoded: boolean
}
```

### Response Structure

```typescript
interface ALBResult {
  statusCode: number
  statusDescription?: string          // e.g., '200 OK'
  headers?: Record<string, string>
  body: string
  isBase64Encoded?: boolean
}
```

### Handler Example

```typescript
import type { ALBEvent, ALBResult } from 'aws-lambda'

export async function handler(event: ALBEvent): Promise<ALBResult> {
  return {
    statusCode: 200,
    statusDescription: '200 OK',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ path: event.path }),
  }
}
```

## SQS

### Event Structure

```typescript
interface SQSEvent {
  Records: SQSRecord[]
}

interface SQSRecord {
  messageId: string
  receiptHandle: string
  body: string                        // Message content (often JSON string)
  attributes: {
    ApproximateReceiveCount: string
    SentTimestamp: string
    SenderId: string
    ApproximateFirstReceiveTimestamp: string
  }
  messageAttributes: Record<string, SQSMessageAttribute>
  md5OfBody: string
  eventSource: 'aws:sqs'
  eventSourceARN: string
  awsRegion: string
}
```

### Response Structure (Partial Batch Failure)

```typescript
interface SQSBatchResponse {
  batchItemFailures: Array<{
    itemIdentifier: string            // messageId of failed message
  }>
}
```

### Handler Example

```typescript
import type { SQSEvent, SQSBatchResponse } from 'aws-lambda'

interface MessagePayload {
  userId: string
  action: string
}

export async function handler(event: SQSEvent): Promise<SQSBatchResponse> {
  const batchItemFailures: SQSBatchResponse['batchItemFailures'] = []

  for (const record of event.Records) {
    try {
      const payload: MessagePayload = JSON.parse(record.body)
      console.log('Processing message:', record.messageId, payload)

      // Process the message
      await processMessage(payload)

    } catch (error) {
      console.error('Failed to process message:', record.messageId, error)
      // Report failure for partial batch response
      batchItemFailures.push({ itemIdentifier: record.messageId })
    }
  }

  return { batchItemFailures }
}

async function processMessage(payload: MessagePayload): Promise<void> {
  // Business logic here
}
```

## SNS

### Event Structure

```typescript
interface SNSEvent {
  Records: SNSEventRecord[]
}

interface SNSEventRecord {
  EventVersion: string
  EventSubscriptionArn: string
  EventSource: 'aws:sns'
  Sns: {
    SignatureVersion: string
    Timestamp: string
    Signature: string
    SigningCertUrl: string
    MessageId: string
    Message: string                   // Payload (often JSON string)
    MessageAttributes: Record<string, SNSMessageAttribute>
    Type: string
    UnsubscribeUrl: string
    TopicArn: string
    Subject: string | null
  }
}
```

### Handler Example

```typescript
import type { SNSEvent } from 'aws-lambda'

export async function handler(event: SNSEvent): Promise<void> {
  for (const record of event.Records) {
    const message = record.Sns.Message
    console.log('SNS Message:', record.Sns.MessageId, message)

    try {
      const payload = JSON.parse(message)
      await processNotification(payload)
    } catch (error) {
      console.error('Failed to process SNS message:', error)
      throw error  // Retry via DLQ
    }
  }
}

async function processNotification(payload: unknown): Promise<void> {
  // Business logic here
}
```

## EventBridge (CloudWatch Events)

### Event Structure

```typescript
interface EventBridgeEvent<T = unknown> {
  version: string
  id: string
  'detail-type': string               // Custom event type
  source: string                      // e.g., 'my.application'
  account: string
  time: string                        // ISO 8601
  region: string
  resources: string[]
  detail: T                           // Custom payload
}
```

### Handler Example

```typescript
interface OrderCreatedDetail {
  orderId: string
  customerId: string
  total: number
}

type OrderCreatedEvent = EventBridgeEvent<OrderCreatedDetail>

export async function handler(event: OrderCreatedEvent): Promise<void> {
  const { detail, 'detail-type': detailType } = event

  console.log('EventBridge event:', detailType, detail.orderId)

  await processOrder(detail)
}

async function processOrder(detail: OrderCreatedDetail): Promise<void> {
  // Business logic here
}
```

## S3

### Event Structure

```typescript
interface S3Event {
  Records: S3EventRecord[]
}

interface S3EventRecord {
  eventVersion: string
  eventSource: 'aws:s3'
  awsRegion: string
  eventTime: string
  eventName: string                   // e.g., 'ObjectCreated:Put'
  s3: {
    s3SchemaVersion: string
    configurationId: string
    bucket: {
      name: string
      ownerIdentity: { principalId: string }
      arn: string
    }
    object: {
      key: string                     // URL-encoded
      size: number
      eTag: string
      sequencer: string
    }
  }
}
```

### Handler Example

```typescript
import type { S3Event } from 'aws-lambda'

export async function handler(event: S3Event): Promise<void> {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '))

    console.log('S3 event:', record.eventName, `s3://${bucket}/${key}`)

    await processS3Object(bucket, key)
  }
}

async function processS3Object(bucket: string, key: string): Promise<void> {
  // Fetch and process the object
}
```

## DynamoDB Streams

### Event Structure

```typescript
interface DynamoDBStreamEvent {
  Records: DynamoDBRecord[]
}

interface DynamoDBRecord {
  eventID: string
  eventName: 'INSERT' | 'MODIFY' | 'REMOVE'
  eventVersion: string
  eventSource: 'aws:dynamodb'
  awsRegion: string
  dynamodb: {
    ApproximateCreationDateTime?: number
    Keys: Record<string, AttributeValue>
    NewImage?: Record<string, AttributeValue>
    OldImage?: Record<string, AttributeValue>
    SequenceNumber: string
    SizeBytes: number
    StreamViewType: string
  }
  eventSourceARN: string
}
```

### Handler Example

```typescript
import type { DynamoDBStreamEvent } from 'aws-lambda'
import { unmarshall } from '@aws-sdk/util-dynamodb'

export async function handler(event: DynamoDBStreamEvent): Promise<void> {
  for (const record of event.Records) {
    console.log('DynamoDB Stream:', record.eventName, record.eventID)

    if (record.eventName === 'INSERT' && record.dynamodb.NewImage) {
      const newItem = unmarshall(record.dynamodb.NewImage as any)
      await handleInsert(newItem)
    }
  }
}

async function handleInsert(item: Record<string, unknown>): Promise<void> {
  // Business logic here
}
```

## Scheduled Events (Cron)

Same as EventBridge, but with specific detail-type.

### Event Structure

```typescript
interface ScheduledEvent {
  version: string
  id: string
  'detail-type': 'Scheduled Event'
  source: 'aws.events'
  account: string
  time: string
  region: string
  resources: string[]                 // Rule ARN
  detail: {}                          // Empty for scheduled events
}
```

### Handler Example

```typescript
interface ScheduledEvent {
  'detail-type': 'Scheduled Event'
  source: 'aws.events'
  time: string
  resources: string[]
}

export async function handler(event: ScheduledEvent): Promise<void> {
  console.log('Scheduled execution at:', event.time)

  await runScheduledTask()
}

async function runScheduledTask(): Promise<void> {
  // Cron job logic here
}
```
