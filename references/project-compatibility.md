# Project compatibility baselines

Use these as detector baselines when `wathba integrate explain` reports an
unsupported target. They are deliberately small reference packages, not
recommendations to replace an application's current dependency versions.
`packageManager` and `engines.node` declarations are optional; the CLI probes
the actual tools before mutation.

Every supported target needs:

- `package.json`
- `tsconfig.json`
- a `typescript` dependency in `dependencies` or `devDependencies`
- Node 24 at execution time
- npm or pnpm

## Generic Node server

```json
{
  "name": "wathba-generic-reference",
  "private": true,
  "type": "module",
  "scripts": {
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {},
  "devDependencies": {
    "typescript": "5.9.3"
  }
}
```

Add `tsconfig.json` with strict TypeScript settings and a server entry such as
`src/index.ts`.

## Next.js server

```json
{
  "name": "wathba-next-reference",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "15.5.7",
    "react": "19.1.0",
    "react-dom": "19.1.0"
  },
  "devDependencies": {
    "typescript": "5.9.3"
  }
}
```

Add `tsconfig.json` and an App Router boundary such as `app/page.tsx`. The CLI
generates the Wathba wrapper under `src/server/wathba/`; call it only from a
server action, route handler, or other server-only module.

## NestJS Fastify

```json
{
  "name": "wathba-nest-reference",
  "private": true,
  "scripts": {
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@nestjs/common": "11.0.0",
    "@nestjs/core": "11.0.0",
    "@nestjs/platform-fastify": "11.0.0"
  },
  "devDependencies": {
    "typescript": "5.9.3"
  }
}
```

Add `tsconfig.json` and bootstrap Nest with `@nestjs/platform-fastify`.
`@nestjs/platform-express` is not an accepted Wathba target.

Before any integration mutation, run:

```sh
wathba integrate explain messaging.otp --project-dir . --json --no-input
```
