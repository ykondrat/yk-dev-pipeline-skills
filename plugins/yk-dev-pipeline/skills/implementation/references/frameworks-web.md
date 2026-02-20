# Web Framework Patterns — Deep Reference

Backend framework patterns for Express, Fastify, Hono, and Next.js API routes.
Load this reference when the project uses any of these frameworks.

---

## 1. Express.js

### Project Setup

```typescript
// src/app.ts — Express application factory
import express, { type Express } from "express";
import helmet from "helmet";
import cors from "cors";
import { pinoHttp } from "pino-http";
import { errorHandler, notFoundHandler } from "./middleware/error.js";
import { userRoutes } from "./routes/user.routes.js";
import { healthRoutes } from "./routes/health.routes.js";

export function createApp(): Express {
  const app = express();

  // Security & parsing
  app.use(helmet());
  app.use(cors({ origin: process.env.CORS_ORIGIN, credentials: true }));
  app.use(express.json({ limit: "1mb" }));

  // Logging
  app.use(pinoHttp({ autoLogging: true }));

  // Routes
  app.use("/health", healthRoutes);
  app.use("/api/v1/users", userRoutes);

  // Error handling (must be last)
  app.use(notFoundHandler);
  app.use(errorHandler);

  return app;
}
```

```typescript
// src/server.ts — Server entry point with graceful shutdown
import { createApp } from "./app.js";
import { logger } from "./lib/logger.js";

const app = createApp();
const port = parseInt(process.env.PORT ?? "3000", 10);

const server = app.listen(port, () => {
  logger.info({ port }, "Server started");
});

// Graceful shutdown
const shutdown = async (signal: string) => {
  logger.info({ signal }, "Shutting down");
  server.close(() => process.exit(0));
  setTimeout(() => process.exit(1), 10_000);
};

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));
```

### Router Pattern

```typescript
// src/routes/user.routes.ts
import { Router } from "express";
import { validate } from "../middleware/validate.js";
import { createUserSchema, updateUserSchema } from "../schemas/user.schema.js";
import * as userController from "../controllers/user.controller.js";
import { authenticate } from "../middleware/auth.js";

export const userRoutes = Router();

userRoutes.post("/", validate(createUserSchema), userController.create);
userRoutes.get("/:id", userController.getById);
userRoutes.put("/:id", authenticate, validate(updateUserSchema), userController.update);
userRoutes.delete("/:id", authenticate, userController.remove);
```

### Middleware Patterns

```typescript
// src/middleware/validate.ts — Zod validation middleware
import type { Request, Response, NextFunction } from "express";
import type { ZodSchema } from "zod";
import { AppError } from "../lib/errors.js";

export function validate(schema: ZodSchema) {
  return (req: Request, _res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      throw new AppError("VALIDATION_ERROR", 400, result.error.flatten());
    }
    req.body = result.data;
    next();
  };
}
```

```typescript
// src/middleware/auth.ts — JWT authentication
import type { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";
import { AppError } from "../lib/errors.js";

export function authenticate(req: Request, _res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith("Bearer ")) {
    throw new AppError("UNAUTHORIZED", 401, "Missing token");
  }

  try {
    const payload = jwt.verify(header.slice(7), process.env.JWT_SECRET!);
    req.user = payload as TokenPayload;
    next();
  } catch {
    throw new AppError("UNAUTHORIZED", 401, "Invalid token");
  }
}
```

```typescript
// src/middleware/error.ts — Centralized error handling
import type { Request, Response, NextFunction } from "express";
import { AppError } from "../lib/errors.js";
import { logger } from "../lib/logger.js";

export function notFoundHandler(req: Request, _res: Response, next: NextFunction) {
  next(new AppError("NOT_FOUND", 404, `Route ${req.method} ${req.path} not found`));
}

// Express error handlers require all 4 parameters
export function errorHandler(
  err: Error,
  _req: Request,
  res: Response,
  _next: NextFunction,
) {
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      error: { code: err.code, message: err.message, details: err.details },
    });
    return;
  }

  // Unexpected error — log full stack, return generic message
  logger.error({ err }, "Unhandled error");
  res.status(500).json({
    error: { code: "INTERNAL_ERROR", message: "Something went wrong" },
  });
}
```

### Controller Pattern

```typescript
// src/controllers/user.controller.ts
import type { Request, Response, NextFunction } from "express";
import * as userService from "../services/user.service.js";

// Wrap async handlers to forward errors to Express error middleware
function asyncHandler(fn: (req: Request, res: Response) => Promise<void>) {
  return (req: Request, res: Response, next: NextFunction) => {
    fn(req, res).catch(next);
  };
}

export const create = asyncHandler(async (req, res) => {
  const user = await userService.create(req.body);
  res.status(201).json({ data: user });
});

export const getById = asyncHandler(async (req, res) => {
  const user = await userService.getById(req.params.id);
  res.json({ data: user });
});
```

### Express Key Rules
- **Always use async error forwarding** — Express doesn't catch async errors. Use `asyncHandler` wrapper or `express-async-errors` package.
- **Error middleware must have 4 parameters** — `(err, req, res, next)`. Even if you don't use `next`, include it.
- **Order matters** — Middleware runs in registration order. Error handlers must be registered last.
- **Don't mutate `req` freely** — Extend `Request` type via declaration merging in a `.d.ts` file.
- **Use `express.json({ limit })` always** — Prevent large payload DoS.

---

## 2. Fastify

### Project Setup

```typescript
// src/app.ts — Fastify application factory
import Fastify, { type FastifyInstance } from "fastify";
import fastifyCors from "@fastify/cors";
import fastifyHelmet from "@fastify/helmet";
import { userRoutes } from "./routes/user.routes.js";
import { healthRoutes } from "./routes/health.routes.js";
import { errorHandler } from "./plugins/error-handler.js";

export async function buildApp(): Promise<FastifyInstance> {
  const app = Fastify({
    logger: {
      level: process.env.LOG_LEVEL ?? "info",
      transport:
        process.env.NODE_ENV === "development"
          ? { target: "pino-pretty" }
          : undefined,
    },
    ajv: {
      customOptions: { removeAdditional: "all", coerceTypes: false },
    },
  });

  // Plugins
  await app.register(fastifyHelmet);
  await app.register(fastifyCors, { origin: process.env.CORS_ORIGIN });

  // Error handler
  app.setErrorHandler(errorHandler);

  // Routes
  await app.register(healthRoutes, { prefix: "/health" });
  await app.register(userRoutes, { prefix: "/api/v1/users" });

  return app;
}
```

```typescript
// src/server.ts
import { buildApp } from "./app.js";

const app = await buildApp();

try {
  await app.listen({ port: parseInt(process.env.PORT ?? "3000", 10), host: "0.0.0.0" });
} catch (err) {
  app.log.error(err);
  process.exit(1);
}
```

### Route Pattern with Schema Validation

```typescript
// src/routes/user.routes.ts
import type { FastifyInstance, FastifyRequest, FastifyReply } from "fastify";
import { Type, type Static } from "@sinclair/typebox";
import * as userService from "../services/user.service.js";

const CreateUserBody = Type.Object({
  email: Type.String({ format: "email" }),
  name: Type.String({ minLength: 1, maxLength: 100 }),
  password: Type.String({ minLength: 8 }),
});
type CreateUserBody = Static<typeof CreateUserBody>;

const UserParams = Type.Object({
  id: Type.String({ format: "uuid" }),
});
type UserParams = Static<typeof UserParams>;

export async function userRoutes(app: FastifyInstance) {
  app.post<{ Body: CreateUserBody }>(
    "/",
    {
      schema: {
        body: CreateUserBody,
        response: {
          201: Type.Object({
            data: Type.Object({ id: Type.String(), email: Type.String(), name: Type.String() }),
          }),
        },
      },
    },
    async (request, reply) => {
      const user = await userService.create(request.body);
      reply.status(201).send({ data: user });
    },
  );

  app.get<{ Params: UserParams }>(
    "/:id",
    { schema: { params: UserParams } },
    async (request, reply) => {
      const user = await userService.getById(request.params.id);
      reply.send({ data: user });
    },
  );
}
```

### Plugin Pattern (Dependency Injection)

```typescript
// src/plugins/database.ts — Fastify plugin for DB connection
import fp from "fastify-plugin";
import type { FastifyInstance } from "fastify";
import { PrismaClient } from "@prisma/client";

declare module "fastify" {
  interface FastifyInstance {
    prisma: PrismaClient;
  }
}

export default fp(async (app: FastifyInstance) => {
  const prisma = new PrismaClient();
  await prisma.$connect();

  app.decorate("prisma", prisma);

  app.addHook("onClose", async () => {
    await prisma.$disconnect();
  });
});
```

### Error Handler

```typescript
// src/plugins/error-handler.ts
import type { FastifyError, FastifyRequest, FastifyReply } from "fastify";
import { AppError } from "../lib/errors.js";

export function errorHandler(
  error: FastifyError | AppError,
  request: FastifyRequest,
  reply: FastifyReply,
) {
  if (error instanceof AppError) {
    reply.status(error.statusCode).send({
      error: { code: error.code, message: error.message, details: error.details },
    });
    return;
  }

  // Fastify validation errors
  if (error.validation) {
    reply.status(400).send({
      error: {
        code: "VALIDATION_ERROR",
        message: "Invalid request",
        details: error.validation,
      },
    });
    return;
  }

  request.log.error({ err: error }, "Unhandled error");
  reply.status(500).send({
    error: { code: "INTERNAL_ERROR", message: "Something went wrong" },
  });
}
```

### Fastify Key Rules
- **Use the plugin system** — Encapsulate routes, DB connections, and shared services in plugins with `fastify-plugin`.
- **Schema-first validation** — Use JSON Schema (or TypeBox) for automatic request/response validation and serialization.
- **Decorators for DI** — Use `app.decorate()` to attach shared services (DB, cache) to the Fastify instance.
- **Async all the way** — Fastify is async-native. Return values from handlers instead of calling `reply.send()` when possible.
- **Don't mix callback and async** — Pick one style per route. Prefer async.
- **Register cleanup hooks** — Use `onClose` hook for graceful shutdown of connections.

---

## 3. Hono

### Project Setup

```typescript
// src/app.ts — Hono application
import { Hono } from "hono";
import { cors } from "hono/cors";
import { secureHeaders } from "hono/secure-headers";
import { logger } from "hono/logger";
import { zValidator } from "@hono/zod-validator";
import { userRoutes } from "./routes/user.routes.js";
import { healthRoutes } from "./routes/health.routes.js";
import { errorHandler } from "./middleware/error.js";

// Type-safe env bindings
type Env = {
  Variables: {
    user: TokenPayload;
  };
};

const app = new Hono<Env>();

// Global middleware
app.use("*", logger());
app.use("*", secureHeaders());
app.use("*", cors({ origin: process.env.CORS_ORIGIN ?? "*" }));

// Error handler
app.onError(errorHandler);

// Routes
app.route("/health", healthRoutes);
app.route("/api/v1/users", userRoutes);

export default app;
```

```typescript
// src/index.ts — Node.js entry point
import { serve } from "@hono/node-server";
import app from "./app.js";

serve({ fetch: app.fetch, port: parseInt(process.env.PORT ?? "3000", 10) }, (info) => {
  console.log(`Server started on port ${info.port}`);
});
```

### Route Pattern with Zod Validation

```typescript
// src/routes/user.routes.ts
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import { z } from "zod";
import * as userService from "../services/user.service.js";
import { authenticate } from "../middleware/auth.js";

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  password: z.string().min(8),
});

export const userRoutes = new Hono();

userRoutes.post("/", zValidator("json", createUserSchema), async (c) => {
  const body = c.req.valid("json");
  const user = await userService.create(body);
  return c.json({ data: user }, 201);
});

userRoutes.get("/:id", async (c) => {
  const user = await userService.getById(c.req.param("id"));
  return c.json({ data: user });
});

userRoutes.delete("/:id", authenticate, async (c) => {
  await userService.remove(c.req.param("id"));
  return c.body(null, 204);
});
```

### Middleware Pattern

```typescript
// src/middleware/auth.ts
import type { MiddlewareHandler } from "hono";
import { HTTPException } from "hono/http-exception";
import { verify } from "hono/jwt";

export const authenticate: MiddlewareHandler = async (c, next) => {
  const header = c.req.header("Authorization");
  if (!header?.startsWith("Bearer ")) {
    throw new HTTPException(401, { message: "Missing token" });
  }

  try {
    const payload = await verify(header.slice(7), process.env.JWT_SECRET!);
    c.set("user", payload as TokenPayload);
    await next();
  } catch {
    throw new HTTPException(401, { message: "Invalid token" });
  }
};
```

### Hono Key Rules
- **Web Standard APIs** — Hono uses `Request`/`Response` Web APIs. Works on Node.js, Deno, Bun, Cloudflare Workers.
- **Use `HTTPException` for errors** — Hono's built-in exception class for consistent error responses.
- **Validators are middleware** — Use `zValidator` from `@hono/zod-validator` for type-safe request validation.
- **Context object `c`** — All request/response data accessed through the context. Use `c.set()`/`c.get()` for passing data between middleware.
- **Lightweight** — Hono has zero dependencies. Use middleware packages selectively.
- **RPC mode** — Use `hono/client` for type-safe client-server communication (end-to-end type safety).

---

## 4. Next.js API Routes (App Router)

### Route Handler Pattern

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import * as userService from "@/lib/services/user.service";

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  password: z.string().min(8),
});

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = parseInt(searchParams.get("page") ?? "1", 10);
  const limit = parseInt(searchParams.get("limit") ?? "20", 10);

  const result = await userService.list({ page, limit });
  return NextResponse.json({ data: result.items, meta: result.meta });
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const parsed = createUserSchema.safeParse(body);

  if (!parsed.success) {
    return NextResponse.json(
      { error: { code: "VALIDATION_ERROR", details: parsed.error.flatten() } },
      { status: 400 },
    );
  }

  const user = await userService.create(parsed.data);
  return NextResponse.json({ data: user }, { status: 201 });
}
```

### Dynamic Route Pattern

```typescript
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from "next/server";
import * as userService from "@/lib/services/user.service";
import { withAuth } from "@/lib/middleware/auth";

type Params = { params: Promise<{ id: string }> };

export async function GET(_request: NextRequest, { params }: Params) {
  const { id } = await params;
  const user = await userService.getById(id);

  if (!user) {
    return NextResponse.json(
      { error: { code: "NOT_FOUND", message: "User not found" } },
      { status: 404 },
    );
  }

  return NextResponse.json({ data: user });
}

export const DELETE = withAuth(async (_request: NextRequest, { params }: Params) => {
  const { id } = await params;
  await userService.remove(id);
  return new NextResponse(null, { status: 204 });
});
```

### Middleware Pattern

```typescript
// middleware.ts (root-level — runs on Edge by default)
import { NextRequest, NextResponse } from "next/server";

export function middleware(request: NextRequest) {
  // Rate limiting, auth checks, redirects, etc.
  const token = request.headers.get("authorization");

  if (request.nextUrl.pathname.startsWith("/api/admin") && !token) {
    return NextResponse.json(
      { error: { code: "UNAUTHORIZED" } },
      { status: 401 },
    );
  }

  // Add request ID for tracing
  const requestId = crypto.randomUUID();
  const headers = new Headers(request.headers);
  headers.set("x-request-id", requestId);

  const response = NextResponse.next({ request: { headers } });
  response.headers.set("x-request-id", requestId);
  return response;
}

export const config = {
  matcher: ["/api/:path*"],
};
```

### Auth Wrapper (Higher-Order Handler)

```typescript
// lib/middleware/auth.ts
import { NextRequest, NextResponse } from "next/server";
import { verifyToken } from "@/lib/auth";

type RouteHandler = (
  request: NextRequest,
  context: { params: Promise<Record<string, string>> },
) => Promise<NextResponse>;

export function withAuth(handler: RouteHandler): RouteHandler {
  return async (request, context) => {
    const header = request.headers.get("authorization");
    if (!header?.startsWith("Bearer ")) {
      return NextResponse.json(
        { error: { code: "UNAUTHORIZED" } },
        { status: 401 },
      );
    }

    try {
      await verifyToken(header.slice(7));
      return handler(request, context);
    } catch {
      return NextResponse.json(
        { error: { code: "UNAUTHORIZED" } },
        { status: 401 },
      );
    }
  };
}
```

### Next.js API Key Rules
- **File-system routing** — Each `route.ts` file maps to a URL segment. Export named functions matching HTTP methods (`GET`, `POST`, `PUT`, `DELETE`).
- **`params` is a Promise in Next.js 15+** — Always `await params` before accessing route parameters.
- **Root `middleware.ts` runs on Edge** — Keep it lightweight. No Node.js APIs (no `fs`, `path`, etc.). For heavy logic, use route-level wrappers.
- **No shared state** — Route handlers are serverless. Don't rely on in-memory state between requests. Use external stores.
- **Use `@/` path alias** — Configure in `tsconfig.json` for clean imports.
- **Server Actions for mutations** — For full-stack Next.js apps, prefer Server Actions over API routes for form submissions.

---

## 5. Cross-Framework Patterns

### Shared Error Class

```typescript
// src/lib/errors.ts — Framework-agnostic error class
export class AppError extends Error {
  constructor(
    public readonly code: string,
    public readonly statusCode: number,
    public readonly details?: unknown,
  ) {
    super(code);
    this.name = "AppError";
  }

  static badRequest(message: string, details?: unknown) {
    return new AppError(message, 400, details);
  }

  static notFound(resource: string) {
    return new AppError("NOT_FOUND", 404, `${resource} not found`);
  }

  static unauthorized(message = "Unauthorized") {
    return new AppError("UNAUTHORIZED", 401, message);
  }

  static forbidden(message = "Forbidden") {
    return new AppError("FORBIDDEN", 403, message);
  }
}
```

### Service Layer (Framework-Independent)

```typescript
// src/services/user.service.ts — Pure business logic, no framework imports
import type { UserRepository } from "../repositories/user.repository.js";
import { AppError } from "../lib/errors.js";
import { hashPassword } from "../lib/auth.js";

export class UserService {
  constructor(private readonly repo: UserRepository) {}

  async create(input: CreateUserInput) {
    const existing = await this.repo.findByEmail(input.email);
    if (existing) {
      throw AppError.badRequest("EMAIL_EXISTS", "Email already registered");
    }

    const hashedPassword = await hashPassword(input.password);
    return this.repo.create({ ...input, password: hashedPassword });
  }

  async getById(id: string) {
    const user = await this.repo.findById(id);
    if (!user) throw AppError.notFound("User");
    return user;
  }
}
```

### Request Logging Middleware (All Frameworks)

Always log:
- Request ID (correlation)
- Method + path
- Status code
- Duration (ms)
- User ID (if authenticated)

Never log:
- Request/response bodies (may contain PII)
- Authorization headers
- Passwords or tokens

### Health Check Endpoint

```typescript
// Pattern works for all frameworks — adapt to framework's response API
async function healthCheck() {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    uptime: process.uptime(),
  };

  const healthy = Object.values(checks).every(
    (c) => typeof c === "number" || c === true,
  );

  return {
    status: healthy ? 200 : 503,
    body: { status: healthy ? "ok" : "degraded", checks },
  };
}
```

---

## 6. Framework Selection Guide

| Factor | Express | Fastify | Hono | Next.js API |
|--------|---------|---------|------|-------------|
| **Best for** | Legacy/broad ecosystem | High-performance APIs | Edge/multi-runtime | Full-stack apps |
| **Performance** | Baseline | ~2-3x Express | ~3-4x Express | Depends on deployment |
| **Validation** | Manual (Zod) | Built-in (JSON Schema) | Manual (Zod) | Manual (Zod) |
| **TypeScript DX** | Moderate | Good (generics) | Excellent (inference) | Good |
| **Middleware** | Sequential chain | Plugin + hooks | Compose pattern | Edge + wrappers |
| **Learning curve** | Low | Medium | Low | Medium (framework context) |
| **Ecosystem** | Largest | Growing | Growing | Next.js ecosystem |
| **Deployment** | Any Node.js | Any Node.js | Node/Deno/Bun/Edge | Vercel/self-hosted |

### When to Use What
- **Express**: Existing Express codebase, team familiarity, vast middleware ecosystem needed
- **Fastify**: New API project needing performance, built-in schema validation, plugin architecture
- **Hono**: Edge deployment (Cloudflare Workers), multi-runtime support, minimal footprint
- **Next.js API Routes**: Already using Next.js for frontend, simple API alongside SSR/RSC app
