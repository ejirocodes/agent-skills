# Microservices

## Table of Contents

- [Microservices Overview](#microservices-overview)
- [TCP Transport](#tcp-transport)
- [Redis Transport](#redis-transport)
- [NATS Transport](#nats-transport)
- [Kafka Transport](#kafka-transport)
- [Message and Event Patterns](#message-and-event-patterns)
- [Hybrid Applications](#hybrid-applications)

---

## Microservices Overview

NestJS microservices use a transport layer to communicate between services. Communication can be synchronous (request-response) or asynchronous (event-based).

### Installation

```bash
npm install @nestjs/microservices
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Transport** | Communication protocol (TCP, Redis, NATS, Kafka, etc.) |
| **Message Pattern** | Request-response pattern (`@MessagePattern`) |
| **Event Pattern** | Fire-and-forget pattern (`@EventPattern`) |
| **ClientProxy** | Client to send messages to microservices |

### Architecture Overview

```
┌─────────────────┐     Message/Event      ┌─────────────────┐
│   API Gateway   │ ──────────────────────>│  Microservice   │
│   (HTTP/REST)   │                        │   (TCP/Redis)   │
└─────────────────┘                        └─────────────────┘
         │                                          │
         │                                          │
         ▼                                          ▼
┌─────────────────┐                        ┌─────────────────┐
│     Client      │                        │    Database     │
└─────────────────┘                        └─────────────────┘
```

---

## TCP Transport

TCP is the default transport, suitable for internal communication within a network.

### Microservice Setup

```typescript
// main.ts (microservice)
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
      options: {
        host: '0.0.0.0',
        port: 3001,
      },
    },
  );

  await app.listen();
  console.log('Microservice is listening on port 3001');
}
bootstrap();
```

### Controller with Message Patterns

```typescript
// users/users.controller.ts (microservice)
import { Controller } from '@nestjs/common';
import { MessagePattern, Payload } from '@nestjs/microservices';
import { UsersService } from './users.service';

@Controller()
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @MessagePattern({ cmd: 'get_users' })
  getUsers() {
    return this.usersService.findAll();
  }

  @MessagePattern({ cmd: 'get_user' })
  getUser(@Payload() data: { id: string }) {
    return this.usersService.findOne(data.id);
  }

  @MessagePattern({ cmd: 'create_user' })
  createUser(@Payload() data: CreateUserDto) {
    return this.usersService.create(data);
  }
}
```

### Client Setup (API Gateway)

```typescript
// app.module.ts (gateway)
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'USERS_SERVICE',
        transport: Transport.TCP,
        options: {
          host: 'localhost',
          port: 3001,
        },
      },
    ]),
  ],
})
export class AppModule {}
```

### Using the Client

```typescript
// gateway/users.controller.ts
import { Controller, Get, Post, Body, Param, Inject } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import { firstValueFrom } from 'rxjs';

@Controller('users')
export class UsersController {
  constructor(
    @Inject('USERS_SERVICE') private readonly usersClient: ClientProxy,
  ) {}

  @Get()
  async getUsers() {
    return firstValueFrom(
      this.usersClient.send({ cmd: 'get_users' }, {}),
    );
  }

  @Get(':id')
  async getUser(@Param('id') id: string) {
    return firstValueFrom(
      this.usersClient.send({ cmd: 'get_user' }, { id }),
    );
  }

  @Post()
  async createUser(@Body() createUserDto: CreateUserDto) {
    return firstValueFrom(
      this.usersClient.send({ cmd: 'create_user' }, createUserDto),
    );
  }
}
```

---

## Redis Transport

Redis provides pub/sub messaging with support for message persistence.

### Installation

```bash
npm install ioredis
```

### Microservice Setup

```typescript
// main.ts (microservice)
async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.REDIS,
      options: {
        host: 'localhost',
        port: 6379,
        retryAttempts: 5,
        retryDelay: 1000,
      },
    },
  );

  await app.listen();
}
```

### Client Setup

```typescript
// app.module.ts (gateway)
@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        name: 'USERS_SERVICE',
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => ({
          transport: Transport.REDIS,
          options: {
            host: configService.get('REDIS_HOST'),
            port: configService.get('REDIS_PORT'),
          },
        }),
        inject: [ConfigService],
      },
    ]),
  ],
})
export class AppModule {}
```

### Event-Based Communication

```typescript
// Microservice controller
@Controller()
export class OrdersController {
  @EventPattern('order_created')
  handleOrderCreated(@Payload() data: OrderCreatedEvent) {
    console.log('Order created:', data);
    // Process the event (no response expected)
  }
}

// Gateway/Publisher
@Controller('orders')
export class OrdersController {
  constructor(
    @Inject('ORDERS_SERVICE') private ordersClient: ClientProxy,
  ) {}

  @Post()
  async createOrder(@Body() createOrderDto: CreateOrderDto) {
    const order = await this.ordersService.create(createOrderDto);

    // Emit event (fire-and-forget)
    this.ordersClient.emit('order_created', {
      orderId: order.id,
      userId: order.userId,
      items: order.items,
    });

    return order;
  }
}
```

---

## NATS Transport

NATS is a high-performance messaging system ideal for cloud-native applications.

### Installation

```bash
npm install nats
```

### Microservice Setup

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.NATS,
      options: {
        servers: ['nats://localhost:4222'],
        queue: 'users_queue', // Queue group for load balancing
      },
    },
  );

  await app.listen();
}
```

### Client Setup

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'USERS_SERVICE',
        transport: Transport.NATS,
        options: {
          servers: ['nats://localhost:4222'],
        },
      },
    ]),
  ],
})
export class AppModule {}
```

### NATS with Wildcards

```typescript
// Microservice - subscribe to pattern
@Controller()
export class NotificationsController {
  // Matches: user.created, user.updated, user.deleted
  @MessagePattern('user.*')
  handleUserEvents(@Payload() data: any, @Ctx() context: NatsContext) {
    const subject = context.getSubject();
    console.log(`Received event on ${subject}:`, data);
  }

  // Matches: order.payment.success, order.shipping.completed
  @MessagePattern('order.>')
  handleOrderEvents(@Payload() data: any) {
    // Handle any order-related event
  }
}
```

---

## Kafka Transport

Apache Kafka provides durable, high-throughput messaging for event streaming.

### Installation

```bash
npm install kafkajs
```

### Microservice Setup

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.KAFKA,
      options: {
        client: {
          clientId: 'users-service',
          brokers: ['localhost:9092'],
        },
        consumer: {
          groupId: 'users-consumer-group',
        },
      },
    },
  );

  await app.listen();
}
```

### Controller with Kafka Topics

```typescript
// users/users.controller.ts
import { Controller } from '@nestjs/common';
import { MessagePattern, Payload, Ctx, KafkaContext } from '@nestjs/microservices';

@Controller()
export class UsersController {
  @MessagePattern('users.get')
  getUsers(@Payload() message: any, @Ctx() context: KafkaContext) {
    const originalMessage = context.getMessage();
    const partition = context.getPartition();
    const topic = context.getTopic();

    console.log(`Received message on topic: ${topic}, partition: ${partition}`);

    return this.usersService.findAll();
  }

  @EventPattern('users.created')
  handleUserCreated(
    @Payload() data: UserCreatedEvent,
    @Ctx() context: KafkaContext,
  ) {
    const { offset } = context.getMessage();
    console.log(`Processing message at offset ${offset}`);

    // Process event
  }
}
```

### Client Setup with Kafka

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'KAFKA_SERVICE',
        transport: Transport.KAFKA,
        options: {
          client: {
            clientId: 'api-gateway',
            brokers: ['localhost:9092'],
          },
          producer: {
            allowAutoTopicCreation: true,
          },
        },
      },
    ]),
  ],
})
export class AppModule {}

// Using the client
@Injectable()
export class UsersService {
  constructor(
    @Inject('KAFKA_SERVICE') private kafkaClient: ClientKafka,
  ) {}

  async onModuleInit() {
    // Subscribe to response topics
    this.kafkaClient.subscribeToResponseOf('users.get');
    await this.kafkaClient.connect();
  }

  async getUsers() {
    return firstValueFrom(
      this.kafkaClient.send('users.get', { timestamp: Date.now() }),
    );
  }

  async emitUserCreated(user: User) {
    this.kafkaClient.emit('users.created', {
      key: user.id,
      value: { userId: user.id, email: user.email },
    });
  }
}
```

---

## Message and Event Patterns

### Message Pattern (Request-Response)

Synchronous communication where a response is expected.

```typescript
// Microservice
@MessagePattern({ cmd: 'sum' })
accumulate(@Payload() data: number[]): number {
  return data.reduce((a, b) => a + b, 0);
}

// Client
const result = await firstValueFrom(
  this.client.send({ cmd: 'sum' }, [1, 2, 3, 4, 5]),
);
console.log(result); // 15
```

### Event Pattern (Fire-and-Forget)

Asynchronous communication with no response.

```typescript
// Microservice
@EventPattern('user_created')
handleUserCreated(@Payload() data: UserCreatedEvent) {
  // Send welcome email
  this.emailService.sendWelcome(data.email);
  // No return value
}

// Client
this.client.emit('user_created', { email: 'user@example.com' });
// Continues immediately, doesn't wait
```

### Error Handling

```typescript
// Microservice
@MessagePattern({ cmd: 'get_user' })
async getUser(@Payload() data: { id: string }) {
  const user = await this.usersService.findOne(data.id);

  if (!user) {
    // Return RpcException for client to catch
    throw new RpcException({
      status: 404,
      message: 'User not found',
    });
  }

  return user;
}

// Client with error handling
@Get(':id')
async getUser(@Param('id') id: string) {
  try {
    return await firstValueFrom(
      this.client.send({ cmd: 'get_user' }, { id }),
    );
  } catch (error) {
    if (error.status === 404) {
      throw new NotFoundException(error.message);
    }
    throw new InternalServerErrorException();
  }
}
```

### Timeout Handling

```typescript
import { timeout, catchError } from 'rxjs/operators';
import { TimeoutError } from 'rxjs';

@Get(':id')
async getUser(@Param('id') id: string) {
  return firstValueFrom(
    this.client.send({ cmd: 'get_user' }, { id }).pipe(
      timeout(5000), // 5 second timeout
      catchError((err) => {
        if (err instanceof TimeoutError) {
          throw new RequestTimeoutException('Service did not respond');
        }
        throw err;
      }),
    ),
  );
}
```

---

## Hybrid Applications

Combine HTTP server with microservice transports in a single application.

### Basic Hybrid Setup

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  // Create HTTP application
  const app = await NestFactory.create(AppModule);

  // Connect microservice transport
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.TCP,
    options: {
      host: '0.0.0.0',
      port: 3001,
    },
  });

  // Connect additional transport
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.REDIS,
    options: {
      host: 'localhost',
      port: 6379,
    },
  });

  // Start all microservices
  await app.startAllMicroservices();

  // Start HTTP server
  await app.listen(3000);

  console.log('HTTP server on port 3000');
  console.log('TCP microservice on port 3001');
  console.log('Redis microservice connected');
}
bootstrap();
```

### Controller with Both HTTP and Microservice

```typescript
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  // HTTP endpoint
  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  // Microservice message pattern
  @MessagePattern({ cmd: 'get_users' })
  getUsers() {
    return this.usersService.findAll();
  }

  // HTTP endpoint
  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  // Microservice event pattern
  @EventPattern('user_created')
  handleUserCreated(@Payload() data: UserCreatedEvent) {
    this.usersService.sendWelcomeEmail(data.email);
  }
}
```

### Service Discovery Pattern

```typescript
// Use environment variables for service URLs
@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        name: 'USERS_SERVICE',
        imports: [ConfigModule],
        useFactory: (config: ConfigService) => ({
          transport: Transport.TCP,
          options: {
            host: config.get('USERS_SERVICE_HOST', 'localhost'),
            port: config.get('USERS_SERVICE_PORT', 3001),
          },
        }),
        inject: [ConfigService],
      },
      {
        name: 'ORDERS_SERVICE',
        imports: [ConfigModule],
        useFactory: (config: ConfigService) => ({
          transport: Transport.TCP,
          options: {
            host: config.get('ORDERS_SERVICE_HOST', 'localhost'),
            port: config.get('ORDERS_SERVICE_PORT', 3002),
          },
        }),
        inject: [ConfigService],
      },
    ]),
  ],
})
export class GatewayModule {}
```

### Health Checks for Microservices

```typescript
// health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheck,
  HealthCheckService,
  MicroserviceHealthIndicator,
} from '@nestjs/terminus';
import { Transport } from '@nestjs/microservices';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private microservice: MicroserviceHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () =>
        this.microservice.pingCheck('users-service', {
          transport: Transport.TCP,
          options: { host: 'localhost', port: 3001 },
        }),
      () =>
        this.microservice.pingCheck('orders-service', {
          transport: Transport.REDIS,
          options: { host: 'localhost', port: 6379 },
        }),
    ]);
  }
}
```
