# Request Lifecycle

## Table of Contents

- [Request Lifecycle Overview](#request-lifecycle-overview)
- [Middleware](#middleware)
- [Guards](#guards)
- [Interceptors](#interceptors)
- [Pipes](#pipes)
- [Exception Filters](#exception-filters)
- [Execution Order](#execution-order)

---

## Request Lifecycle Overview

Every incoming request flows through a specific pipeline of components:

```
Request
   │
   ▼
┌─────────────────┐
│   Middleware    │  ← Express/Fastify middleware
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Guards      │  ← Authorization (can proceed?)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Interceptors   │  ← Before handler (transform request)
│    (Before)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     Pipes       │  ← Validation & transformation
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Controller    │  ← Route handler
│    Handler      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Interceptors   │  ← After handler (transform response)
│    (After)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Exception     │  ← Error handling (if exception thrown)
│    Filters      │
└────────┬────────┘
         │
         ▼
Response
```

---

## Middleware

Middleware runs before the route handler and has access to the raw request/response objects.

### Class Middleware

```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const start = Date.now();

    res.on('finish', () => {
      const duration = Date.now() - start;
      console.log(`${req.method} ${req.url} ${res.statusCode} - ${duration}ms`);
    });

    next();
  }
}
```

### Functional Middleware

For simple middleware without dependencies:

```typescript
import { Request, Response, NextFunction } from 'express';

export function correlationIdMiddleware(
  req: Request,
  res: Response,
  next: NextFunction,
) {
  const correlationId = req.headers['x-correlation-id'] || crypto.randomUUID();
  req['correlationId'] = correlationId;
  res.setHeader('x-correlation-id', correlationId);
  next();
}
```

### Applying Middleware

```typescript
// In module
@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware, correlationIdMiddleware)
      .exclude(
        { path: 'health', method: RequestMethod.GET },
        { path: 'metrics', method: RequestMethod.GET },
      )
      .forRoutes('*'); // Apply to all routes
  }
}

// Route-specific
consumer
  .apply(AuthMiddleware)
  .forRoutes(
    { path: 'users', method: RequestMethod.ALL },
    { path: 'posts', method: RequestMethod.POST },
  );

// Controller-specific
consumer
  .apply(AdminMiddleware)
  .forRoutes(AdminController);
```

### Global Middleware (main.ts)

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Apply Express middleware globally
  app.use(helmet());
  app.use(compression());
  app.use(correlationIdMiddleware);

  await app.listen(3000);
}
```

---

## Guards

Guards determine whether a request will be handled by the route handler. Use for authorization logic.

### Basic Guard

```typescript
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean | Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.split(' ')[1];

    if (!token) {
      throw new UnauthorizedException('No token provided');
    }

    try {
      const payload = this.jwtService.verify(token);
      request.user = payload;
      return true;
    } catch {
      throw new UnauthorizedException('Invalid token');
    }
  }
}
```

### Role-Based Guard

```typescript
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredRoles) {
      return true; // No roles required
    }

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}

// Usage
@Controller('admin')
@UseGuards(AuthGuard, RolesGuard)
export class AdminController {
  @Get('users')
  @Roles('admin', 'superadmin')
  getUsers() {}
}
```

### Applying Guards

```typescript
// Method level
@Get('profile')
@UseGuards(AuthGuard)
getProfile() {}

// Controller level
@Controller('users')
@UseGuards(AuthGuard)
export class UsersController {}

// Global level (main.ts)
app.useGlobalGuards(new AuthGuard());

// Global with DI (app.module.ts)
@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: AuthGuard,
    },
  ],
})
export class AppModule {}
```

### ExecutionContext Utilities

```typescript
@Injectable()
export class MultiContextGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    // Get context type
    const type = context.getType(); // 'http' | 'rpc' | 'ws' | 'graphql'

    if (type === 'http') {
      const request = context.switchToHttp().getRequest();
      const response = context.switchToHttp().getResponse();
    }

    if (type === 'ws') {
      const client = context.switchToWs().getClient();
      const data = context.switchToWs().getData();
    }

    if (type === 'rpc') {
      const data = context.switchToRpc().getData();
      const ctx = context.switchToRpc().getContext();
    }

    // Get handler and class metadata
    const handler = context.getHandler(); // Method reference
    const controller = context.getClass(); // Class reference

    return true;
  }
}
```

---

## Interceptors

Interceptors can transform the result returned from a function, transform exceptions, extend basic function behavior, or override a function entirely.

### Response Transformation Interceptor

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
  statusCode: number;
  timestamp: string;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, Response<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<Response<T>> {
    return next.handle().pipe(
      map((data) => ({
        data,
        statusCode: context.switchToHttp().getResponse().statusCode,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

### Logging Interceptor

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger(LoggingInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url, body } = request;
    const start = Date.now();

    this.logger.log(`Incoming ${method} ${url}`);

    return next.handle().pipe(
      tap({
        next: (data) => {
          const duration = Date.now() - start;
          this.logger.log(`${method} ${url} completed in ${duration}ms`);
        },
        error: (error) => {
          const duration = Date.now() - start;
          this.logger.error(`${method} ${url} failed in ${duration}ms`, error);
        },
      }),
    );
  }
}
```

### Timeout Interceptor

```typescript
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  constructor(private readonly timeoutMs: number = 5000) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(this.timeoutMs),
      catchError((err) => {
        if (err instanceof TimeoutError) {
          throw new RequestTimeoutException('Request timed out');
        }
        return throwError(() => err);
      }),
    );
  }
}
```

### Cache Interceptor

```typescript
@Injectable()
export class CacheInterceptor implements NestInterceptor {
  constructor(private cacheService: CacheService) {}

  async intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();

    // Only cache GET requests
    if (request.method !== 'GET') {
      return next.handle();
    }

    const cacheKey = request.url;
    const cached = await this.cacheService.get(cacheKey);

    if (cached) {
      return of(cached);
    }

    return next.handle().pipe(
      tap((data) => {
        this.cacheService.set(cacheKey, data, 60); // TTL 60 seconds
      }),
    );
  }
}
```

### Applying Interceptors

```typescript
// Method level
@Get()
@UseInterceptors(LoggingInterceptor)
findAll() {}

// Controller level
@Controller('users')
@UseInterceptors(TransformInterceptor)
export class UsersController {}

// Global level (main.ts)
app.useGlobalInterceptors(new LoggingInterceptor());

// Global with DI
@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: TransformInterceptor,
    },
  ],
})
export class AppModule {}
```

---

## Pipes

Pipes transform input data or validate it before it reaches the route handler.

### Built-in Pipes

```typescript
import {
  ParseIntPipe,
  ParseBoolPipe,
  ParseArrayPipe,
  ParseUUIDPipe,
  ParseEnumPipe,
  DefaultValuePipe,
  ParseFloatPipe,
} from '@nestjs/common';

@Controller('users')
export class UsersController {
  // ParseIntPipe
  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findOne(id);
  }

  // ParseUUIDPipe with version
  @Get(':uuid')
  findByUuid(@Param('uuid', new ParseUUIDPipe({ version: '4' })) uuid: string) {}

  // ParseBoolPipe
  @Get()
  findAll(@Query('active', ParseBoolPipe) active: boolean) {}

  // DefaultValuePipe
  @Get()
  findPaginated(
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
    @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number,
  ) {}

  // ParseArrayPipe
  @Get()
  findByIds(
    @Query('ids', new ParseArrayPipe({ items: Number, separator: ',' }))
    ids: number[],
  ) {}

  // ParseEnumPipe
  @Get()
  findByStatus(
    @Query('status', new ParseEnumPipe(UserStatus)) status: UserStatus,
  ) {}
}
```

### Custom Validation Pipe

```typescript
import {
  PipeTransform,
  Injectable,
  BadRequestException,
} from '@nestjs/common';

@Injectable()
export class ParseDatePipe implements PipeTransform<string, Date> {
  transform(value: string): Date {
    const date = new Date(value);

    if (isNaN(date.getTime())) {
      throw new BadRequestException(`Invalid date: ${value}`);
    }

    return date;
  }
}

// Usage
@Get()
findByDate(@Query('date', ParseDatePipe) date: Date) {}
```

### Schema Validation Pipe (Zod)

```typescript
import { z, ZodSchema } from 'zod';

@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown) {
    const result = this.schema.safeParse(value);

    if (!result.success) {
      throw new BadRequestException({
        message: 'Validation failed',
        errors: result.error.format(),
      });
    }

    return result.data;
  }
}

// Usage
const createUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

@Post()
create(
  @Body(new ZodValidationPipe(createUserSchema))
  createUserDto: z.infer<typeof createUserSchema>,
) {}
```

### Applying Pipes

```typescript
// Parameter level
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {}

// Method level
@Post()
@UsePipes(ValidationPipe)
create(@Body() dto: CreateUserDto) {}

// Controller level
@Controller('users')
@UsePipes(new ValidationPipe({ transform: true }))
export class UsersController {}

// Global level (main.ts)
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true,
  }),
);

// Global with DI
@Module({
  providers: [
    {
      provide: APP_PIPE,
      useValue: new ValidationPipe({ whitelist: true }),
    },
  ],
})
export class AppModule {}
```

---

## Exception Filters

Exception filters handle errors thrown during request processing.

### Built-in HTTP Exceptions

```typescript
import {
  BadRequestException,
  UnauthorizedException,
  ForbiddenException,
  NotFoundException,
  ConflictException,
  InternalServerErrorException,
  NotImplementedException,
  BadGatewayException,
  ServiceUnavailableException,
  GatewayTimeoutException,
} from '@nestjs/common';

// Throwing exceptions
throw new NotFoundException('User not found');
throw new BadRequestException('Invalid email format');
throw new UnauthorizedException();

// With custom response
throw new BadRequestException({
  message: 'Validation failed',
  errors: ['email is required', 'password must be at least 8 characters'],
});
```

### Custom Exception Filter

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();

    const error =
      typeof exceptionResponse === 'string'
        ? { message: exceptionResponse }
        : (exceptionResponse as object);

    const errorResponse = {
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      method: request.method,
      ...error,
    };

    this.logger.error(
      `${request.method} ${request.url} ${status}`,
      exception.stack,
    );

    response.status(status).json(errorResponse);
  }
}
```

### Catch-All Exception Filter

```typescript
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const message =
      exception instanceof HttpException
        ? exception.message
        : 'Internal server error';

    this.logger.error(
      `${request.method} ${request.url}`,
      exception instanceof Error ? exception.stack : 'Unknown error',
    );

    response.status(status).json({
      statusCode: status,
      message,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

### Custom Exception Classes

```typescript
export class BusinessException extends HttpException {
  constructor(
    public readonly code: string,
    message: string,
    status: HttpStatus = HttpStatus.BAD_REQUEST,
  ) {
    super({ code, message }, status);
  }
}

@Catch(BusinessException)
export class BusinessExceptionFilter implements ExceptionFilter {
  catch(exception: BusinessException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    response.status(exception.getStatus()).json({
      code: exception.code,
      message: exception.message,
      timestamp: new Date().toISOString(),
    });
  }
}

// Usage
throw new BusinessException('USER_EXISTS', 'User already exists');
```

### Applying Exception Filters

```typescript
// Method level
@Get(':id')
@UseFilters(HttpExceptionFilter)
findOne(@Param('id') id: string) {}

// Controller level
@Controller('users')
@UseFilters(new HttpExceptionFilter())
export class UsersController {}

// Global level (main.ts)
app.useGlobalFilters(new AllExceptionsFilter());

// Global with DI
@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: AllExceptionsFilter,
    },
  ],
})
export class AppModule {}
```

---

## Execution Order

Understanding the exact order of execution is critical for debugging.

### Global Binding Order

When multiple components are registered globally:

1. **Filters** - Execute in reverse order of registration (last registered = first executed)
2. **Pipes** - Execute in order of registration
3. **Guards** - Execute in order of registration
4. **Interceptors** - Execute in order of registration (before), reverse for after

### Complete Request Flow

```
1. Incoming request
2. Globally bound middleware
3. Module-bound middleware
4. Global guards
5. Controller guards
6. Route guards
7. Global interceptors (before)
8. Controller interceptors (before)
9. Route interceptors (before)
10. Global pipes
11. Controller pipes
12. Route pipes
13. Route parameter pipes
14. Controller method (handler)
15. Route interceptors (after)
16. Controller interceptors (after)
17. Global interceptors (after)
18. Exception filters (if exception thrown)
    - Route filters
    - Controller filters
    - Global filters
19. Server response
```

### Example: Multiple Components

```typescript
// Global registration (main.ts)
app.useGlobalGuards(new Guard1(), new Guard2());
app.useGlobalInterceptors(new Interceptor1(), new Interceptor2());
app.useGlobalPipes(new Pipe1(), new Pipe2());
app.useGlobalFilters(new Filter1(), new Filter2());

// Controller
@Controller('users')
@UseGuards(Guard3)
@UseInterceptors(Interceptor3)
@UsePipes(Pipe3)
@UseFilters(Filter3)
export class UsersController {
  @Get(':id')
  @UseGuards(Guard4)
  @UseInterceptors(Interceptor4)
  @UsePipes(Pipe4)
  findOne(@Param('id', Pipe5) id: number) {}
}

// Execution order for GET /users/1:
// Guards: Guard1 → Guard2 → Guard3 → Guard4
// Interceptors (before): Interceptor1 → Interceptor2 → Interceptor3 → Interceptor4
// Pipes: Pipe1 → Pipe2 → Pipe3 → Pipe4 → Pipe5
// Handler
// Interceptors (after): Interceptor4 → Interceptor3 → Interceptor2 → Interceptor1
// Exception filters (if error): Filter3 → Filter2 → Filter1
```
