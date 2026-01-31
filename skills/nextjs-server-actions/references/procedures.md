# Procedures Reference

Procedures are middleware that add context and authorization to server actions.

## Basic Procedure

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure } from "zsa";

export const baseProcedure = createServerActionProcedure().handler(async () => {
  // Return value becomes `ctx` in actions
  return {
    timestamp: new Date(),
  };
});
```

## Authentication Procedure

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure } from "zsa";
import { auth } from "@/lib/auth";
import { headers, cookies } from "next/headers";

export const authedProcedure = createServerActionProcedure().handler(
  async () => {
    // Using NextAuth
    const session = await auth();

    if (!session?.user) {
      throw new Error("Not authenticated");
    }

    return {
      user: {
        id: session.user.id!,
        email: session.user.email!,
        name: session.user.name,
        role: session.user.role as "user" | "admin",
      },
    };
  }
);

// Alternative: Custom auth with cookies
export const customAuthProcedure = createServerActionProcedure().handler(
  async () => {
    const cookieStore = await cookies();
    const token = cookieStore.get("auth-token")?.value;

    if (!token) {
      throw new Error("No auth token");
    }

    const user = await verifyToken(token);

    if (!user) {
      throw new Error("Invalid token");
    }

    return { user };
  }
);
```

## Chaining Procedures

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure } from "zsa";
import z from "zod";

// Base: Authentication
export const authedProcedure = createServerActionProcedure().handler(
  async () => {
    const session = await auth();
    if (!session?.user) throw new Error("Not authenticated");
    return { user: session.user };
  }
);

// Chain: Admin check
export const adminProcedure = createServerActionProcedure(authedProcedure)
  .handler(async ({ ctx }) => {
    // ctx contains user from authedProcedure
    if (ctx.user.role !== "admin") {
      throw new Error("Admin access required");
    }
    return ctx; // Pass through context
  });

// Chain: Super admin check
export const superAdminProcedure = createServerActionProcedure(adminProcedure)
  .handler(async ({ ctx }) => {
    if (ctx.user.role !== "super_admin") {
      throw new Error("Super admin access required");
    }
    return ctx;
  });
```

## Procedure with Input

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure } from "zsa";
import z from "zod";

// Procedure that requires specific input
export const ownsResourceProcedure = createServerActionProcedure(authedProcedure)
  .input(
    z.object({
      resourceId: z.string(),
      resourceType: z.enum(["post", "comment", "file"]),
    })
  )
  .handler(async ({ ctx, input }) => {
    const { resourceId, resourceType } = input;
    
    const resource = await db[resourceType].findUnique({
      where: { id: resourceId },
    });

    if (!resource) {
      throw new Error("Resource not found");
    }

    if (resource.ownerId !== ctx.user.id) {
      throw new Error("Not authorized to access this resource");
    }

    return {
      ...ctx,
      resource,
    };
  });

// Usage - postId is automatically required
export const updatePost = ownsResourceProcedure
  .createServerAction()
  .input(
    z.object({
      resourceId: z.string(), // Required by procedure
      resourceType: z.literal("post"),
      title: z.string(),
      content: z.string(),
    })
  )
  .handler(async ({ input, ctx }) => {
    // ctx.resource is the validated post
    return db.post.update({
      where: { id: ctx.resource.id },
      data: { title: input.title, content: input.content },
    });
  });
```

## Procedure Callbacks

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure } from "zsa";

// Callbacks apply to ALL actions using this procedure
export const loggingProcedure = createServerActionProcedure()
  .onStart(async () => {
    console.log("Action started");
  })
  .onSuccess(async ({ data }) => {
    console.log("Action succeeded:", data);
  })
  .onError(async ({ err }) => {
    console.error("Action failed:", err);
    await reportError(err);
  })
  .onComplete(async () => {
    console.log("Action completed");
  })
  .handler(async () => {
    return {};
  });

// Chain with other procedures
export const authedLoggingProcedure = createServerActionProcedure(loggingProcedure)
  .handler(async () => {
    const session = await auth();
    if (!session) throw new Error("Not authenticated");
    return { user: session.user };
  });
```

## Rate Limiting Procedure

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure } from "zsa";
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";
import { headers } from "next/headers";

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, "10 s"), // 10 requests per 10 seconds
});

export const rateLimitedProcedure = createServerActionProcedure().handler(
  async () => {
    const headersList = await headers();
    const ip = headersList.get("x-forwarded-for") ?? "127.0.0.1";

    const { success, limit, reset, remaining } = await ratelimit.limit(ip);

    if (!success) {
      throw new Error(`Rate limit exceeded. Try again in ${reset - Date.now()}ms`);
    }

    return {
      rateLimit: { limit, remaining, reset },
    };
  }
);

// Combine with auth
export const authedRateLimitedProcedure = createServerActionProcedure(
  rateLimitedProcedure
)
  .handler(async ({ ctx }) => {
    const session = await auth();
    if (!session) throw new Error("Not authenticated");
    return { ...ctx, user: session.user };
  });
```

## Organization/Tenant Procedure

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure } from "zsa";
import z from "zod";

export const orgProcedure = createServerActionProcedure(authedProcedure)
  .input(z.object({ orgId: z.string() }))
  .handler(async ({ ctx, input }) => {
    // Check user belongs to organization
    const membership = await db.orgMembership.findFirst({
      where: {
        userId: ctx.user.id,
        orgId: input.orgId,
      },
      include: { organization: true },
    });

    if (!membership) {
      throw new Error("Not a member of this organization");
    }

    return {
      ...ctx,
      org: membership.organization,
      membership,
    };
  });

// Admin within organization
export const orgAdminProcedure = createServerActionProcedure(orgProcedure)
  .handler(async ({ ctx }) => {
    if (ctx.membership.role !== "admin" && ctx.membership.role !== "owner") {
      throw new Error("Admin access required");
    }
    return ctx;
  });
```

## Feature Flag Procedure

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure } from "zsa";
import z from "zod";

export const featureFlagProcedure = createServerActionProcedure(authedProcedure)
  .input(z.object({ feature: z.string() }))
  .handler(async ({ ctx, input }) => {
    const hasFeature = await checkFeatureFlag(ctx.user.id, input.feature);

    if (!hasFeature) {
      throw new Error(`Feature '${input.feature}' is not enabled`);
    }

    return ctx;
  });

// Usage
export const betaFeatureAction = featureFlagProcedure
  .createServerAction()
  .input(z.object({ feature: z.literal("beta-editor"), content: z.string() }))
  .handler(async ({ input }) => {
    // Feature is guaranteed to be enabled
    return processWithBetaEditor(input.content);
  });
```

## Accessing Request Context

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure } from "zsa";
import { headers, cookies } from "next/headers";

export const requestContextProcedure = createServerActionProcedure().handler(
  async () => {
    const headersList = await headers();
    const cookieStore = await cookies();

    return {
      ip: headersList.get("x-forwarded-for") ?? "unknown",
      userAgent: headersList.get("user-agent") ?? "unknown",
      locale: headersList.get("accept-language")?.split(",")[0] ?? "en",
      theme: cookieStore.get("theme")?.value ?? "light",
    };
  }
);
```

## Error Handling in Procedures

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure, ZSAError } from "zsa";

export const authedProcedure = createServerActionProcedure()
  .onError(async ({ err }) => {
    // Log all errors from this procedure's actions
    console.error("Procedure error:", err);

    // Custom error handling
    if (err.code === "NOT_AUTHORIZED") {
      await logSecurityEvent(err);
    }
  })
  .handler(async () => {
    const session = await auth();

    if (!session) {
      // Throw with specific code
      throw new ZSAError("NOT_AUTHORIZED", "Please sign in to continue");
    }

    return { user: session.user };
  });
```

## Reusable Procedure Factory

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure } from "zsa";
import z from "zod";

// Factory function for permission-based procedures
export function createPermissionProcedure(permission: string) {
  return createServerActionProcedure(authedProcedure).handler(async ({ ctx }) => {
    const hasPermission = await checkPermission(ctx.user.id, permission);

    if (!hasPermission) {
      throw new Error(`Missing permission: ${permission}`);
    }

    return ctx;
  });
}

// Create specific permission procedures
export const canCreatePostProcedure = createPermissionProcedure("posts:create");
export const canDeletePostProcedure = createPermissionProcedure("posts:delete");
export const canManageUsersProcedure = createPermissionProcedure("users:manage");

// Usage
export const createPost = canCreatePostProcedure
  .createServerAction()
  .input(z.object({ title: z.string() }))
  .handler(async ({ input, ctx }) => {
    return db.post.create({
      data: { title: input.title, authorId: ctx.user.id },
    });
  });
```

## Complete Procedure Stack Example

```typescript
// lib/procedures.ts
"use server";

import { createServerActionProcedure } from "zsa";
import z from "zod";

// Layer 1: Base with logging
const baseProcedure = createServerActionProcedure()
  .onStart(async () => console.log("[Action] Started"))
  .onComplete(async () => console.log("[Action] Completed"))
  .handler(async () => ({}));

// Layer 2: Rate limiting
const rateLimitedProcedure = createServerActionProcedure(baseProcedure)
  .handler(async () => {
    await checkRateLimit();
    return {};
  });

// Layer 3: Authentication
const authedProcedure = createServerActionProcedure(rateLimitedProcedure)
  .handler(async () => {
    const session = await auth();
    if (!session) throw new Error("Not authenticated");
    return { user: session.user };
  });

// Layer 4: Organization context
const orgProcedure = createServerActionProcedure(authedProcedure)
  .input(z.object({ orgId: z.string() }))
  .handler(async ({ ctx, input }) => {
    const org = await getOrgWithMembership(ctx.user.id, input.orgId);
    if (!org) throw new Error("Not a member");
    return { ...ctx, org };
  });

// Layer 5: Admin check
const orgAdminProcedure = createServerActionProcedure(orgProcedure)
  .handler(async ({ ctx }) => {
    if (!ctx.org.isAdmin) throw new Error("Not an admin");
    return ctx;
  });

// Export for use
export {
  baseProcedure,
  rateLimitedProcedure,
  authedProcedure,
  orgProcedure,
  orgAdminProcedure,
};
```