use services, adapters, scoped to the domain (e.g. location db service)

do not use standalone functions etc
# Interacting with 3rd part services

This guide outlines best practices for integrating and interacting with third-party services in applications.

## What to do?

- **Use Service/Adapter Pattern:** Create dedicated service or adapter static (most of the time) classes that encapsulate all interactions with third-party services. This promotes separation of concerns and makes it easier to manage changes in the third-party API.

Example of usage with DB client & service:

```ts
// ./src/infrastructure/db/services/index.ts
import 'server-only';

import { db } from '../client';

export class DbService {
  protected static client = db;
}

// ./src/infrastructure/db/services/user.ts
export class UserDbService extends DbService {
  static async getUserById(userId: string) {
    // ...
    return this.client.user.findUnique({ where: { id: userId } });
  }
  
  // ...
}
```

Wrap adapters (adjusting data to needs of the service) into static classes as well.

- **Scope to Domain:** Ensure that services/adapters are scoped to specific domains or functionalities (e.g., AIService, PageDbService). This helps in organizing code and makes it easier to locate and maintain.

- **Encapsulate Logic:** All logic related to the third-party service should be encapsulated when possible. Good example is Cloudflare Turnstile HOF for Next.js server actions:

```ts
// ./src/utilities/turnstile/with-turnstile.ts
import 'server-only';
import z from 'zod';
import { validateTurnstileToken } from 'next-turnstile';

const _NotEmptyStringSchema = z.string().min(1);

export class TurnstileError extends Error {
  constructor(message: string = 'Invalid Turnstile token') {
    super(message);

    this.name = 'TurnstileError';
  }
}

// eslint-disable-next-line @typescript-eslint/no-explicit-any
export const withTurnstile = <T extends (...args: any[]) => Promise<any>>(handler: T) => {
  return async (token: string | null, ...args: Parameters<T>): Promise<ReturnType<T>> => {
    const { data: validatedToken, success } = _NotEmptyStringSchema.safeParse(token);

    if (!success) {
      throw new Error('Invalid Turnstile token');
    }

    const turnstile = await validateTurnstileToken({
      token: _NotEmptyStringSchema.parse(validatedToken),
      secretKey: process.env.TURNSTILE_SECRET_KEY,
    });

    if (!turnstile.success) {
      throw new TurnstileError();
    }

    return await handler(...args);
  };
};
```

Usage: 

```ts
// define server action
"use server"

// ...

import { withTurnstile } from '@/utilities/turnstile/with-turnstile';

export const submitContactForm = withTurnstile(async (data: ContactFormShape) => {
  const result = await EmailService.sendContactFormEmail(ContactFormSchema.parse(data));

  if (!result.accepted.length) {
    throw new Error('Failed to submit form');
  }
});
```

Use on the frontend:

```tsx
// ...

const ContactForm = () => {
  // ...
  
  const { turnstile, turnstileToken, setTurnstileToken, verifyTurnstile, unverifyTurnstile } = useFormTurnstile();

  const onSubmit = async (data: ContactFormShape) => {
    try {
      await submitContactForm(turnstileToken, data);

      toast.success(t('success'));

      setTurnstileToken(null);
      turnstile.reset();
    } catch (error) {
      // handle error...
    }
  }
  
  
  return <form onSubmit={onSubmit}>
    {/* form fields */}
    
    <Turnstile
      sitekey={process.env.NEXT_PUBLIC_TURNSTILE_SITE_KEY!}
      onSuccess={verifyTurnstile}
      onError={unverifyTurnstile}
      appearance="execute"
    />
  </form>
}
```

Do not limit yourself to only static methods, not to HOFs. Feel free to use different abstractions as needed, but keep them structured and scoped.

## What NOT to do?

Do not define standalone functions for interacting with third-party services, especially some that are "flying around" the codebase. Prefer structured, well scoped (local/global) solutions, like above.
