# Validation & Pipes

## Table of Contents

- [ValidationPipe Setup](#validationpipe-setup)
- [DTO Best Practices](#dto-best-practices)
- [Class-Validator Decorators](#class-validator-decorators)
- [Class-Transformer Usage](#class-transformer-usage)
- [Custom Validation Pipes](#custom-validation-pipes)
- [Mapped Types](#mapped-types)

---

## ValidationPipe Setup

The `ValidationPipe` uses `class-validator` and `class-transformer` packages to validate incoming data.

### Installation

```bash
npm install class-validator class-transformer
```

### Global Configuration (Recommended)

```typescript
// main.ts
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,              // Strip properties not in DTO
      forbidNonWhitelisted: true,   // Throw error for unknown properties
      transform: true,              // Auto-transform payloads to DTO instances
      transformOptions: {
        enableImplicitConversion: true, // Convert types based on TS metadata
      },
    }),
  );

  await app.listen(3000);
}
```

### Configuration Options Explained

| Option | Default | Description |
|--------|---------|-------------|
| `whitelist` | `false` | Remove non-decorated properties from validated objects |
| `forbidNonWhitelisted` | `false` | Throw error when non-decorated properties are present |
| `transform` | `false` | Automatically transform payloads to DTO class instances |
| `disableErrorMessages` | `false` | Don't return validation error details in response |
| `skipMissingProperties` | `false` | Skip validation for missing properties |
| `skipNullProperties` | `false` | Skip validation for null properties |
| `skipUndefinedProperties` | `false` | Skip validation for undefined properties |
| `enableDebugMessages` | `false` | Log additional debug info during validation |
| `exceptionFactory` | default | Custom function to create validation exception |
| `validationError.target` | `true` | Include target object in error |
| `validationError.value` | `true` | Include validated value in error |

### Custom Error Response

```typescript
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
    exceptionFactory: (errors) => {
      const messages = errors.map((error) => ({
        field: error.property,
        errors: Object.values(error.constraints || {}),
      }));

      return new BadRequestException({
        statusCode: 400,
        message: 'Validation failed',
        errors: messages,
      });
    },
  }),
);
```

### Per-Route Configuration

```typescript
@Post()
@UsePipes(new ValidationPipe({ transform: true, groups: ['create'] }))
create(@Body() createUserDto: CreateUserDto) {}
```

---

## DTO Best Practices

### Basic DTO Structure

```typescript
// dto/create-user.dto.ts
import {
  IsEmail,
  IsString,
  MinLength,
  MaxLength,
  IsOptional,
  Matches,
} from 'class-validator';

export class CreateUserDto {
  @IsEmail({}, { message: 'Invalid email format' })
  email: string;

  @IsString()
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  @MaxLength(32)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'Password must contain uppercase, lowercase, and number',
  })
  password: string;

  @IsOptional()
  @IsString()
  @MaxLength(100)
  name?: string;
}
```

### Nested Object Validation

```typescript
import { Type } from 'class-transformer';
import { ValidateNested, IsArray } from 'class-validator';

class AddressDto {
  @IsString()
  street: string;

  @IsString()
  city: string;

  @IsString()
  @Matches(/^\d{5}(-\d{4})?$/)
  zipCode: string;
}

export class CreateUserDto {
  @IsEmail()
  email: string;

  @ValidateNested()
  @Type(() => AddressDto)  // Required for nested object validation
  address: AddressDto;

  @IsArray()
  @ValidateNested({ each: true })  // Validate each item in array
  @Type(() => AddressDto)
  addresses: AddressDto[];
}
```

### Conditional Validation

```typescript
import { ValidateIf, IsNotEmpty, IsString } from 'class-validator';

export class PaymentDto {
  @IsString()
  paymentType: 'card' | 'bank';

  @ValidateIf((o) => o.paymentType === 'card')
  @IsNotEmpty()
  @IsCreditCard()
  cardNumber?: string;

  @ValidateIf((o) => o.paymentType === 'bank')
  @IsNotEmpty()
  @IsIBAN()
  iban?: string;
}
```

### Validation Groups

```typescript
export class CreateUserDto {
  @IsEmail({}, { groups: ['create', 'update'] })
  email: string;

  @IsString({ groups: ['create'] })
  @MinLength(8, { groups: ['create'] })
  password: string;

  @IsOptional({ groups: ['update'] })
  @IsString({ groups: ['update'] })
  name?: string;
}

// Usage
@Post()
@UsePipes(new ValidationPipe({ groups: ['create'] }))
create(@Body() dto: CreateUserDto) {}

@Patch(':id')
@UsePipes(new ValidationPipe({ groups: ['update'] }))
update(@Param('id') id: string, @Body() dto: CreateUserDto) {}
```

---

## Class-Validator Decorators

### String Validators

```typescript
// Basic string
@IsString()
@IsNotEmpty()
@MinLength(2)
@MaxLength(100)
name: string;

// Email
@IsEmail()
email: string;

// URL
@IsUrl()
website: string;

// UUID
@IsUUID('4')
id: string;

// Regex pattern
@Matches(/^[a-z0-9-]+$/)
slug: string;

// Enum
@IsEnum(UserRole)
role: UserRole;

// Alpha/Alphanumeric
@IsAlpha()
firstName: string;

@IsAlphanumeric()
username: string;
```

### Number Validators

```typescript
// Integer
@IsInt()
@Min(1)
@Max(100)
age: number;

// Decimal/Float
@IsNumber({ maxDecimalPlaces: 2 })
@IsPositive()
price: number;

// Negative
@IsNegative()
temperature: number;

// Divisible by
@IsDivisibleBy(5)
quantity: number;
```

### Boolean & Date Validators

```typescript
// Boolean
@IsBoolean()
isActive: boolean;

// Boolean string (accepts 'true', 'false', '1', '0')
@IsBooleanString()
enabled: string;

// Date
@IsDate()
@Type(() => Date)
createdAt: Date;

// Date string (ISO format)
@IsDateString()
birthDate: string;
```

### Array Validators

```typescript
// Array of specific type
@IsArray()
@IsString({ each: true })  // Validate each element
@ArrayMinSize(1)
@ArrayMaxSize(10)
@ArrayUnique()
tags: string[];

// Array of numbers
@IsArray()
@IsInt({ each: true })
@ArrayMinSize(1)
scores: number[];

// Array contains specific value
@ArrayContains(['admin'])
roles: string[];
```

### Object Validators

```typescript
// Nested object
@ValidateNested()
@Type(() => AddressDto)
address: AddressDto;

// Object is not empty
@IsNotEmptyObject()
metadata: Record<string, any>;

// Instance of class
@IsInstance(Date)
date: Date;
```

### Custom Decorators

```typescript
import {
  registerDecorator,
  ValidationOptions,
  ValidationArguments,
} from 'class-validator';

// Custom decorator to check if value matches another property
export function Match(property: string, options?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      name: 'match',
      target: object.constructor,
      propertyName: propertyName,
      constraints: [property],
      options: options,
      validator: {
        validate(value: any, args: ValidationArguments) {
          const [relatedPropertyName] = args.constraints;
          const relatedValue = (args.object as any)[relatedPropertyName];
          return value === relatedValue;
        },
        defaultMessage(args: ValidationArguments) {
          const [relatedPropertyName] = args.constraints;
          return `${propertyName} must match ${relatedPropertyName}`;
        },
      },
    });
  };
}

// Usage
export class RegisterDto {
  @IsString()
  @MinLength(8)
  password: string;

  @IsString()
  @Match('password', { message: 'Passwords do not match' })
  confirmPassword: string;
}
```

### Async Validation (Database Lookup)

```typescript
import {
  ValidatorConstraint,
  ValidatorConstraintInterface,
} from 'class-validator';

@ValidatorConstraint({ name: 'isEmailUnique', async: true })
@Injectable()
export class IsEmailUniqueConstraint implements ValidatorConstraintInterface {
  constructor(private usersService: UsersService) {}

  async validate(email: string): Promise<boolean> {
    const user = await this.usersService.findByEmail(email);
    return !user; // Return true if email doesn't exist
  }

  defaultMessage(): string {
    return 'Email already exists';
  }
}

export function IsEmailUnique(options?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName: propertyName,
      options: options,
      constraints: [],
      validator: IsEmailUniqueConstraint,
    });
  };
}

// Register in module
@Module({
  providers: [IsEmailUniqueConstraint],
})
export class UsersModule {}

// Usage
export class CreateUserDto {
  @IsEmail()
  @IsEmailUnique()
  email: string;
}
```

---

## Class-Transformer Usage

### Basic Transformation

```typescript
import { Transform, Type, Exclude, Expose } from 'class-transformer';

export class UserResponseDto {
  @Expose()
  id: number;

  @Expose()
  email: string;

  @Exclude()
  password: string;  // Never sent to client

  @Transform(({ value }) => value.toLowerCase())
  @Expose()
  username: string;

  @Type(() => Date)
  @Expose()
  createdAt: Date;
}
```

### Transform Decorators

```typescript
// Trim whitespace
@Transform(({ value }) => value?.trim())
@IsString()
name: string;

// Convert to lowercase
@Transform(({ value }) => value?.toLowerCase())
@IsEmail()
email: string;

// Parse JSON string
@Transform(({ value }) => {
  try {
    return typeof value === 'string' ? JSON.parse(value) : value;
  } catch {
    return value;
  }
})
metadata: Record<string, any>;

// Default value
@Transform(({ value }) => value ?? 'default')
status: string;

// Convert array from comma-separated string
@Transform(({ value }) =>
  typeof value === 'string' ? value.split(',').map((v) => v.trim()) : value,
)
@IsArray()
@IsString({ each: true })
tags: string[];
```

### Exclude/Expose Strategies

```typescript
import { Exclude, Expose, plainToInstance } from 'class-transformer';

// Exclude by default, only expose marked properties
@Exclude()
export class UserResponseDto {
  @Expose()
  id: number;

  @Expose()
  email: string;

  // password is excluded automatically

  @Expose({ groups: ['admin'] })
  role: string;  // Only exposed for admin group
}

// Transform entity to DTO
const user = await this.usersRepository.findOne(id);
return plainToInstance(UserResponseDto, user, {
  excludeExtraneousValues: true,
  groups: ['admin'], // Include admin-only fields
});
```

### Serialization Interceptor

```typescript
import { ClassSerializerInterceptor } from '@nestjs/common';

// Global serialization
app.useGlobalInterceptors(new ClassSerializerInterceptor(app.get(Reflector)));

// Or per-controller
@Controller('users')
@UseInterceptors(ClassSerializerInterceptor)
export class UsersController {
  @Get(':id')
  @SerializeOptions({ groups: ['admin'] })
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }
}
```

---

## Custom Validation Pipes

### Schema-Based Pipe (Zod)

```typescript
import { z, ZodSchema, ZodError } from 'zod';
import { PipeTransform, BadRequestException } from '@nestjs/common';

export class ZodValidationPipe<T> implements PipeTransform<unknown, T> {
  constructor(private schema: ZodSchema<T>) {}

  transform(value: unknown): T {
    const result = this.schema.safeParse(value);

    if (!result.success) {
      const errors = result.error.errors.map((err) => ({
        path: err.path.join('.'),
        message: err.message,
      }));

      throw new BadRequestException({
        message: 'Validation failed',
        errors,
      });
    }

    return result.data;
  }
}

// Define schema
const createUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string().optional(),
});

type CreateUserDto = z.infer<typeof createUserSchema>;

// Usage
@Post()
create(
  @Body(new ZodValidationPipe(createUserSchema))
  createUserDto: CreateUserDto,
) {
  return this.usersService.create(createUserDto);
}
```

### File Validation Pipe

```typescript
import { PipeTransform, BadRequestException } from '@nestjs/common';

export interface FileValidationOptions {
  maxSize?: number;
  mimeTypes?: string[];
}

export class FileValidationPipe implements PipeTransform {
  constructor(private options: FileValidationOptions = {}) {}

  transform(file: Express.Multer.File): Express.Multer.File {
    if (!file) {
      throw new BadRequestException('File is required');
    }

    const { maxSize = 5 * 1024 * 1024, mimeTypes } = this.options;

    if (file.size > maxSize) {
      throw new BadRequestException(
        `File size exceeds ${maxSize / 1024 / 1024}MB limit`,
      );
    }

    if (mimeTypes && !mimeTypes.includes(file.mimetype)) {
      throw new BadRequestException(
        `File type ${file.mimetype} is not allowed. Allowed: ${mimeTypes.join(', ')}`,
      );
    }

    return file;
  }
}

// Usage
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
uploadFile(
  @UploadedFile(
    new FileValidationPipe({
      maxSize: 2 * 1024 * 1024,
      mimeTypes: ['image/jpeg', 'image/png'],
    }),
  )
  file: Express.Multer.File,
) {
  return { filename: file.filename };
}
```

### Query Parsing Pipe

```typescript
export interface PaginationQuery {
  page: number;
  limit: number;
  sort?: string;
  order?: 'asc' | 'desc';
}

export class PaginationPipe implements PipeTransform<any, PaginationQuery> {
  transform(value: any): PaginationQuery {
    return {
      page: Math.max(1, parseInt(value.page, 10) || 1),
      limit: Math.min(100, Math.max(1, parseInt(value.limit, 10) || 10)),
      sort: value.sort || 'createdAt',
      order: value.order === 'asc' ? 'asc' : 'desc',
    };
  }
}

// Usage
@Get()
findAll(@Query(PaginationPipe) pagination: PaginationQuery) {
  return this.usersService.findAll(pagination);
}
```

---

## Mapped Types

NestJS provides utility types for creating DTOs based on existing ones.

### PartialType

Makes all properties optional:

```typescript
import { PartialType } from '@nestjs/mapped-types';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsString()
  name: string;
}

// All properties become optional
export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

### PickType

Pick specific properties:

```typescript
import { PickType } from '@nestjs/mapped-types';

// Only email and password
export class LoginDto extends PickType(CreateUserDto, [
  'email',
  'password',
] as const) {}
```

### OmitType

Exclude specific properties:

```typescript
import { OmitType } from '@nestjs/mapped-types';

// Everything except password
export class UserProfileDto extends OmitType(CreateUserDto, [
  'password',
] as const) {}
```

### IntersectionType

Combine multiple DTOs:

```typescript
import { IntersectionType } from '@nestjs/mapped-types';

class PaginationDto {
  @IsInt()
  @Min(1)
  page: number;

  @IsInt()
  @Min(1)
  @Max(100)
  limit: number;
}

class FilterDto {
  @IsOptional()
  @IsString()
  search?: string;

  @IsOptional()
  @IsEnum(UserStatus)
  status?: UserStatus;
}

// Combines pagination and filter properties
export class QueryUsersDto extends IntersectionType(
  PaginationDto,
  FilterDto,
) {}
```

### Combining Mapped Types

```typescript
// Partial + Pick
export class UpdateEmailDto extends PartialType(
  PickType(CreateUserDto, ['email'] as const),
) {}

// Omit + Intersection
export class CreateAdminDto extends IntersectionType(
  OmitType(CreateUserDto, ['role'] as const),
  class {
    @IsEnum(AdminRole)
    role: AdminRole;
  },
) {}
```

### With Swagger Support

When using `@nestjs/swagger`, import from `@nestjs/swagger` instead:

```typescript
import { PartialType, PickType, OmitType, IntersectionType } from '@nestjs/swagger';

// Swagger decorators are preserved
export class UpdateUserDto extends PartialType(CreateUserDto) {}
```
