# Testing Patterns

## Table of Contents

- [Testing Module Setup](#testing-module-setup)
- [Unit Testing Services](#unit-testing-services)
- [Unit Testing Controllers](#unit-testing-controllers)
- [E2E Testing](#e2e-testing)
- [Mocking Providers](#mocking-providers)
- [Database Testing](#database-testing)

---

## Testing Module Setup

The `@nestjs/testing` package provides utilities for creating isolated test modules.

### Basic Setup

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { UsersRepository } from './users.repository';

describe('UsersService', () => {
  let service: UsersService;
  let repository: UsersRepository;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: UsersRepository,
          useValue: {
            findAll: jest.fn(),
            findOne: jest.fn(),
            create: jest.fn(),
            update: jest.fn(),
            delete: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repository = module.get<UsersRepository>(UsersRepository);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
```

### Symptoms: Test Module Not Compiling

- "Nest can't resolve dependencies" error
- Undefined provider errors
- Circular dependency errors in tests

### Fix

Ensure all dependencies are provided or mocked:

```typescript
const module: TestingModule = await Test.createTestingModule({
  providers: [
    UsersService,
    // Mock all dependencies
    { provide: UsersRepository, useValue: mockRepository },
    { provide: ConfigService, useValue: mockConfigService },
    { provide: 'CACHE_MANAGER', useValue: mockCacheManager },
  ],
}).compile();
```

---

## Unit Testing Services

### Service with Repository

```typescript
// users/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { NotFoundException } from '@nestjs/common';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';

describe('UsersService', () => {
  let service: UsersService;
  let repository: Repository<User>;

  const mockUser: User = {
    id: '1',
    email: 'test@example.com',
    password: 'hashedPassword',
    name: 'Test User',
    role: 'user',
    isActive: true,
    posts: [],
    createdAt: new Date(),
    updatedAt: new Date(),
  };

  const mockRepository = {
    find: jest.fn(),
    findOne: jest.fn(),
    create: jest.fn(),
    save: jest.fn(),
    update: jest.fn(),
    delete: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useValue: mockRepository,
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repository = module.get<Repository<User>>(getRepositoryToken(User));

    // Reset mocks before each test
    jest.clearAllMocks();
  });

  describe('findAll', () => {
    it('should return an array of users', async () => {
      mockRepository.find.mockResolvedValue([mockUser]);

      const result = await service.findAll();

      expect(result).toEqual([mockUser]);
      expect(mockRepository.find).toHaveBeenCalledWith({
        relations: ['posts'],
        order: { createdAt: 'DESC' },
      });
    });
  });

  describe('findOne', () => {
    it('should return a user when found', async () => {
      mockRepository.findOne.mockResolvedValue(mockUser);

      const result = await service.findOne('1');

      expect(result).toEqual(mockUser);
      expect(mockRepository.findOne).toHaveBeenCalledWith({
        where: { id: '1' },
        relations: ['posts'],
      });
    });

    it('should throw NotFoundException when user not found', async () => {
      mockRepository.findOne.mockResolvedValue(null);

      await expect(service.findOne('999')).rejects.toThrow(NotFoundException);
    });
  });

  describe('create', () => {
    it('should create and return a user', async () => {
      const createUserDto = { email: 'new@example.com', password: 'password' };
      mockRepository.create.mockReturnValue(mockUser);
      mockRepository.save.mockResolvedValue(mockUser);

      const result = await service.create(createUserDto);

      expect(result).toEqual(mockUser);
      expect(mockRepository.create).toHaveBeenCalledWith(createUserDto);
      expect(mockRepository.save).toHaveBeenCalled();
    });
  });
});
```

### Testing Async Methods

```typescript
describe('async operations', () => {
  it('should handle async errors', async () => {
    mockRepository.findOne.mockRejectedValue(new Error('Database error'));

    await expect(service.findOne('1')).rejects.toThrow('Database error');
  });

  it('should handle concurrent operations', async () => {
    mockRepository.findOne.mockResolvedValue(mockUser);

    const results = await Promise.all([
      service.findOne('1'),
      service.findOne('2'),
      service.findOne('3'),
    ]);

    expect(mockRepository.findOne).toHaveBeenCalledTimes(3);
  });
});
```

---

## Unit Testing Controllers

### Controller with Service

```typescript
// users/users.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';

describe('UsersController', () => {
  let controller: UsersController;
  let service: UsersService;

  const mockUser = {
    id: '1',
    email: 'test@example.com',
    name: 'Test User',
  };

  const mockUsersService = {
    findAll: jest.fn(),
    findOne: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
    remove: jest.fn(),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        {
          provide: UsersService,
          useValue: mockUsersService,
        },
      ],
    }).compile();

    controller = module.get<UsersController>(UsersController);
    service = module.get<UsersService>(UsersService);

    jest.clearAllMocks();
  });

  describe('findAll', () => {
    it('should return an array of users', async () => {
      mockUsersService.findAll.mockResolvedValue([mockUser]);

      const result = await controller.findAll();

      expect(result).toEqual([mockUser]);
      expect(service.findAll).toHaveBeenCalled();
    });
  });

  describe('create', () => {
    it('should create a user', async () => {
      const createUserDto: CreateUserDto = {
        email: 'new@example.com',
        password: 'password123',
      };
      mockUsersService.create.mockResolvedValue({ id: '2', ...createUserDto });

      const result = await controller.create(createUserDto);

      expect(result).toHaveProperty('id', '2');
      expect(service.create).toHaveBeenCalledWith(createUserDto);
    });
  });
});
```

### Testing Guards on Controllers

```typescript
describe('UsersController with guards', () => {
  let controller: UsersController;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        { provide: UsersService, useValue: mockUsersService },
      ],
    })
      .overrideGuard(JwtAuthGuard)
      .useValue({ canActivate: () => true })
      .overrideGuard(RolesGuard)
      .useValue({ canActivate: () => true })
      .compile();

    controller = module.get<UsersController>(UsersController);
  });

  it('should allow access when guards pass', async () => {
    mockUsersService.findAll.mockResolvedValue([mockUser]);

    const result = await controller.findAll();

    expect(result).toBeDefined();
  });
});
```

---

## E2E Testing

### Basic E2E Setup

```typescript
// test/app.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('AppController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();

    // Apply same pipes as in main.ts
    app.useGlobalPipes(
      new ValidationPipe({
        whitelist: true,
        forbidNonWhitelisted: true,
        transform: true,
      }),
    );

    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('/users (GET)', () => {
    it('should return users array', () => {
      return request(app.getHttpServer())
        .get('/users')
        .expect(200)
        .expect((res) => {
          expect(Array.isArray(res.body)).toBe(true);
        });
    });
  });

  describe('/users (POST)', () => {
    it('should create a user', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({
          email: 'test@example.com',
          password: 'password123',
        })
        .expect(201)
        .expect((res) => {
          expect(res.body).toHaveProperty('id');
          expect(res.body.email).toBe('test@example.com');
        });
    });

    it('should reject invalid email', () => {
      return request(app.getHttpServer())
        .post('/users')
        .send({
          email: 'invalid-email',
          password: 'password123',
        })
        .expect(400);
    });
  });
});
```

### E2E with Authentication

```typescript
describe('Auth E2E', () => {
  let app: INestApplication;
  let accessToken: string;

  beforeAll(async () => {
    // ... setup
  });

  describe('Authentication flow', () => {
    it('should register a new user', async () => {
      const response = await request(app.getHttpServer())
        .post('/auth/register')
        .send({
          email: 'newuser@example.com',
          password: 'password123',
        })
        .expect(201);

      expect(response.body).toHaveProperty('accessToken');
      accessToken = response.body.accessToken;
    });

    it('should login with credentials', async () => {
      const response = await request(app.getHttpServer())
        .post('/auth/login')
        .send({
          email: 'newuser@example.com',
          password: 'password123',
        })
        .expect(200);

      expect(response.body).toHaveProperty('accessToken');
      accessToken = response.body.accessToken;
    });

    it('should access protected route with token', async () => {
      return request(app.getHttpServer())
        .get('/users/profile')
        .set('Authorization', `Bearer ${accessToken}`)
        .expect(200);
    });

    it('should reject request without token', async () => {
      return request(app.getHttpServer())
        .get('/users/profile')
        .expect(401);
    });
  });
});
```

### Symptoms: E2E Tests Sharing State

- Tests pass individually but fail when run together
- Data from previous tests affecting current test
- Flaky tests due to database state

### Fix: Database Isolation

```typescript
// test/test-utils.ts
import { PrismaService } from '../src/prisma/prisma.service';

export async function cleanDatabase(prisma: PrismaService) {
  // Delete in order to respect foreign keys
  await prisma.post.deleteMany();
  await prisma.user.deleteMany();
}

// In test file
describe('Users E2E', () => {
  let app: INestApplication;
  let prisma: PrismaService;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    prisma = app.get(PrismaService);
    await app.init();
  });

  beforeEach(async () => {
    await cleanDatabase(prisma);
  });

  afterAll(async () => {
    await cleanDatabase(prisma);
    await app.close();
  });
});
```

### Using Transactions for Isolation

```typescript
// test/test-utils.ts
export function withTransaction(
  prisma: PrismaService,
  fn: (tx: PrismaClient) => Promise<void>,
) {
  return async () => {
    await prisma.$transaction(async (tx) => {
      await fn(tx as any);
      throw new Error('ROLLBACK'); // Force rollback
    }).catch((e) => {
      if (e.message !== 'ROLLBACK') throw e;
    });
  };
}
```

---

## Mocking Providers

### Mock Factory

```typescript
// test/mocks/users.service.mock.ts
export const mockUsersService = () => ({
  findAll: jest.fn(),
  findOne: jest.fn(),
  create: jest.fn(),
  update: jest.fn(),
  remove: jest.fn(),
});

export type MockUsersService = ReturnType<typeof mockUsersService>;

// Usage
const module = await Test.createTestingModule({
  providers: [
    {
      provide: UsersService,
      useFactory: mockUsersService,
    },
  ],
}).compile();
```

### Override Provider

```typescript
describe('with overridden provider', () => {
  let module: TestingModule;

  beforeEach(async () => {
    module = await Test.createTestingModule({
      imports: [UsersModule],
    })
      .overrideProvider(UsersService)
      .useValue({
        findAll: jest.fn().mockResolvedValue([]),
        findOne: jest.fn().mockResolvedValue(null),
      })
      .compile();
  });
});
```

### Mocking External Services

```typescript
// Mocking HttpService
const mockHttpService = {
  get: jest.fn(),
  post: jest.fn(),
  axiosRef: {
    get: jest.fn(),
    post: jest.fn(),
  },
};

const module = await Test.createTestingModule({
  providers: [
    ExternalApiService,
    {
      provide: HttpService,
      useValue: mockHttpService,
    },
  ],
}).compile();

// Test
it('should fetch external data', async () => {
  mockHttpService.get.mockReturnValue(
    of({ data: { result: 'success' } }),
  );

  const result = await service.fetchData();

  expect(result).toEqual({ result: 'success' });
});
```

### Mocking ConfigService

```typescript
const mockConfigService = {
  get: jest.fn((key: string) => {
    const config = {
      JWT_SECRET: 'test-secret',
      DATABASE_URL: 'postgresql://test:test@localhost/test',
      NODE_ENV: 'test',
    };
    return config[key];
  }),
};

const module = await Test.createTestingModule({
  providers: [
    AuthService,
    { provide: ConfigService, useValue: mockConfigService },
  ],
}).compile();
```

---

## Database Testing

### In-Memory Database (SQLite)

```typescript
// test/test-database.module.ts
import { TypeOrmModule } from '@nestjs/typeorm';

export const TestDatabaseModule = TypeOrmModule.forRoot({
  type: 'sqlite',
  database: ':memory:',
  entities: [User, Post],
  synchronize: true,
  dropSchema: true,
});

// Usage
describe('UsersService Integration', () => {
  let module: TestingModule;
  let service: UsersService;

  beforeEach(async () => {
    module = await Test.createTestingModule({
      imports: [
        TestDatabaseModule,
        TypeOrmModule.forFeature([User]),
      ],
      providers: [UsersService],
    }).compile();

    service = module.get<UsersService>(UsersService);
  });

  afterEach(async () => {
    await module.close();
  });

  it('should create and retrieve a user', async () => {
    const created = await service.create({
      email: 'test@example.com',
      password: 'password',
    });

    const found = await service.findOne(created.id);

    expect(found.email).toBe('test@example.com');
  });
});
```

### Test Containers (Docker)

```typescript
// test/setup-test-db.ts
import { PostgreSqlContainer } from '@testcontainers/postgresql';

let container: StartedPostgreSqlContainer;

beforeAll(async () => {
  container = await new PostgreSqlContainer()
    .withDatabase('test_db')
    .withUsername('test')
    .withPassword('test')
    .start();

  process.env.DATABASE_URL = container.getConnectionUri();
}, 60000);

afterAll(async () => {
  await container.stop();
});
```

### Symptoms: Test Database Connection Issues

- "Connection refused" errors
- Tests hanging on database operations
- "Too many connections" errors

### Fix: Proper Connection Management

```typescript
describe('Database Tests', () => {
  let app: INestApplication;
  let dataSource: DataSource;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    dataSource = app.get(DataSource);
    await app.init();
  });

  afterAll(async () => {
    // Close connections properly
    if (dataSource.isInitialized) {
      await dataSource.destroy();
    }
    await app.close();
  });

  beforeEach(async () => {
    // Use query runner for better control
    const queryRunner = dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    // Store for cleanup
    (global as any).__QUERY_RUNNER__ = queryRunner;
  });

  afterEach(async () => {
    const queryRunner = (global as any).__QUERY_RUNNER__;
    if (queryRunner) {
      await queryRunner.rollbackTransaction();
      await queryRunner.release();
    }
  });
});
```

### Repository Mock Helpers

```typescript
// test/helpers/repository.mock.ts
export function createMockRepository<T>(): MockType<Repository<T>> {
  return {
    find: jest.fn(),
    findOne: jest.fn(),
    findOneBy: jest.fn(),
    save: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
    delete: jest.fn(),
    createQueryBuilder: jest.fn(() => ({
      where: jest.fn().mockReturnThis(),
      andWhere: jest.fn().mockReturnThis(),
      orderBy: jest.fn().mockReturnThis(),
      skip: jest.fn().mockReturnThis(),
      take: jest.fn().mockReturnThis(),
      getMany: jest.fn(),
      getOne: jest.fn(),
    })),
  };
}

type MockType<T> = {
  [P in keyof T]?: jest.Mock;
};
```
