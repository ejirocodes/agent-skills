# Database Integration

## Table of Contents

- [TypeORM](#typeorm)
- [Prisma](#prisma)
- [Drizzle ORM](#drizzle-orm)
- [Common Patterns](#common-patterns)

---

## TypeORM

TypeORM is the most established ORM in the NestJS ecosystem with first-party support.

### Installation

```bash
npm install @nestjs/typeorm typeorm pg # PostgreSQL
# or
npm install @nestjs/typeorm typeorm mysql2 # MySQL
```

### Module Setup

```typescript
// app.module.ts
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        host: config.get('DB_HOST'),
        port: config.get('DB_PORT'),
        username: config.get('DB_USER'),
        password: config.get('DB_PASSWORD'),
        database: config.get('DB_NAME'),
        entities: [__dirname + '/**/*.entity{.ts,.js}'],
        synchronize: config.get('NODE_ENV') === 'development', // Never in production
        logging: config.get('NODE_ENV') === 'development',
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

### Entity Definition

```typescript
// users/entities/user.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
  ManyToOne,
  Index,
} from 'typeorm';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  @Index()
  email: string;

  @Column()
  password: string;

  @Column({ nullable: true })
  name: string;

  @Column({ type: 'enum', enum: ['user', 'admin'], default: 'user' })
  role: string;

  @Column({ default: true })
  isActive: boolean;

  @OneToMany(() => Post, (post) => post.author)
  posts: Post[];

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

### Relations

```typescript
// posts/entities/post.entity.ts
@Entity('posts')
export class Post {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;

  @Column('text')
  content: string;

  @ManyToOne(() => User, (user) => user.posts, { onDelete: 'CASCADE' })
  author: User;

  @Column()
  authorId: string;

  @ManyToMany(() => Tag)
  @JoinTable({
    name: 'posts_tags',
    joinColumn: { name: 'postId', referencedColumnName: 'id' },
    inverseJoinColumn: { name: 'tagId', referencedColumnName: 'id' },
  })
  tags: Tag[];
}
```

### Repository Pattern

```typescript
// users/users.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

// users/users.service.ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  async findAll(): Promise<User[]> {
    return this.usersRepository.find({
      relations: ['posts'],
      order: { createdAt: 'DESC' },
    });
  }

  async findOne(id: string): Promise<User> {
    const user = await this.usersRepository.findOne({
      where: { id },
      relations: ['posts'],
    });

    if (!user) {
      throw new NotFoundException(`User #${id} not found`);
    }

    return user;
  }

  async create(createUserDto: CreateUserDto): Promise<User> {
    const user = this.usersRepository.create(createUserDto);
    return this.usersRepository.save(user);
  }

  async update(id: string, updateUserDto: UpdateUserDto): Promise<User> {
    await this.usersRepository.update(id, updateUserDto);
    return this.findOne(id);
  }

  async remove(id: string): Promise<void> {
    const result = await this.usersRepository.delete(id);
    if (result.affected === 0) {
      throw new NotFoundException(`User #${id} not found`);
    }
  }
}
```

### Query Builder

```typescript
async findByFilters(filters: FilterDto): Promise<User[]> {
  const query = this.usersRepository
    .createQueryBuilder('user')
    .leftJoinAndSelect('user.posts', 'posts');

  if (filters.role) {
    query.andWhere('user.role = :role', { role: filters.role });
  }

  if (filters.search) {
    query.andWhere(
      '(user.email ILIKE :search OR user.name ILIKE :search)',
      { search: `%${filters.search}%` },
    );
  }

  if (filters.isActive !== undefined) {
    query.andWhere('user.isActive = :isActive', { isActive: filters.isActive });
  }

  return query
    .orderBy('user.createdAt', 'DESC')
    .skip((filters.page - 1) * filters.limit)
    .take(filters.limit)
    .getMany();
}
```

### Transactions

```typescript
@Injectable()
export class UsersService {
  constructor(private dataSource: DataSource) {}

  async createWithProfile(createUserDto: CreateUserDto): Promise<User> {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      const user = queryRunner.manager.create(User, createUserDto);
      await queryRunner.manager.save(user);

      const profile = queryRunner.manager.create(Profile, {
        userId: user.id,
        bio: '',
      });
      await queryRunner.manager.save(profile);

      await queryRunner.commitTransaction();
      return user;
    } catch (error) {
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      await queryRunner.release();
    }
  }
}
```

---

## Prisma

Prisma provides type-safe database access with excellent TypeScript integration.

### Installation

```bash
npm install @prisma/client
npm install -D prisma
npx prisma init
```

### Schema Definition

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  password  String
  name      String?
  role      Role     @default(USER)
  isActive  Boolean  @default(true)
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([email])
  @@map("users")
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  authorId  String
  tags      Tag[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("posts")
}

model Tag {
  id    Int    @id @default(autoincrement())
  name  String @unique
  posts Post[]

  @@map("tags")
}

enum Role {
  USER
  ADMIN
}
```

### Prisma Service

```typescript
// prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  constructor() {
    super({
      log:
        process.env.NODE_ENV === 'development'
          ? ['query', 'info', 'warn', 'error']
          : ['error'],
    });
  }

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }

  // Helper for transactions
  async cleanDatabase() {
    if (process.env.NODE_ENV === 'production') {
      throw new Error('cleanDatabase not allowed in production');
    }
    // Delete in correct order to respect foreign keys
    await this.post.deleteMany();
    await this.user.deleteMany();
  }
}
```

### Prisma Module

```typescript
// prisma/prisma.module.ts
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

### Service with Prisma

```typescript
// users/users.service.ts
@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  async findAll(): Promise<User[]> {
    return this.prisma.user.findMany({
      include: { posts: true },
      orderBy: { createdAt: 'desc' },
    });
  }

  async findOne(id: string): Promise<User> {
    const user = await this.prisma.user.findUnique({
      where: { id },
      include: { posts: true },
    });

    if (!user) {
      throw new NotFoundException(`User #${id} not found`);
    }

    return user;
  }

  async create(data: CreateUserDto): Promise<User> {
    return this.prisma.user.create({
      data,
      include: { posts: true },
    });
  }

  async update(id: string, data: UpdateUserDto): Promise<User> {
    try {
      return await this.prisma.user.update({
        where: { id },
        data,
        include: { posts: true },
      });
    } catch (error) {
      if (error.code === 'P2025') {
        throw new NotFoundException(`User #${id} not found`);
      }
      throw error;
    }
  }

  async remove(id: string): Promise<void> {
    try {
      await this.prisma.user.delete({ where: { id } });
    } catch (error) {
      if (error.code === 'P2025') {
        throw new NotFoundException(`User #${id} not found`);
      }
      throw error;
    }
  }
}
```

### Prisma Transactions

```typescript
async createWithProfile(data: CreateUserDto): Promise<User> {
  return this.prisma.$transaction(async (tx) => {
    const user = await tx.user.create({
      data: {
        email: data.email,
        password: data.password,
        name: data.name,
      },
    });

    await tx.profile.create({
      data: {
        userId: user.id,
        bio: '',
      },
    });

    return user;
  });
}

// Interactive transaction with timeout
async transferCredits(fromId: string, toId: string, amount: number) {
  return this.prisma.$transaction(
    async (tx) => {
      const sender = await tx.user.update({
        where: { id: fromId },
        data: { credits: { decrement: amount } },
      });

      if (sender.credits < 0) {
        throw new BadRequestException('Insufficient credits');
      }

      await tx.user.update({
        where: { id: toId },
        data: { credits: { increment: amount } },
      });

      return { success: true };
    },
    {
      maxWait: 5000,
      timeout: 10000,
      isolationLevel: 'Serializable',
    },
  );
}
```

### Prisma Filters and Pagination

```typescript
async findByFilters(filters: FilterDto) {
  const where: Prisma.UserWhereInput = {};

  if (filters.role) {
    where.role = filters.role;
  }

  if (filters.search) {
    where.OR = [
      { email: { contains: filters.search, mode: 'insensitive' } },
      { name: { contains: filters.search, mode: 'insensitive' } },
    ];
  }

  if (filters.isActive !== undefined) {
    where.isActive = filters.isActive;
  }

  const [users, total] = await Promise.all([
    this.prisma.user.findMany({
      where,
      skip: (filters.page - 1) * filters.limit,
      take: filters.limit,
      orderBy: { createdAt: 'desc' },
      include: { posts: { take: 5 } },
    }),
    this.prisma.user.count({ where }),
  ]);

  return {
    data: users,
    meta: {
      total,
      page: filters.page,
      limit: filters.limit,
      totalPages: Math.ceil(total / filters.limit),
    },
  };
}
```

---

## Drizzle ORM

Drizzle is a lightweight, type-safe ORM with SQL-like syntax.

### Installation

```bash
npm install drizzle-orm pg
npm install -D drizzle-kit @types/pg
```

### Schema Definition

```typescript
// src/database/schema.ts
import {
  pgTable,
  serial,
  text,
  varchar,
  timestamp,
  boolean,
  integer,
  uuid,
  pgEnum,
} from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';

export const roleEnum = pgEnum('role', ['user', 'admin']);

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  password: text('password').notNull(),
  name: varchar('name', { length: 100 }),
  role: roleEnum('role').default('user'),
  isActive: boolean('is_active').default(true),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
});

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: varchar('title', { length: 255 }).notNull(),
  content: text('content'),
  published: boolean('published').default(false),
  authorId: uuid('author_id')
    .notNull()
    .references(() => users.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
});

export const tags = pgTable('tags', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 50 }).notNull().unique(),
});

export const postsTags = pgTable('posts_tags', {
  postId: integer('post_id')
    .notNull()
    .references(() => posts.id, { onDelete: 'cascade' }),
  tagId: integer('tag_id')
    .notNull()
    .references(() => tags.id, { onDelete: 'cascade' }),
});

// Relations
export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

export const postsRelations = relations(posts, ({ one, many }) => ({
  author: one(users, {
    fields: [posts.authorId],
    references: [users.id],
  }),
  tags: many(postsTags),
}));
```

### Drizzle Configuration

```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/database/schema.ts',
  out: './drizzle/migrations',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
});
```

### Drizzle Module

```typescript
// src/database/drizzle.module.ts
import { Module, Global } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Pool } from 'pg';
import { drizzle, NodePgDatabase } from 'drizzle-orm/node-postgres';
import * as schema from './schema';

export const DRIZZLE = Symbol('drizzle-connection');

@Global()
@Module({
  providers: [
    {
      provide: DRIZZLE,
      inject: [ConfigService],
      useFactory: async (configService: ConfigService) => {
        const pool = new Pool({
          connectionString: configService.get<string>('DATABASE_URL'),
        });
        return drizzle(pool, { schema }) as NodePgDatabase<typeof schema>;
      },
    },
  ],
  exports: [DRIZZLE],
})
export class DrizzleModule {}
```

### Service with Drizzle

```typescript
// users/users.service.ts
import { Inject, Injectable, NotFoundException } from '@nestjs/common';
import { NodePgDatabase } from 'drizzle-orm/node-postgres';
import { eq, ilike, and, or, desc } from 'drizzle-orm';
import { DRIZZLE } from '../database/drizzle.module';
import * as schema from '../database/schema';

@Injectable()
export class UsersService {
  constructor(
    @Inject(DRIZZLE) private db: NodePgDatabase<typeof schema>,
  ) {}

  async findAll() {
    return this.db.query.users.findMany({
      with: { posts: true },
      orderBy: [desc(schema.users.createdAt)],
    });
  }

  async findOne(id: string) {
    const user = await this.db.query.users.findFirst({
      where: eq(schema.users.id, id),
      with: { posts: true },
    });

    if (!user) {
      throw new NotFoundException(`User #${id} not found`);
    }

    return user;
  }

  async create(createUserDto: CreateUserDto) {
    const [user] = await this.db
      .insert(schema.users)
      .values(createUserDto)
      .returning();

    return user;
  }

  async update(id: string, updateUserDto: UpdateUserDto) {
    const [user] = await this.db
      .update(schema.users)
      .set({ ...updateUserDto, updatedAt: new Date() })
      .where(eq(schema.users.id, id))
      .returning();

    if (!user) {
      throw new NotFoundException(`User #${id} not found`);
    }

    return user;
  }

  async remove(id: string) {
    const [deleted] = await this.db
      .delete(schema.users)
      .where(eq(schema.users.id, id))
      .returning({ id: schema.users.id });

    if (!deleted) {
      throw new NotFoundException(`User #${id} not found`);
    }
  }
}
```

### Drizzle Filters and Pagination

```typescript
async findByFilters(filters: FilterDto) {
  const conditions = [];

  if (filters.role) {
    conditions.push(eq(schema.users.role, filters.role));
  }

  if (filters.search) {
    conditions.push(
      or(
        ilike(schema.users.email, `%${filters.search}%`),
        ilike(schema.users.name, `%${filters.search}%`),
      ),
    );
  }

  if (filters.isActive !== undefined) {
    conditions.push(eq(schema.users.isActive, filters.isActive));
  }

  const whereClause = conditions.length > 0 ? and(...conditions) : undefined;

  const users = await this.db
    .select()
    .from(schema.users)
    .where(whereClause)
    .orderBy(desc(schema.users.createdAt))
    .limit(filters.limit)
    .offset((filters.page - 1) * filters.limit);

  const [{ count }] = await this.db
    .select({ count: sql<number>`count(*)` })
    .from(schema.users)
    .where(whereClause);

  return {
    data: users,
    meta: {
      total: Number(count),
      page: filters.page,
      limit: filters.limit,
    },
  };
}
```

### Drizzle Transactions

```typescript
async createWithProfile(data: CreateUserDto) {
  return this.db.transaction(async (tx) => {
    const [user] = await tx
      .insert(schema.users)
      .values({
        email: data.email,
        password: data.password,
        name: data.name,
      })
      .returning();

    await tx.insert(schema.profiles).values({
      userId: user.id,
      bio: '',
    });

    return user;
  });
}
```

---

## Common Patterns

### Repository Abstraction

Create a generic interface that works with any ORM:

```typescript
// common/interfaces/repository.interface.ts
export interface IRepository<T> {
  findAll(): Promise<T[]>;
  findOne(id: string | number): Promise<T | null>;
  create(data: Partial<T>): Promise<T>;
  update(id: string | number, data: Partial<T>): Promise<T>;
  delete(id: string | number): Promise<void>;
}

// Implementation can use TypeORM, Prisma, or Drizzle
@Injectable()
export class UsersRepository implements IRepository<User> {
  constructor(private prisma: PrismaService) {}

  async findAll() {
    return this.prisma.user.findMany();
  }

  // ... other methods
}
```

### Database Health Check

```typescript
// health/database.health.ts
import { Injectable } from '@nestjs/common';
import {
  HealthIndicator,
  HealthIndicatorResult,
  HealthCheckError,
} from '@nestjs/terminus';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class DatabaseHealthIndicator extends HealthIndicator {
  constructor(private prisma: PrismaService) {
    super();
  }

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    try {
      await this.prisma.$queryRaw`SELECT 1`;
      return this.getStatus(key, true);
    } catch (error) {
      throw new HealthCheckError(
        'Database check failed',
        this.getStatus(key, false, { message: error.message }),
      );
    }
  }
}

// health/health.controller.ts
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: DatabaseHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.isHealthy('database'),
    ]);
  }
}
```

### Soft Delete Pattern

```typescript
// TypeORM
@Entity()
export class User {
  @DeleteDateColumn()
  deletedAt?: Date;
}

// Prisma - manual implementation
model User {
  deletedAt DateTime?
}

// Service
async softDelete(id: string) {
  return this.prisma.user.update({
    where: { id },
    data: { deletedAt: new Date() },
  });
}

async findActive() {
  return this.prisma.user.findMany({
    where: { deletedAt: null },
  });
}

// Drizzle
async softDelete(id: string) {
  return this.db
    .update(schema.users)
    .set({ deletedAt: new Date() })
    .where(eq(schema.users.id, id));
}
```

### Migrations

```bash
# TypeORM
npx typeorm migration:generate -d data-source.ts src/migrations/CreateUsers
npx typeorm migration:run -d data-source.ts

# Prisma
npx prisma migrate dev --name create_users
npx prisma migrate deploy  # Production

# Drizzle
npx drizzle-kit generate
npx drizzle-kit migrate
```
