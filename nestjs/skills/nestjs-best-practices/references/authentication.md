# Authentication & Authorization

## Table of Contents

- [Passport Integration](#passport-integration)
- [Local Strategy](#local-strategy)
- [JWT Strategy](#jwt-strategy)
- [Refresh Tokens](#refresh-tokens)
- [OAuth2 Strategies](#oauth2-strategies)
- [Guards and Decorators](#guards-and-decorators)
- [Role-Based Access Control](#role-based-access-control)

---

## Passport Integration

Passport is the most popular Node.js authentication library, integrated via `@nestjs/passport`.

### Installation

```bash
npm install @nestjs/passport passport passport-local passport-jwt
npm install -D @types/passport-local @types/passport-jwt
npm install @nestjs/jwt
```

### Auth Module Setup

```typescript
// auth/auth.module.ts
import { Module } from '@nestjs/common';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { LocalStrategy } from './strategies/local.strategy';
import { JwtStrategy } from './strategies/jwt.strategy';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [
    UsersModule,
    PassportModule.register({ defaultStrategy: 'jwt' }),
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (config: ConfigService) => ({
        secret: config.get('JWT_SECRET'),
        signOptions: { expiresIn: '15m' },
      }),
      inject: [ConfigService],
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule {}
```

---

## Local Strategy

Username/password authentication for login endpoints.

### Strategy Implementation

```typescript
// auth/strategies/local.strategy.ts
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { AuthService } from '../auth.service';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super({
      usernameField: 'email',  // Use email instead of username
      passwordField: 'password',
    });
  }

  async validate(email: string, password: string): Promise<any> {
    const user = await this.authService.validateUser(email, password);

    if (!user) {
      throw new UnauthorizedException('Invalid credentials');
    }

    return user;  // Attached to request.user
  }
}
```

### Auth Service

```typescript
// auth/auth.service.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UsersService } from '../users/users.service';
import * as bcrypt from 'bcrypt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

  async validateUser(email: string, password: string): Promise<any> {
    const user = await this.usersService.findByEmail(email);

    if (user && (await bcrypt.compare(password, user.password))) {
      const { password, ...result } = user;
      return result;
    }

    return null;
  }

  async login(user: any) {
    const payload = { email: user.email, sub: user.id, role: user.role };

    return {
      accessToken: this.jwtService.sign(payload),
      user: {
        id: user.id,
        email: user.email,
        role: user.role,
      },
    };
  }

  async register(createUserDto: CreateUserDto) {
    const hashedPassword = await bcrypt.hash(createUserDto.password, 10);

    const user = await this.usersService.create({
      ...createUserDto,
      password: hashedPassword,
    });

    return this.login(user);
  }
}
```

### Local Auth Guard

```typescript
// auth/guards/local-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}
```

### Login Endpoint

```typescript
// auth/auth.controller.ts
@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  @Post('login')
  @UseGuards(LocalAuthGuard)
  @HttpCode(HttpStatus.OK)
  async login(@Request() req) {
    return this.authService.login(req.user);
  }

  @Post('register')
  async register(@Body() createUserDto: CreateUserDto) {
    return this.authService.register(createUserDto);
  }
}
```

---

## JWT Strategy

Stateless authentication using JSON Web Tokens.

### Strategy Implementation

```typescript
// auth/strategies/jwt.strategy.ts
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { UsersService } from '../../users/users.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    private configService: ConfigService,
    private usersService: UsersService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get('JWT_SECRET'),
    });
  }

  async validate(payload: any) {
    // Optionally verify user still exists/is active
    const user = await this.usersService.findById(payload.sub);

    if (!user || !user.isActive) {
      throw new UnauthorizedException('User not found or inactive');
    }

    // Return value is attached to request.user
    return {
      id: payload.sub,
      email: payload.email,
      role: payload.role,
    };
  }
}
```

### JWT Auth Guard

```typescript
// auth/guards/jwt-auth.guard.ts
import {
  Injectable,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    // Check for @Public() decorator
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) {
      return true;
    }

    return super.canActivate(context);
  }

  handleRequest(err: any, user: any, info: any) {
    if (err || !user) {
      throw err || new UnauthorizedException('Invalid or expired token');
    }
    return user;
  }
}
```

### Public Decorator

```typescript
// auth/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

### Global JWT Protection

```typescript
// app.module.ts
@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,
    },
  ],
})
export class AppModule {}

// Usage - all routes protected by default
@Controller('users')
export class UsersController {
  @Get()
  findAll() {} // Protected

  @Get('public')
  @Public()  // Not protected
  findPublic() {}
}
```

---

## Refresh Tokens

Implement token refresh for better security.

### Service Methods

```typescript
// auth/auth.service.ts
@Injectable()
export class AuthService {
  constructor(
    private jwtService: JwtService,
    private configService: ConfigService,
    private usersService: UsersService,
  ) {}

  async login(user: any) {
    const tokens = await this.generateTokens(user);
    await this.updateRefreshToken(user.id, tokens.refreshToken);

    return {
      ...tokens,
      user: { id: user.id, email: user.email },
    };
  }

  async generateTokens(user: any) {
    const payload = { email: user.email, sub: user.id, role: user.role };

    const [accessToken, refreshToken] = await Promise.all([
      this.jwtService.signAsync(payload, {
        secret: this.configService.get('JWT_ACCESS_SECRET'),
        expiresIn: '15m',
      }),
      this.jwtService.signAsync(payload, {
        secret: this.configService.get('JWT_REFRESH_SECRET'),
        expiresIn: '7d',
      }),
    ]);

    return { accessToken, refreshToken };
  }

  async updateRefreshToken(userId: number, refreshToken: string) {
    const hashedRefreshToken = await bcrypt.hash(refreshToken, 10);
    await this.usersService.update(userId, {
      refreshToken: hashedRefreshToken,
    });
  }

  async refreshTokens(userId: number, refreshToken: string) {
    const user = await this.usersService.findById(userId);

    if (!user || !user.refreshToken) {
      throw new ForbiddenException('Access denied');
    }

    const refreshTokenMatches = await bcrypt.compare(
      refreshToken,
      user.refreshToken,
    );

    if (!refreshTokenMatches) {
      throw new ForbiddenException('Access denied');
    }

    const tokens = await this.generateTokens(user);
    await this.updateRefreshToken(user.id, tokens.refreshToken);

    return tokens;
  }

  async logout(userId: number) {
    await this.usersService.update(userId, { refreshToken: null });
  }
}
```

### Refresh Token Strategy

```typescript
// auth/strategies/jwt-refresh.strategy.ts
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { Request } from 'express';
import { Injectable } from '@nestjs/common';

@Injectable()
export class JwtRefreshStrategy extends PassportStrategy(
  Strategy,
  'jwt-refresh',
) {
  constructor(private configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: configService.get('JWT_REFRESH_SECRET'),
      passReqToCallback: true,
    });
  }

  validate(req: Request, payload: any) {
    const refreshToken = req.get('Authorization').replace('Bearer', '').trim();

    return {
      ...payload,
      refreshToken,
    };
  }
}

// Guard
@Injectable()
export class JwtRefreshGuard extends AuthGuard('jwt-refresh') {}
```

### Refresh Endpoint

```typescript
@Controller('auth')
export class AuthController {
  @Post('refresh')
  @UseGuards(JwtRefreshGuard)
  refreshTokens(@Request() req) {
    const userId = req.user.sub;
    const refreshToken = req.user.refreshToken;
    return this.authService.refreshTokens(userId, refreshToken);
  }

  @Post('logout')
  @UseGuards(JwtAuthGuard)
  logout(@Request() req) {
    return this.authService.logout(req.user.id);
  }
}
```

---

## OAuth2 Strategies

### Google OAuth2

```bash
npm install passport-google-oauth20
npm install -D @types/passport-google-oauth20
```

```typescript
// auth/strategies/google.strategy.ts
import { PassportStrategy } from '@nestjs/passport';
import { Strategy, VerifyCallback } from 'passport-google-oauth20';
import { Injectable } from '@nestjs/common';

@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor(private configService: ConfigService) {
    super({
      clientID: configService.get('GOOGLE_CLIENT_ID'),
      clientSecret: configService.get('GOOGLE_CLIENT_SECRET'),
      callbackURL: configService.get('GOOGLE_CALLBACK_URL'),
      scope: ['email', 'profile'],
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: any,
    done: VerifyCallback,
  ): Promise<any> {
    const { name, emails, photos } = profile;

    const user = {
      email: emails[0].value,
      firstName: name.givenName,
      lastName: name.familyName,
      picture: photos[0].value,
      accessToken,
    };

    done(null, user);
  }
}

// Guard
@Injectable()
export class GoogleOAuthGuard extends AuthGuard('google') {
  constructor() {
    super({
      accessType: 'offline',
      prompt: 'consent',
    });
  }
}
```

### OAuth Controller

```typescript
@Controller('auth')
export class AuthController {
  @Get('google')
  @UseGuards(GoogleOAuthGuard)
  async googleAuth() {}

  @Get('google/callback')
  @UseGuards(GoogleOAuthGuard)
  async googleAuthCallback(@Request() req, @Res() res: Response) {
    const tokens = await this.authService.loginWithOAuth(req.user);

    // Redirect to frontend with tokens
    res.redirect(
      `${this.configService.get('FRONTEND_URL')}/auth/callback?token=${tokens.accessToken}`,
    );
  }
}
```

### GitHub OAuth2

```typescript
// auth/strategies/github.strategy.ts
import { Strategy } from 'passport-github2';

@Injectable()
export class GithubStrategy extends PassportStrategy(Strategy, 'github') {
  constructor(private configService: ConfigService) {
    super({
      clientID: configService.get('GITHUB_CLIENT_ID'),
      clientSecret: configService.get('GITHUB_CLIENT_SECRET'),
      callbackURL: configService.get('GITHUB_CALLBACK_URL'),
      scope: ['user:email'],
    });
  }

  async validate(accessToken: string, refreshToken: string, profile: any) {
    const { username, emails, photos } = profile;

    return {
      email: emails[0].value,
      username,
      avatar: photos[0]?.value,
      accessToken,
    };
  }
}
```

---

## Guards and Decorators

### Current User Decorator

```typescript
// auth/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
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

### Auth Decorator (Combined)

```typescript
// auth/decorators/auth.decorator.ts
import { applyDecorators, UseGuards } from '@nestjs/common';
import { ApiBearerAuth, ApiUnauthorizedResponse } from '@nestjs/swagger';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    UseGuards(JwtAuthGuard, RolesGuard),
    Roles(...roles),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}

// Usage
@Get('admin')
@Auth(Role.Admin)
adminOnly(@CurrentUser() user: User) {
  return { message: 'Admin area', user };
}
```

### Optional Auth Guard

```typescript
// auth/guards/optional-auth.guard.ts
@Injectable()
export class OptionalAuthGuard extends AuthGuard('jwt') {
  handleRequest(err: any, user: any) {
    // Don't throw on missing/invalid token
    return user || null;
  }
}

// Usage - user may or may not be authenticated
@Get('posts')
@UseGuards(OptionalAuthGuard)
getPosts(@CurrentUser() user: User | null) {
  if (user) {
    return this.postsService.getPersonalized(user.id);
  }
  return this.postsService.getPublic();
}
```

---

## Role-Based Access Control

### Define Roles

```typescript
// auth/enums/role.enum.ts
export enum Role {
  User = 'user',
  Admin = 'admin',
  SuperAdmin = 'superadmin',
}
```

### Roles Decorator

```typescript
// auth/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { Role } from '../enums/role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

### Roles Guard

```typescript
// auth/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';
import { Role } from '../enums/role.enum';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredRoles) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();

    return requiredRoles.some((role) => user.role === role);
  }
}
```

### Permission-Based Access Control

```typescript
// auth/enums/permission.enum.ts
export enum Permission {
  CreateUser = 'create:user',
  ReadUser = 'read:user',
  UpdateUser = 'update:user',
  DeleteUser = 'delete:user',
  ManageRoles = 'manage:roles',
}

// Role-Permission mapping
export const RolePermissions: Record<Role, Permission[]> = {
  [Role.User]: [Permission.ReadUser],
  [Role.Admin]: [
    Permission.CreateUser,
    Permission.ReadUser,
    Permission.UpdateUser,
  ],
  [Role.SuperAdmin]: Object.values(Permission),
};
```

### Permissions Decorator and Guard

```typescript
// auth/decorators/permissions.decorator.ts
export const PERMISSIONS_KEY = 'permissions';
export const Permissions = (...permissions: Permission[]) =>
  SetMetadata(PERMISSIONS_KEY, permissions);

// auth/guards/permissions.guard.ts
@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredPermissions = this.reflector.getAllAndOverride<Permission[]>(
      PERMISSIONS_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredPermissions) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();
    const userPermissions = RolePermissions[user.role] || [];

    return requiredPermissions.every((permission) =>
      userPermissions.includes(permission),
    );
  }
}

// Usage
@Delete(':id')
@UseGuards(JwtAuthGuard, PermissionsGuard)
@Permissions(Permission.DeleteUser)
deleteUser(@Param('id') id: string) {
  return this.usersService.delete(id);
}
```

### Resource Ownership Guard

```typescript
// auth/guards/ownership.guard.ts
@Injectable()
export class OwnershipGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const resourceId = request.params.id;

    // Admins bypass ownership check
    if (user.role === Role.Admin || user.role === Role.SuperAdmin) {
      return true;
    }

    // Get resource and check ownership
    // This is simplified - inject service via module
    const resource = request.resource; // Set by earlier middleware/interceptor

    return resource?.userId === user.id;
  }
}

// Usage with interceptor to load resource
@Get(':id')
@UseGuards(JwtAuthGuard, OwnershipGuard)
@UseInterceptors(LoadResourceInterceptor)
getPost(@Param('id') id: string) {
  return this.postsService.findOne(id);
}
```
