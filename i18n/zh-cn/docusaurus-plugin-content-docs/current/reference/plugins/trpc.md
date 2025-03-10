---
description: Plugin for generating tRPC data query routes
sidebar_position: 7
---

# @zenstackhq/trpc

The `@zenstackhq/trpc` plugin generates a [tRPC router](https://trpc.io/docs/router) to make database CRUD calls. You can use it directly as your main router or merge it with other routers to create a more complex setup. The router syntactically mirrors the APIs of a standard Prisma client, including the function names and shapes of parameters (hooks directly use types generated by Prisma). 

:::info
The `@zenstackhq/trpc` plugin depends on the [`@core/zod`](/docs/reference/plugins/zod) plugin to generate Zod schemas for the routers' input, and will enable that plugin automatically if it's not already enabled.
:::

This plugin is based on [prisma-trpc-generator](https://github.com/omar-dulaimi/prisma-trpc-generator). Thanks to [Omar Dulaimi](https://github.com/omar-dulaimi) for making this happen!

## Options

| Name   | Type   | Description      | Required | Default |
| ------ | ------ | ---------------- | -------- | ------- |
| output | String | Output directory (relative to the path of ZModel) | Yes      | |
| generateModels | String, String[] | Array or comma separated string for the models to generate routers for. | No      | All models |
| generateModelActions | String, String[] | Array or comma separated string for actions to generate for each model: `create`, `findUnique`, `update`, etc. | No      | All supported Prisma actions |
| generateClientHelpers | String, String[] | Array or comma separated string for the types of client helpers to generate. Supported values: "react" or "next". See [here](#client-helpers) for more details. | No      | |
| zodSchemasImport | String | Import path for the generated zod schemas. The trpc plugin relies on the `@core/zod` plugin to generate zod schemas for input validation. If you set a custom output location for the zod schemas, you can use this option to override the import path. | No      | @zenstackhq/runtime/zod |

:::info
When `@core/zod` plugin is automatically enabled by the `@zenstackhq/trpc` plugin, if the `@zenstackhq/trpc` plugin has a `generateModels` option specified, it'll be carried over to the `@core/zod` plugin as well.
:::

## Dependencies

- [`@core/zod`](/docs/reference/plugins/zod)

## Details

### Preparing tRPC Context

The generated tRPC routers require a `prisma` field in the context and you need to make sure to include it when creating the context. You can use any PrismaClient instance for it, but most likely you want to use one created with [`enhance`](/docs/reference/runtime-api#enhance) so that the client enforces access control policies.

```ts
export const createContext = async ({ req, res }: CreateNextContextOptions) => {
    const session = await getServerAuthSession({ req, res });
    return {
        session,
        // use access-control-enabled db client
        prisma: enhance(prisma, { user: session?.user }),
    };
};
```

### Client Helpers

TRPC relies on TypeScript's type inference to provide the client-side API call signatures. This is very neat and powerful, but it has some limitations regarding generic APIs. For example, one of the best features of Prisma's API is its output type automatically adapts to the input's shape. e.g.:

```ts
const post = await prisma.post.findFirst({ include: { author: true } });
```

For the code above, the `post` variable is smartly inferred to be of type `Post & { author: User }`. However, if you wrap it with tRPC, such typing adaptivity is lost:

```ts
// trpc router
const router = t.router({
  findFirst: t.procedure
    .input(PostInputSchema)
    .query(({ ctx, input }) => ctx.prisma.findFirst(input))
})

// client side
const { data } = trpc.post.findFirst({ include: { author: true } });
```

Now the `data` field has a fixed `Post` type, even though at runtime it carries the extra `author` field.

To mitigate this limitation, ZenStack's tRPC plugin can generate a "type-fixing" helper that re-enables the inference power from the client side. To use it, you need to specify the `generateClientHelpers` option:

```zmodel
plugin trpc {
  provider = '@zenstackhq/trpc'
  output = 'server/routers/generated'
  generateClientHelpers = 'next'
}
```

Use value "next" if you're using tRPC with Next.js, or "react" if using it with react-query. With that option on, the plugin generates an extra "client" folder containing helpers that do the type fixing. The only code change you need to make is instead of calling the `createTRPCNext` or `createTRPCReact` API, you call the one generated instead.

Here's an example of how to use it with Next.js:

```ts
import { createTRPCNext } from 'server/routers/generated/client/next';
export const trpc = createTRPCNext<AppRouter>({
    ...
});
```

## Example

Here's an example with a blogging app:

```zmodel title='/schema.zmodel'
plugin trpc {
  provider = '@zenstackhq/trpc'
  output = 'server/routers/generated'
  generateModelActions = 'create,update,findUnique,findMany'
}

model User {
  id            String    @id @default(cuid())
  email         String
  posts         Post[]

  // everyone can signup, and user profile is also publicly readable
  @@allow('create,read', true)
}

model Post {
  id        String @id @default(cuid())
  title     String
  published Boolean @default(false)
  author    User @relation(fields: [authorId], references: [id])
  authorId  String

  // author has full access
  @@allow('all', auth() == author)

  // logged-in users can view published posts
  @@allow('read', auth() != null && published)
}
```

```ts title='/server/routers/_app.ts'
import { createRouter as createCRUDRouter } from './generated/routers';
import { initTRPC } from '@trpc/server';
import { type Context } from '../context';

const t = initTRPC.context<Context>().create();

export const appRouter = createCRUDRouter(t.router, t.procedure);
export type AppRouter = typeof appRouter;
```

Check out the [Using With tRPC](/docs/guides/trpc) guide for more details about using ZenStack in a tRPC project.
