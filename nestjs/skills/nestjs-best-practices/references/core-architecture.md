# Core Architecture

## Table of Contents

- [Module Organization](#module-organization)
- [Provider Types](#provider-types)
- [Dependency Injection Patterns](#dependency-injection-patterns)
- [Circular Dependencies](#circular-dependencies)
- [Custom Decorators](#custom-decorators)
- [Dynamic Modules](#dynamic-modules)
- [ConfigModule Patterns](#configmodule-patterns)

---

## Module Organization

Modules are the fundamental building blocks in NestJS. They organize related functionality and establish clear boundaries.

### Feature Module Pattern

```typescript
// users/users.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    CommonModule,
  ],
  controllers: [UsersController],
  providers: [UsersService, UsersRepository],
  exports: [UsersService], // Make available to other modules
})
export class UsersModule {}
```

### Shared Module Pattern

For cross-cutting concerns used by multiple feature modules:

```typescript
// common/common.module.ts
@Module({
  providers: [
    LoggerService,
    CacheService,
    { provide: 'DATE_SERVICE', useClass: DateService },
  ],
  exports: [
    LoggerService,
    CacheService,
    'DATE_SERVICE',
  ],
})
export class CommonModule {}
```

### Core Module Pattern

For singleton services that should only be instantiated once:

```typescript
// core/core.module.ts
@Global() // Makes exports available everywhere without importing
@Module({
  providers: [ConfigService, DatabaseService],
  exports: [ConfigService, DatabaseService],
})
export class CoreModule {
  constructor(@Optional() @SkipSelf() parentModule?: CoreModule) {
    if (parentModule) {
      throw new Error('CoreModule is already loaded. Import it only in AppModule.');
    }
  }
}
```

### Module Re-exports

Re-export modules to simplify imports:

```typescript
@Module({
  imports: [CommonModule, LoggingModule],
  exports: [CommonModule, LoggingModule], // Re-export for consumers
})
export class SharedModule {}
```

---

## Provider Types

NestJS supports multiple ways to define providers for different use cases.

### Standard Provider (useClass)

The most common pattern - a class decorated with `@Injectable()`:

```typescript
// Shorthand
@Module({
  providers: [UsersService],
})

// Equivalent to
@Module({
  providers: [
    {
      provide: UsersService,
      useClass: UsersService,
    },
  ],
})
```

### Value Provider (useValue)

For injecting constant values, configuration objects, or mock objects:

```typescript
@Module({
  providers: [
    {
      provide: 'API_KEY',
      useValue: process.env.API_KEY,
    },
    {
      provide: 'CONFIG',
      useValue: {
        apiUrl: 'https://api.example.com',
        timeout: 5000,
      },
    },
  ],
})
export class AppModule {}

// Injection
@Injectable()
export class ApiService {
  constructor(
    @Inject('API_KEY') private apiKey: string,
    @Inject('CONFIG') private config: { apiUrl: string; timeout: number },
  ) {}
}
```

### Factory Provider (useFactory)

For dynamic provider creation with dependencies:

```typescript
@Module({
  providers: [
    {
      provide: 'DATABASE_CONNECTION',
      useFactory: async (configService: ConfigService) => {
        const options = configService.get<DatabaseConfig>('database');
        return createConnection(options);
      },
      inject: [ConfigService], // Dependencies for factory
    },
  ],
})
export class DatabaseModule {}
```

### Async Factory with Multiple Dependencies

```typescript
@Module({
  providers: [
    {
      provide: 'CACHE_MANAGER',
      useFactory: async (
        configService: ConfigService,
        logger: LoggerService,
      ) => {
        const redisUrl = configService.get('REDIS_URL');
        logger.log(`Connecting to Redis: ${redisUrl}`);
        return new CacheManager({ url: redisUrl });
      },
      inject: [ConfigService, LoggerService],
    },
  ],
})
export class CacheModule {}
```

### Existing Provider (useExisting)

Create an alias to an existing provider:

```typescript
@Module({
  providers: [
    UsersService,
    {
      provide: 'UserServiceAlias',
      useExisting: UsersService, // Points to same instance
    },
  ],
})
export class UsersModule {}
```

---

## Dependency Injection Patterns

### Constructor Injection (Recommended)

```typescript
@Injectable()
export class UsersService {
  constructor(
    private readonly usersRepository: UsersRepository,
    private readonly logger: LoggerService,
  ) {}
}
```

### Property Injection

Use when constructor injection isn't possible (e.g., base classes):

```typescript
@Injectable()
export class BaseService {
  @Inject(LoggerService)
  protected logger: LoggerService;
}
```

### Token-Based Injection

For non-class tokens (strings, symbols):

```typescript
// Define token
export const DATABASE_TOKEN = Symbol('DATABASE');

// Register provider
@Module({
  providers: [
    {
      provide: DATABASE_TOKEN,
      useFactory: () => new Database(),
    },
  ],
})

// Inject
@Injectable()
export class UsersService {
  constructor(
    @Inject(DATABASE_TOKEN) private db: Database,
  ) {}
}
```

### Optional Dependencies

```typescript
@Injectable()
export class NotificationService {
  constructor(
    @Optional() @Inject('SLACK_CLIENT') private slack?: SlackClient,
  ) {}

  notify(message: string) {
    if (this.slack) {
      this.slack.send(message);
    }
  }
}
```

### Scope-Based Injection

```typescript
// Request-scoped provider
@Injectable({ scope: Scope.REQUEST })
export class RequestContextService {
  constructor(@Inject(REQUEST) private request: Request) {}

  getUserId() {
    return this.request.user?.id;
  }
}

// Transient provider (new instance each injection)
@Injectable({ scope: Scope.TRANSIENT })
export class HelperService {}
```

---

## Circular Dependencies

Circular dependencies occur when Module A imports Module B, and Module B imports Module A.

### Symptoms

- `Error: Nest cannot create the <Module> instance`
- `Potential causes: A circular dependency`
- `Cannot read properties of undefined`

### Solution 1: forwardRef()

Wrap the circular reference in `forwardRef()`:

```typescript
// users/users.module.ts
@Module({
  imports: [forwardRef(() => PostsModule)],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

// posts/posts.module.ts
@Module({
  imports: [forwardRef(() => UsersModule)],
  providers: [PostsService],
  exports: [PostsService],
})
export class PostsModule {}
```

For service-level circular dependencies:

```typescript
@Injectable()
export class UsersService {
  constructor(
    @Inject(forwardRef(() => PostsService))
    private postsService: PostsService,
  ) {}
}
```

### Solution 2: Extract Shared Module

Better long-term solution - extract shared functionality:

```typescript
// BEFORE: Circular dependency
// UsersModule <-> PostsModule

// AFTER: Extract shared logic
@Module({
  providers: [UserPostLinkService],
  exports: [UserPostLinkService],
})
export class SharedModule {}

@Module({
  imports: [SharedModule],
  providers: [UsersService],
})
export class UsersModule {}

@Module({
  imports: [SharedModule],
  providers: [PostsService],
})
export class PostsModule {}
```

### Solution 3: Event-Based Decoupling

Use EventEmitter2 to decouple modules:

```typescript
// Install: npm install @nestjs/event-emitter

// users.service.ts
@Injectable()
export class UsersService {
  constructor(private eventEmitter: EventEmitter2) {}

  async deleteUser(id: number) {
    await this.usersRepository.delete(id);
    this.eventEmitter.emit('user.deleted', { userId: id });
  }
}

// posts.service.ts
@Injectable()
export class PostsService {
  @OnEvent('user.deleted')
  handleUserDeleted(payload: { userId: number }) {
    this.postsRepository.deleteByUserId(payload.userId);
  }
}
```

---

## Custom Decorators

### Parameter Decorators

Extract data from the request:

```typescript
// decorators/user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: keyof User | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);

// Usage
@Get('profile')
getProfile(@CurrentUser() user: User) {
  return user;
}

@Get('email')
getEmail(@CurrentUser('email') email: string) {
  return { email };
}
```

### Combining Decorators

Create a single decorator that applies multiple decorators:

```typescript
import { applyDecorators, UseGuards, SetMetadata } from '@nestjs/common';
import { ApiBearerAuth, ApiUnauthorizedResponse } from '@nestjs/swagger';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(JwtAuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}

// Usage
@Post()
@Auth(Role.Admin)
create(@Body() dto: CreateUserDto) {}
```

### Class Decorators with Metadata

```typescript
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// Usage
@Controller('admin')
@Roles('admin', 'superadmin')
export class AdminController {}

// Access metadata in guard
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!roles) return true;

    const request = context.switchToHttp().getRequest();
    return roles.includes(request.user?.role);
  }
}
```

---

## Dynamic Modules

Dynamic modules allow configurable module behavior.

### forRoot Pattern (Singleton Config)

```typescript
// database/database.module.ts
@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseOptions): DynamicModule {
    return {
      module: DatabaseModule,
      global: true, // Optional: make globally available
      providers: [
        {
          provide: 'DATABASE_OPTIONS',
          useValue: options,
        },
        {
          provide: 'DATABASE_CONNECTION',
          useFactory: async (opts: DatabaseOptions) => {
            return createConnection(opts);
          },
          inject: ['DATABASE_OPTIONS'],
        },
      ],
      exports: ['DATABASE_CONNECTION'],
    };
  }
}

// app.module.ts
@Module({
  imports: [
    DatabaseModule.forRoot({
      host: 'localhost',
      port: 5432,
      database: 'myapp',
    }),
  ],
})
export class AppModule {}
```

### forRootAsync Pattern (Async Config)

```typescript
@Module({})
export class DatabaseModule {
  static forRootAsync(options: DatabaseAsyncOptions): DynamicModule {
    return {
      module: DatabaseModule,
      imports: options.imports || [],
      providers: [
        {
          provide: 'DATABASE_OPTIONS',
          useFactory: options.useFactory,
          inject: options.inject || [],
        },
        {
          provide: 'DATABASE_CONNECTION',
          useFactory: async (opts: DatabaseOptions) => createConnection(opts),
          inject: ['DATABASE_OPTIONS'],
        },
      ],
      exports: ['DATABASE_CONNECTION'],
    };
  }
}

// Usage with ConfigService
@Module({
  imports: [
    DatabaseModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        host: config.get('DB_HOST'),
        port: config.get('DB_PORT'),
        database: config.get('DB_NAME'),
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

### forFeature Pattern (Per-Module Config)

```typescript
@Module({})
export class TypeOrmModule {
  static forFeature(entities: EntityClassOrSchema[]): DynamicModule {
    const providers = entities.map((entity) => ({
      provide: getRepositoryToken(entity),
      useFactory: (dataSource: DataSource) => dataSource.getRepository(entity),
      inject: [DataSource],
    }));

    return {
      module: TypeOrmModule,
      providers,
      exports: providers,
    };
  }
}

// Usage in feature module
@Module({
  imports: [TypeOrmModule.forFeature([User, Profile])],
  providers: [UsersService],
})
export class UsersModule {}
```

---

## ConfigModule Patterns

### Basic Setup

```typescript
// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true, // No need to import in every module
      envFilePath: ['.env.local', '.env'],
      cache: true, // Cache env vars for performance
    }),
  ],
})
export class AppModule {}
```

### With Validation

```typescript
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test')
          .default('development'),
        PORT: Joi.number().default(3000),
        DATABASE_URL: Joi.string().required(),
        JWT_SECRET: Joi.string().required().min(32),
      }),
      validationOptions: {
        abortEarly: false, // Show all validation errors
      },
    }),
  ],
})
export class AppModule {}
```

### Custom Configuration Files

```typescript
// config/database.config.ts
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT, 10) || 5432,
  name: process.env.DB_NAME,
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
}));

// app.module.ts
import databaseConfig from './config/database.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [databaseConfig],
    }),
  ],
})
export class AppModule {}

// Usage
@Injectable()
export class DatabaseService {
  constructor(private configService: ConfigService) {
    const dbHost = this.configService.get<string>('database.host');
    // Or get entire namespace
    const dbConfig = this.configService.get('database');
  }
}
```

### Type-Safe Configuration

```typescript
// config/configuration.ts
export interface AppConfig {
  port: number;
  database: {
    host: string;
    port: number;
  };
}

export default (): AppConfig => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT, 10) || 5432,
  },
});

// Usage with type safety
const port = this.configService.get<number>('port');
const dbConfig = this.configService.get<AppConfig['database']>('database');
```
