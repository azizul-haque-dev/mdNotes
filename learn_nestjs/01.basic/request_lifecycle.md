Request Lifecycle
এটা খুব ভালোভাবে বুঝবেন:

Request
  ↓
Middleware
  ↓
Guards
  ↓
Interceptors
  ↓
Pipes
  ↓
Controller
  ↓
Service
  ↓
Repository / Database
  ↓
Response

বিশেষ করে পার্থক্য:

Middleware  → request preprocessing
Guard       → authentication / authorization
Pipe        → validation / transformation
Interceptor → before/after request logic
Filter      → exception handling

এগুলো production NestJS-এর foundation।

