অবশ্যই। এবার **Authorization**-কে একদম beginner-friendly ভাবে, কিন্তু **production-grade NestJS backend** মাথায় রেখে বুঝি।

Authentication আর Authorization-এর পার্থক্য আগে এক লাইনে:

```text
Authentication = আপনি কে?
Authorization  = আপনি কী করতে পারবেন?
```

উদাহরণ:

```text
Login
 ↓
"আমি Arman"
 ↓
Authentication ✅

"আমি কি product delete করতে পারব?"
 ↓
Authorization ✅ / ❌
```

---

# Authorization

ধরুন আপনার application-এ তিন ধরনের user আছে:

```text
User
 ├── USER
 ├── FARMER
 └── ADMIN
```

তাদের permission আলাদা:

```text
USER
 └── product:read

FARMER
 ├── product:read
 ├── product:create
 └── product:update

ADMIN
 ├── product:read
 ├── product:create
 ├── product:update
 ├── product:delete
 └── user:manage
```

এখন এই access control করার জন্য আমরা শিখব:

```text
RBAC
Roles
Permissions
Guards
Custom Decorators
Policy-based Authorization
```

---

# 1. RBAC

RBAC-এর পূর্ণরূপ:

> **Role-Based Access Control**

সহজ ভাষায়:

> User-এর role দেখে তাকে permission দেওয়া।

ধরুন:

```text
USER
```

সে product দেখতে পারবে।

```text
FARMER
```

সে নিজের product create/update করতে পারবে।

```text
ADMIN
```

সবকিছু manage করতে পারবে।

Flow:

```text
Request
   ↓
User
   ↓
Role
   ↓
Authorization Check
   ↓
Allow / Deny
```

উদাহরণ:

```text
GET /products
```

```text
USER    → ✅
FARMER  → ✅
ADMIN   → ✅
```

কিন্তু:

```text
DELETE /products/123
```

```text
USER    → ❌
FARMER  → ❌
ADMIN   → ✅
```

---

# 2. Roles

Role হলো user-এর **পরিচয় অনুযায়ী access category**।

NestJS-এ প্রথমে enum বানানো যায়:

```ts
export enum Role {
  USER = 'USER',
  FARMER = 'FARMER',
  ADMIN = 'ADMIN',
}
```

User:

```ts
export interface User {
  id: string;
  email: string;
  role: Role;
}
```

Example:

```json
{
  "id": "user_123",
  "email": "arman@example.com",
  "role": "FARMER"
}
```

এখন backend জানে:

```text
এই user = FARMER
```

---

## Role দিয়ে access control

ধরুন শুধু ADMIN user delete করতে পারবে।

```ts
if (user.role !== Role.ADMIN) {
  throw new ForbiddenException();
}
```

কাজ করবে, কিন্তু অনেক জায়গায় এভাবে লিখতে থাকলে code messy হয়ে যাবে।

তাই NestJS-এর **Guards + Decorators** ব্যবহার করা হয়।

---

# 3. Permissions

Role হচ্ছে:

```text
USER
FARMER
ADMIN
```

কিন্তু permission হচ্ছে আরও specific:

```text
product:create
product:read
product:update
product:delete
user:manage
```

এখানে পার্থক্যটা খুব গুরুত্বপূর্ণ।

### Role

```text
FARMER
```

### Permission

```text
product:create
```

একজন farmer-এর:

```text
FARMER
   ↓
product:create
product:update
product:read
```

একজন ADMIN-এর:

```text
ADMIN
   ↓
product:create
product:read
product:update
product:delete
user:manage
```

---

# Role বনাম Permission

এটা এভাবে মনে রাখুন:

```text
Role = আপনি কে?

Permission = আপনি কী করতে পারবেন?
```

উদাহরণ:

```text
FARMER
  ↓
product:create
product:update
```

অর্থাৎ:

> Farmer হওয়া role, product create করা permission।

---

# Permission Enum

```ts
export enum Permission {
  PRODUCT_READ = 'product:read',
  PRODUCT_CREATE = 'product:create',
  PRODUCT_UPDATE = 'product:update',
  PRODUCT_DELETE = 'product:delete',
  USER_MANAGE = 'user:manage',
}
```

তারপর role → permissions mapping:

```ts
export const ROLE_PERMISSIONS: Record<Role, Permission[]> = {
  [Role.USER]: [
    Permission.PRODUCT_READ,
  ],

  [Role.FARMER]: [
    Permission.PRODUCT_READ,
    Permission.PRODUCT_CREATE,
    Permission.PRODUCT_UPDATE,
  ],

  [Role.ADMIN]: [
    Permission.PRODUCT_READ,
    Permission.PRODUCT_CREATE,
    Permission.PRODUCT_UPDATE,
    Permission.PRODUCT_DELETE,
    Permission.USER_MANAGE,
  ],
};
```

এখন:

```ts
ROLE_PERMISSIONS[Role.FARMER]
```

result:

```ts
[
  'product:read',
  'product:create',
  'product:update',
]
```

---

# 4. Guards

NestJS-এর **Guard** হলো এমন একটা security checkpoint যেটা controller-এর আগে চলে।

Flow:

```text
Request
   ↓
Middleware
   ↓
Guard
   ↓
Controller
   ↓
Service
```

Guard বলতে পারেন:

> "ভিতরে ঢোকার আগে আমি check করব আপনি allowed কিনা।"

---

## Simple Guard

```ts
@Injectable()
export class AdminGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();

    const user = request.user;

    return user?.role === Role.ADMIN;
  }
}
```

Controller:

```ts
@UseGuards(JwtAuthGuard, AdminGuard)
@Delete(':id')
deleteProduct(@Param('id') id: string) {
  return this.productsService.delete(id);
}
```

Flow:

```text
Request
   ↓
JwtAuthGuard
   ↓
User authenticated?
   ↓
AdminGuard
   ↓
ADMIN?
   ↓
Controller
```

---

## `401` বনাম `403`

Authorization বুঝতে এটা জানা জরুরি।

### 401 Unauthorized

User authenticated না।

```text
"আমি কে?"
```

server জানে না।

```text
No token
   ↓
401
```

### 403 Forbidden

Server জানে আপনি কে, কিন্তু আপনার permission নেই।

```text
User = FARMER

DELETE /products
       ↓
Permission নেই
       ↓
403 Forbidden
```

অর্থাৎ:

```text
401 = আপনি authenticated নন

403 = authenticated, কিন্তু allowed নন
```

---

# 5. Custom Decorators

এখন ধরুন আমরা চাই:

```ts
@Roles(Role.ADMIN)
```

এভাবে controller লিখতে।

NestJS-এ custom decorator বানানো যায়।

```ts
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';

export const Roles = (...roles: Role[]) =>
  SetMetadata(ROLES_KEY, roles);
```

এখন controller:

```ts
@Roles(Role.ADMIN)
@UseGuards(JwtAuthGuard, RolesGuard)
@Delete(':id')
deleteProduct(@Param('id') id: string) {
  return this.productsService.delete(id);
}
```

এখানে:

```ts
@Roles(Role.ADMIN)
```

মানে:

> এই endpoint শুধু ADMIN-এর জন্য।

---

# RolesGuard

এখন Guard metadata থেকে roles বের করবে।

```ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
  ) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles =
      this.reflector.getAllAndOverride<Role[]>(
        ROLES_KEY,
        [
          context.getHandler(),
          context.getClass(),
        ],
      );

    if (!requiredRoles) {
      return true;
    }

    const request =
      context.switchToHttp().getRequest();

    const user = request.user;

    return requiredRoles.includes(user.role);
  }
}
```

Flow:

```text
@Roles(ADMIN)
      ↓
RolesGuard
      ↓
Read metadata
      ↓
Get request.user
      ↓
user.role === ADMIN?
      ↓
YES → Controller
NO  → 403
```

---

# 6. Permission Guard

Role-based system-এর limitation হলো role অনেক সময় যথেষ্ট হয় না।

ধরুন:

```text
FARMER
```

দুইজন farmer:

```text
Farmer A
Farmer B
```

দুজনের role একই।

কিন্তু business rule:

```text
Farmer A → নিজের product update করতে পারবে
Farmer B → Farmer A-এর product update করতে পারবে না
```

শুধু role দিয়ে এটা বোঝানো কঠিন।

Permission system এখানে বেশি flexible।

---

## Permission decorator

```ts
export const PERMISSIONS_KEY = 'permissions';

export const Permissions = (
  ...permissions: Permission[]
) =>
  SetMetadata(
    PERMISSIONS_KEY,
    permissions,
  );
```

Controller:

```ts
@Permissions(Permission.PRODUCT_DELETE)
@UseGuards(JwtAuthGuard, PermissionsGuard)
@Delete(':id')
deleteProduct(@Param('id') id: string) {
  return this.productsService.delete(id);
}
```

---

# PermissionsGuard

```ts
@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
  ) {}

  canActivate(
    context: ExecutionContext,
  ): boolean {
    const requiredPermissions =
      this.reflector.getAllAndOverride<
        Permission[]
      >(PERMISSIONS_KEY, [
        context.getHandler(),
        context.getClass(),
      ]);

    if (!requiredPermissions?.length) {
      return true;
    }

    const request =
      context.switchToHttp().getRequest();

    const user = request.user;

    const userPermissions =
      ROLE_PERMISSIONS[user.role];

    return requiredPermissions.every(
      permission =>
        userPermissions.includes(permission),
    );
  }
}
```

এখন:

```text
ADMIN
 ↓
product:delete
 ↓
DELETE /products/:id
 ↓
✅
```

কিন্তু:

```text
FARMER
 ↓
product:delete
 ↓
❌
403 Forbidden
```

---

# 7. Policy-Based Authorization

এখন আসি একটু advanced এবং production-level authorization-এ।

RBAC বলছে:

```text
FARMER → product:update
```

কিন্তু প্রশ্ন:

> Farmer কি যেকোনো product update করতে পারবে?

সম্ভবত না।

ধরুন:

```text
Product #1
ownerId = farmer_123
```

User:

```text
farmer_123
```

তাহলে:

```text
farmer_123
   ↓
Product #1
   ↓
ownerId = farmer_123
   ↓
✅ Can update
```

কিন্তু:

```text
farmer_456
   ↓
Product #1
   ↓
ownerId = farmer_123
   ↓
❌ Cannot update
```

এটাই **Policy-based authorization**-এর জায়গা।

---

## Policy কী?

Policy হলো business rule।

উদাহরণ:

```text
User can update product IF:

user.role === FARMER
AND
product.ownerId === user.id
```

Code:

```ts
canUpdateProduct(
  user: User,
  product: Product,
): boolean {
  return (
    user.role === Role.ADMIN ||
    (
      user.role === Role.FARMER &&
      product.ownerId === user.id
    )
  );
}
```

এখানে:

```text
ADMIN
  ↓
যেকোনো product update

FARMER
  ↓
শুধু নিজের product update
```

---

# Policy Handler

একটা policy class করতে পারেন:

```ts
@Injectable()
export class ProductPolicy {
  canUpdate(
    user: User,
    product: Product,
  ): boolean {
    if (user.role === Role.ADMIN) {
      return true;
    }

    if (user.role === Role.FARMER) {
      return product.ownerId === user.id;
    }

    return false;
  }
}
```

Service/controller থেকে:

```ts
const product =
  await this.productsService.findById(id);

const allowed =
  this.productPolicy.canUpdate(
    req.user,
    product,
  );

if (!allowed) {
  throw new ForbiddenException(
    'You cannot update this product',
  );
}
```

---

# সবগুলো একসাথে

Production application-এ authorization অনেকটা এমন:

```text
                     Authorization
                           │
          ┌────────────────┼────────────────┐
          │                │                │
         RBAC          Permissions       Policies
          │                │                │
        Roles          Actions         Business Rules
          │                │                │
     USER/FARMER       product:        ownerId
       /ADMIN           update          === user.id
          │                │                │
          └────────────────┼────────────────┘
                           │
                         Guard
                           │
                           ▼
                     Controller
```

---

# একটি বাস্তব Example

ধরুন endpoint:

```http
DELETE /products/123
```

আমরা চাই:

> শুধু ADMIN product delete করতে পারবে।

Controller:

```ts
@Delete(':id')
@Permissions(Permission.PRODUCT_DELETE)
@UseGuards(
  JwtAuthGuard,
  PermissionsGuard,
)
deleteProduct(
  @Param('id') id: string,
) {
  return this.productsService.delete(id);
}
```

User:

```json
{
  "id": "123",
  "role": "FARMER"
}
```

তার permissions:

```text
product:read
product:create
product:update
```

Required:

```text
product:delete
```

তাই:

```text
Required:
product:delete

User has:
product:read
product:create
product:update

Match?
❌

403 Forbidden
```

ADMIN:

```text
ADMIN
 ↓
product:delete
 ↓
Permission exists
 ↓
✅
 ↓
Controller
```

---

# Role + Permission + Policy একসাথে

একটা বাস্তব application-এ তিনটা layer এমন হতে পারে:

### Layer 1 — Authentication

```text
JWT
 ↓
Who is this user?
 ↓
req.user
```

### Layer 2 — Role/Permission

```text
Does this user have
product:update?
```

### Layer 3 — Policy

```text
Can this user update
THIS particular product?
```

পুরো flow:

```text
Request
   ↓
JWT Guard
   ↓
Authenticated?
   │
   ├── NO → 401
   │
   └── YES
        ↓
   Permission Guard
        ↓
   Has permission?
        │
        ├── NO → 403
        │
        └── YES
             ↓
          Policy
             ↓
      Business rule valid?
             │
             ├── NO → 403
             │
             └── YES
                  ↓
              Controller
                  ↓
               Service
```

এটাই একটা **production-grade NestJS authorization architecture-এর মূল ধারণা**।
