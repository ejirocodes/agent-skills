# OpenAPI & GraphQL

## Table of Contents

- [OpenAPI/Swagger Setup](#openapiswagger-setup)
- [API Decorators](#api-decorators)
- [CLI Plugin](#cli-plugin)
- [GraphQL Code-First](#graphql-code-first)
- [Resolvers](#resolvers)
- [Subscriptions](#subscriptions)

---

## OpenAPI/Swagger Setup

The `@nestjs/swagger` module generates OpenAPI (Swagger) documentation automatically from your controllers and DTOs.

### Installation

```bash
npm install @nestjs/swagger
```

### Basic Setup

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('My API')
    .setDescription('The API documentation')
    .setVersion('1.0')
    .addTag('users')
    .addTag('auth')
    .addBearerAuth(
      {
        type: 'http',
        scheme: 'bearer',
        bearerFormat: 'JWT',
        name: 'JWT',
        description: 'Enter JWT token',
        in: 'header',
      },
      'JWT-auth', // Reference name for @ApiBearerAuth()
    )
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document); // Available at /api

  await app.listen(3000);
}
bootstrap();
```

### Advanced Configuration

```typescript
const config = new DocumentBuilder()
  .setTitle('E-Commerce API')
  .setDescription('API for e-commerce platform')
  .setVersion('2.0')
  .setContact('API Support', 'https://example.com', 'support@example.com')
  .setLicense('MIT', 'https://opensource.org/licenses/MIT')
  .addServer('http://localhost:3000', 'Development')
  .addServer('https://api.example.com', 'Production')
  .addApiKey({ type: 'apiKey', name: 'X-API-KEY', in: 'header' }, 'api-key')
  .addOAuth2({
    type: 'oauth2',
    flows: {
      implicit: {
        authorizationUrl: 'https://auth.example.com/oauth/authorize',
        scopes: {
          'read:users': 'Read user data',
          'write:users': 'Modify user data',
        },
      },
    },
  })
  .build();

const document = SwaggerModule.createDocument(app, config, {
  include: [UsersModule, AuthModule], // Only include specific modules
  deepScanRoutes: true,
  operationIdFactory: (controllerKey: string, methodKey: string) =>
    `${controllerKey}_${methodKey}`,
});

// Customize Swagger UI
SwaggerModule.setup('api', app, document, {
  swaggerOptions: {
    persistAuthorization: true,
    tagsSorter: 'alpha',
    operationsSorter: 'alpha',
  },
  customCss: '.swagger-ui .topbar { display: none }',
  customSiteTitle: 'My API Docs',
});
```

---

## API Decorators

### Controller and Method Decorators

```typescript
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiBearerAuth,
  ApiParam,
  ApiQuery,
  ApiBody,
  ApiConsumes,
  ApiProduces,
} from '@nestjs/swagger';

@ApiTags('users')
@ApiBearerAuth('JWT-auth')
@Controller('users')
export class UsersController {
  @Get()
  @ApiOperation({ summary: 'Get all users', description: 'Returns all users with pagination' })
  @ApiQuery({ name: 'page', required: false, type: Number, description: 'Page number' })
  @ApiQuery({ name: 'limit', required: false, type: Number, description: 'Items per page' })
  @ApiResponse({ status: 200, description: 'Users retrieved successfully', type: [UserResponseDto] })
  @ApiResponse({ status: 401, description: 'Unauthorized' })
  findAll(
    @Query('page') page?: number,
    @Query('limit') limit?: number,
  ) {
    return this.usersService.findAll({ page, limit });
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get user by ID' })
  @ApiParam({ name: 'id', type: String, description: 'User UUID' })
  @ApiResponse({ status: 200, description: 'User found', type: UserResponseDto })
  @ApiResponse({ status: 404, description: 'User not found' })
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }

  @Post()
  @ApiOperation({ summary: 'Create a new user' })
  @ApiBody({ type: CreateUserDto })
  @ApiResponse({ status: 201, description: 'User created', type: UserResponseDto })
  @ApiResponse({ status: 400, description: 'Validation failed' })
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Post('upload')
  @ApiConsumes('multipart/form-data')
  @ApiBody({
    schema: {
      type: 'object',
      properties: {
        file: {
          type: 'string',
          format: 'binary',
        },
      },
    },
  })
  @UseInterceptors(FileInterceptor('file'))
  uploadFile(@UploadedFile() file: Express.Multer.File) {
    return { filename: file.filename };
  }
}
```

### DTO Property Decorators

```typescript
import { ApiProperty, ApiPropertyOptional, ApiHideProperty } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({
    example: 'user@example.com',
    description: 'User email address',
    format: 'email',
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    example: 'StrongP@ss123',
    description: 'User password',
    minLength: 8,
  })
  @IsString()
  @MinLength(8)
  password: string;

  @ApiPropertyOptional({
    example: 'John Doe',
    description: 'User full name',
  })
  @IsOptional()
  @IsString()
  name?: string;

  @ApiProperty({
    enum: ['user', 'admin'],
    default: 'user',
    description: 'User role',
  })
  @IsEnum(['user', 'admin'])
  role: string;

  @ApiProperty({
    type: [String],
    example: ['tag1', 'tag2'],
    description: 'User tags',
  })
  @IsArray()
  @IsString({ each: true })
  tags: string[];
}

export class UserResponseDto {
  @ApiProperty({ example: '123e4567-e89b-12d3-a456-426614174000' })
  id: string;

  @ApiProperty({ example: 'user@example.com' })
  email: string;

  @ApiHideProperty() // Won't appear in Swagger
  password: string;

  @ApiProperty({ example: '2024-01-01T00:00:00.000Z' })
  createdAt: Date;
}
```

### Response Schemas

```typescript
export class PaginatedResponseDto<T> {
  data: T[];
  meta: PaginationMeta;
}

class PaginationMeta {
  @ApiProperty({ example: 100 })
  total: number;

  @ApiProperty({ example: 1 })
  page: number;

  @ApiProperty({ example: 10 })
  limit: number;

  @ApiProperty({ example: 10 })
  totalPages: number;
}

// Usage with generic
@Get()
@ApiOkResponse({
  schema: {
    allOf: [
      { $ref: getSchemaPath(PaginatedResponseDto) },
      {
        properties: {
          data: {
            type: 'array',
            items: { $ref: getSchemaPath(UserResponseDto) },
          },
        },
      },
    ],
  },
})
findAll() {}
```

---

## CLI Plugin

The CLI plugin automatically adds Swagger decorators at compile time, reducing boilerplate.

### Setup

```json
// nest-cli.json
{
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "plugins": [
      {
        "name": "@nestjs/swagger",
        "options": {
          "classValidatorShim": true,
          "introspectComments": true,
          "dtoFileNameSuffix": [".dto.ts", ".entity.ts"]
        }
      }
    ]
  }
}
```

### What the Plugin Does

Before (manual decorators):

```typescript
export class CreateUserDto {
  @ApiProperty({ description: 'User email' })
  @IsEmail()
  email: string;

  @ApiPropertyOptional({ description: 'User name' })
  @IsOptional()
  @IsString()
  name?: string;
}
```

After (with plugin - decorators added automatically):

```typescript
export class CreateUserDto {
  /**
   * User email
   */
  @IsEmail()
  email: string;

  /**
   * User name
   */
  @IsOptional()
  @IsString()
  name?: string;
}
```

The plugin reads JSDoc comments and TypeScript types to generate `@ApiProperty` decorators automatically.

---

## GraphQL Code-First

The code-first approach generates the GraphQL schema from TypeScript classes and decorators.

### Installation

```bash
npm install @nestjs/graphql @nestjs/apollo @apollo/server graphql
```

### Module Setup

```typescript
// app.module.ts
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { join } from 'path';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: join(process.cwd(), 'src/schema.gql'),
      sortSchema: true,
      playground: process.env.NODE_ENV !== 'production',
      context: ({ req }) => ({ req }),
    }),
    UsersModule,
  ],
})
export class AppModule {}
```

### Object Types

```typescript
// users/entities/user.entity.ts
import { ObjectType, Field, ID, Int } from '@nestjs/graphql';

@ObjectType()
export class User {
  @Field(() => ID)
  id: string;

  @Field()
  email: string;

  @Field({ nullable: true })
  name?: string;

  @Field(() => Int)
  age: number;

  @Field(() => [Post], { nullable: 'items' })
  posts: Post[];

  @Field()
  createdAt: Date;

  // Password not exposed in GraphQL
}

@ObjectType()
export class Post {
  @Field(() => ID)
  id: number;

  @Field()
  title: string;

  @Field({ nullable: true })
  content?: string;

  @Field(() => User)
  author: User;
}
```

### Input Types

```typescript
// users/dto/create-user.input.ts
import { InputType, Field } from '@nestjs/graphql';
import { IsEmail, MinLength } from 'class-validator';

@InputType()
export class CreateUserInput {
  @Field()
  @IsEmail()
  email: string;

  @Field()
  @MinLength(8)
  password: string;

  @Field({ nullable: true })
  name?: string;
}

@InputType()
export class UpdateUserInput {
  @Field({ nullable: true })
  name?: string;

  @Field({ nullable: true })
  @IsEmail()
  email?: string;
}
```

### Enums and Scalars

```typescript
import { registerEnumType } from '@nestjs/graphql';

export enum Role {
  USER = 'user',
  ADMIN = 'admin',
}

registerEnumType(Role, {
  name: 'Role',
  description: 'User roles',
});

// Usage
@ObjectType()
export class User {
  @Field(() => Role)
  role: Role;
}
```

---

## Resolvers

### Basic Resolver

```typescript
// users/users.resolver.ts
import { Resolver, Query, Mutation, Args, ID } from '@nestjs/graphql';
import { UseGuards } from '@nestjs/common';
import { User } from './entities/user.entity';
import { UsersService } from './users.service';
import { CreateUserInput } from './dto/create-user.input';
import { UpdateUserInput } from './dto/update-user.input';
import { GqlAuthGuard } from '../auth/guards/gql-auth.guard';
import { CurrentUser } from '../auth/decorators/current-user.decorator';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly usersService: UsersService) {}

  @Query(() => [User], { name: 'users' })
  findAll() {
    return this.usersService.findAll();
  }

  @Query(() => User, { name: 'user', nullable: true })
  findOne(@Args('id', { type: () => ID }) id: string) {
    return this.usersService.findOne(id);
  }

  @Query(() => User, { name: 'me' })
  @UseGuards(GqlAuthGuard)
  me(@CurrentUser() user: User) {
    return user;
  }

  @Mutation(() => User)
  createUser(@Args('input') input: CreateUserInput) {
    return this.usersService.create(input);
  }

  @Mutation(() => User)
  @UseGuards(GqlAuthGuard)
  updateUser(
    @Args('id', { type: () => ID }) id: string,
    @Args('input') input: UpdateUserInput,
  ) {
    return this.usersService.update(id, input);
  }

  @Mutation(() => Boolean)
  @UseGuards(GqlAuthGuard)
  removeUser(@Args('id', { type: () => ID }) id: string) {
    return this.usersService.remove(id);
  }
}
```

### Field Resolvers

```typescript
@Resolver(() => User)
export class UsersResolver {
  constructor(
    private readonly usersService: UsersService,
    private readonly postsService: PostsService,
  ) {}

  @ResolveField(() => [Post])
  async posts(@Parent() user: User) {
    return this.postsService.findByAuthorId(user.id);
  }

  @ResolveField(() => Int)
  async postsCount(@Parent() user: User) {
    return this.postsService.countByAuthorId(user.id);
  }
}
```

### DataLoader for N+1 Prevention

```typescript
// users/users.loader.ts
import * as DataLoader from 'dataloader';
import { Injectable, Scope } from '@nestjs/common';
import { UsersService } from './users.service';

@Injectable({ scope: Scope.REQUEST })
export class UsersLoader {
  constructor(private usersService: UsersService) {}

  readonly batchUsers = new DataLoader<string, User>(async (ids) => {
    const users = await this.usersService.findByIds([...ids]);
    const usersMap = new Map(users.map((user) => [user.id, user]));
    return ids.map((id) => usersMap.get(id));
  });
}

// posts/posts.resolver.ts
@Resolver(() => Post)
export class PostsResolver {
  constructor(private readonly usersLoader: UsersLoader) {}

  @ResolveField(() => User)
  async author(@Parent() post: Post) {
    return this.usersLoader.batchUsers.load(post.authorId);
  }
}
```

### GraphQL Auth Guard

```typescript
// auth/guards/gql-auth.guard.ts
import { ExecutionContext, Injectable } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  getRequest(context: ExecutionContext) {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req;
  }
}

// auth/decorators/current-user.decorator.ts (GraphQL version)
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

export const CurrentUser = createParamDecorator(
  (data: unknown, context: ExecutionContext) => {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req.user;
  },
);
```

---

## Subscriptions

Real-time updates via WebSocket connections.

### Setup

```typescript
// app.module.ts
@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: true,
      subscriptions: {
        'graphql-ws': true,
        'subscriptions-transport-ws': true, // Legacy support
      },
      context: ({ req, connection }) => {
        // For subscriptions, connection exists instead of req
        return { req: connection ? connection.context : req };
      },
    }),
  ],
})
export class AppModule {}
```

### PubSub Setup

```typescript
// pubsub/pubsub.module.ts
import { Global, Module } from '@nestjs/common';
import { PubSub } from 'graphql-subscriptions';

export const PUB_SUB = 'PUB_SUB';

@Global()
@Module({
  providers: [
    {
      provide: PUB_SUB,
      useValue: new PubSub(),
    },
  ],
  exports: [PUB_SUB],
})
export class PubSubModule {}
```

### Subscription Resolver

```typescript
// posts/posts.resolver.ts
import { Resolver, Mutation, Subscription, Args } from '@nestjs/graphql';
import { Inject } from '@nestjs/common';
import { PubSub } from 'graphql-subscriptions';
import { PUB_SUB } from '../pubsub/pubsub.module';

const POST_ADDED = 'postAdded';
const POST_UPDATED = 'postUpdated';

@Resolver(() => Post)
export class PostsResolver {
  constructor(
    private readonly postsService: PostsService,
    @Inject(PUB_SUB) private pubSub: PubSub,
  ) {}

  @Mutation(() => Post)
  async createPost(@Args('input') input: CreatePostInput) {
    const post = await this.postsService.create(input);
    this.pubSub.publish(POST_ADDED, { postAdded: post });
    return post;
  }

  @Subscription(() => Post)
  postAdded() {
    return this.pubSub.asyncIterableIterator(POST_ADDED);
  }

  // Subscription with filter
  @Subscription(() => Post, {
    filter: (payload, variables) => {
      return payload.postUpdated.authorId === variables.authorId;
    },
  })
  postUpdated(@Args('authorId') authorId: string) {
    return this.pubSub.asyncIterableIterator(POST_UPDATED);
  }

  // Subscription with resolve
  @Subscription(() => Post, {
    resolve: (payload) => payload.postAdded,
  })
  newPost() {
    return this.pubSub.asyncIterableIterator(POST_ADDED);
  }
}
```

### Redis PubSub for Production

```bash
npm install graphql-redis-subscriptions ioredis
```

```typescript
// pubsub/pubsub.module.ts
import { RedisPubSub } from 'graphql-redis-subscriptions';
import Redis from 'ioredis';

@Global()
@Module({
  providers: [
    {
      provide: PUB_SUB,
      useFactory: (configService: ConfigService) => {
        const options = {
          host: configService.get('REDIS_HOST'),
          port: configService.get('REDIS_PORT'),
        };

        return new RedisPubSub({
          publisher: new Redis(options),
          subscriber: new Redis(options),
        });
      },
      inject: [ConfigService],
    },
  ],
  exports: [PUB_SUB],
})
export class PubSubModule {}
```

### Client Usage

```graphql
# Subscribe to new posts
subscription {
  postAdded {
    id
    title
    author {
      name
    }
  }
}

# Subscribe with filter
subscription {
  postUpdated(authorId: "123") {
    id
    title
  }
}
```
