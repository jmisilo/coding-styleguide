# Next.js (App Router) Conventions

## Repository Structure (Recommended Boundaries)

Suggested layering (rename folders to match your repo):

- `src/app`: routing + application/domain code (Next.js App Router).
- `src/features`: large reusable feature modules (mini-domains).
- `src/infrastructure`: external systems (DB, auth, payments, email, analytics, storage, rate-limit, integrations).
- `src/ui`: reusable UI components that cross domain boundaries.
- `src/utilities`: cross-cutting helpers/HOFs.
- `src/schemas` + `src/types`: Zod schemas + derived TS types.

## Server vs Client

- Default to Server Components. Add `'use client'` only when you need hooks, state, effects, browser APIs, or client-only libraries.
- Mark server-only modules with `import 'server-only';` to prevent accidental client bundling.
- Mark server actions with `'use server';` and keep them small/typed.

## Route Handlers

- Validate request input early (`zod`), return explicit `{ message, code }` JSON on expected failures.
- Prefer `NextResponse.json(...)` for structured errors.
- If you support Edge and a vendor SDK is incompatible, use `fetch` and keep the logic isolated behind a small adapter.

## Route Params / Segment Params

Use `params: Promise<{ slug: string }>` and `const { slug } = await params`.

## Server Actions: Standard Wrapper (`withProtectedAction` pattern)

Use a thin wrapper for the common concerns:

- auth/session lookup (usually from `next/headers`)
- payload validation (Zod)
- output validation (optional but recommended)
- unified return shape: `{ data, failure }` (avoid throwing for expected business failures)

Reference implementation (adapt to your auth + logger):

```ts
import { headers } from 'next/headers';
import 'server-only';
import { type z } from 'zod';

type Success<T extends object> = { data: T; failure: null };
type Failure = { data: null; failure: { message: string; code?: string } };

export const withProtectedAction = <
  P extends Record<string, unknown>,
  O extends Record<string, unknown>,
  TUser,
>(
  handler: (payload: P & { user: TUser }) => Promise<Success<O> | Failure>,
  opts: { actionId: string; schemas: { payload: z.ZodType<P>; output: z.ZodType<O> } },
) => {
  return async (_payload: P): Promise<Success<O> | Failure> => {
    // 1) auth/session lookup (replace getSession with your auth provider)
    const session = await getSession({ headers: await headers() }); // { user: TUser } | null
    if (!session) return { data: null, failure: { message: 'Unauthorized', code: 'unauthorized' } };

    // 2) payload validation
    const parsed = opts.schemas.payload.safeParse(_payload);
    if (!parsed.success)
      return { data: null, failure: { message: 'Invalid payload', code: 'invalid_payload' } };

    // 3) execute
    const result = await handler({ ...parsed.data, user: session.user });

    // 4) output validation (if success)
    if (result.data) opts.schemas.output.parse(result.data);

    return result;
  };
};
```

Client usage pattern:

- Call server actions inside `startTransition`.
- Display `result.failure.message` via UI (or toast if no place) instead of `try/catch` for expected failures.

## Caching + Revalidation (Mutation Follow-ups)

- When a server action or route handler mutates data used by routes, explicitly revalidate:
  - `revalidatePath('/some/path')` for path-based invalidation
  - `revalidateTag('tag')` if you use fetch tags (on the server-side)
- Keep revalidation calls next to the mutation (same function) so cache behavior stays discoverable.

## Auth handling (RSC)

To encapsulate auth logic, and utilize RSC, use `withAuth` HOC for server components auth protection. Here is a reference implementation:

```tsx
import { headers } from 'next/headers';
import { permanentRedirect } from 'next/navigation';

import { type FC } from 'react';

import 'server-only';

import { Path } from '@/constants/path.enum';
import { auth } from '@/infrastructure/auth'; // Better Auth library abstraction
import { logger } from '@/infrastructure/logger';
import { type User } from '@/types/auth';

/** @description Higher-order component to wrap a RSC page/component with authentication.*/
export const withAuth = <T extends object, R extends boolean = false>(
  Component: FC<R extends true ? T : T & { user: User }>,
  options: { id?: string; reverse?: R } = {},
): FC<T> => {
  // eslint-disable-next-line react/display-name
  return async (props: T) => {
    const session = await auth.api.getSession({
      headers: await headers(),
    });

    logger.info(`withAuth: session retrieved; is session - ${!!session}`);

    // handle reverse auth (e.g. login/register pages)
    if (options.reverse === true) {
      if (session) {
        if (options.id) {
          logger.error(
            {
              pageId: options.id,
              userId: session.user.id,
            },
            'withAuth (reversed): User is authenticated. Redirecting to dashboard page.',
          );
        }

        return permanentRedirect(Path.Builder);
      }

      // @ts-expect-error typescript doesn't infer the correct type - it should not expect user
      return <Component {...props} />;
    }

    if (!session) {
      if (options.id) {
        logger.error(
          { pageId: options.id },
          'withAuth: User is not authenticated. Redirecting to login page.',
        );
      }

      return permanentRedirect(Path.Login);
    }

    return <Component {...props} user={session.user} />;
  };
};
```

Example usage:

```tsx
import { withAuth } from '@/utilities/auth/with-auth';

// ...

const DashboardPage = withAuth(
  async ({ user }) => {
    const data = await UserDbService.getProfile(user.id)
    
    // ...
  },
  { id: 'dashboard-page' },
)
```
