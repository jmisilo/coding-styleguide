# Production Systems Principles

This guide outlines the best practices for building applications & systems.

## What to care about?

While building applications we should care about speed and quality, but the most important principle is to make sure the system is safe. By safe we mean:

- we control the input into the system - we validate and sanitize what comes into APIs/services (e.g. with `zod` or similar libraries, in TypeScript codebases)
- we handle errors and failures gracefully - we don't crash or leak sensitive data
- (additional) we log and monitor.

After that we can focus on speed of the development and quality of the UX & code.
