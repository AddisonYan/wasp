---
title: Migration from 0.13.X to 0.14.X
---

:::note Are you on 0.11.X or earlier?

This guide only covers the migration from **0.13.X to 0.14.X**. If you are migrating from 0.11.X or earlier, please read the [migration guide from 0.11.X to 0.12.X](./migrate-from-0-11-to-0-12.md) first.

:::

## What's new in 0.14.0?

### Using Prisma Schema file directly

Before 0.14.0, users defined their entities in the `.wasp` file, and Wasp generated the `schema.prisma` file based on that. This approach had some limitations, and users couldn't use some advanced Prisma features.

Wasp now exposes the `schema.prisma` file directly to the user. You now define your entities in the `schema.prisma` file and Wasp uses that to generate the database schema and Prisma client. You can use all the Prisma features directly in the `schema.prisma` file. Simply put, the `schema.prisma` file is now the source of truth for your database schema.

<Tabs>
<TabItem value="before" label="Before">

```wasp title="main.wasp"
app myApp {
  wasp: {
    version: "^0.13.0"
  },
  title: "MyApp",
  db: {
    system: PostgreSQL
  },
}

entity User {=psl
  id       Int @id @default(autoincrement())
  tasks    Task[]
psl=}

entity Task {=psl
  id          Int @id @default(autoincrement())
  description String
  isDone      Boolean
  userId      Int
  user        User @relation(fields: [userId], references: [id])
psl=}
```

</TabItem>
<TabItem value="after" label="After">

```wasp title="main.wasp"
app myApp {
  wasp: {
    version: "^0.13.0"
  },
  title: "MyApp",
}
```

```prisma title="schema.prisma"
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id       Int @id @default(autoincrement())
  tasks    Task[]
}

model Task {
  id          Int @id @default(autoincrement())
  description String
  isDone      Boolean
  userId      Int
  user        User @relation(fields: [userId], references: [id])
}
```

</TabItem>
</Tabs>

### Better auth user API

Wasp introduced a much simpler API for accessing user auth fields like `username`, `email` or `isEmailVerified` on the `user` object. You don't need to use helper functions every time you want to access the user's `username` or do extra steps to get proper typing.

## How to migrate?

### Migrate to the new `schema.prisma` file
To use the new `schema.prisma` file, you need to move your entities from the `.wasp` file to the `schema.prisma` file.

1. **Create a new `schema.prisma` file**

   Create a new file named `schema.prisma` in the root of your project:

   ```c
    .
    ├── main.wasp
    ...
    // highlight-next-line
    ├── schema.prisma
    ├── src
    ├── tsconfig.json
    └── vite.config.ts
   ```
2. **Add the `datasource` block** to the `schema.prisma` file

   This block specifies the database type and connection URL:

   ```prisma title="schema.prisma"
   datasource db {
     provider = "postgresql"
     url      = env("DATABASE_URL")
   }
   ```

   - The `provider` should be either `"postgresql"` or `"sqlite"`. 
   
   - The `url` must be set to `env("DATABASE_URL")` so that Wasp can inject the database URL from the environment variables.

3. **Add the `generator` block** to the `schema.prisma` file

   This block specifies the Prisma Client generator Wasp uses:

   ```prisma title="schema.prisma"
   datasource db {
     provider = "postgresql"
     url      = env("DATABASE_URL")
   }

   // highlight-start
   generator client {
     provider = "prisma-client-js"
   }
    // highlight-end
   ```

   - The `provider` should be set to `"prisma-client-js"`.

4. **Move your entities** to the `schema.prisma` file

  Move the entities from the `.wasp` file to the `schema.prisma` file:

  ```prisma title="schema.prisma"
  datasource db {
    provider = "postgresql"
    url      = env("DATABASE_URL")
  }

  generator client {
    provider = "prisma-client-js"
  }

  // There are some example entities, you should move your entities here
  // highlight-start
  model User {
    id       Int @id @default(autoincrement())
    tasks    Task[]
  }

  model Task {
    id          Int @id @default(autoincrement())
    description String
    isDone      Boolean
    userId      Int
    user        User @relation(fields: [userId], references: [id])
  }
  // highlight-end
  ```

  When moving the entities over, you'll need to change `entity` to `model` and remove the `=psl` and `psl=` tags.

  If you had the following in the `.wasp` file:
  ```wasp title="main.wasp"
  entity Task {=psl
    // Stays the same
  psl=}
  ```

  ... it would look like this in the `schema.prisma` file:
  ```prisma title="schema.prisma"
  model Task {
    // Stays the same
  }
  ```


5. **Run the Wasp CLI** to generate the Prisma client

   Regenerate the Prisma client by running the following command:

   ```bash
   wasp db migrate-dev
   ```

   This command generates the Prisma client based on the `schema.prisma` file.

Now you are using the new `schema.prisma` file!

Read more about the [Prisma Schema File](./data-model/prisma-file.md) and how Wasp uses it to generate the database schema and Prisma client.


### Migrate how you access user auth fields

We had to make a couple of breaking changes to reach the new simpler API.

Follow the steps below to migrate:

1. **Replace the `getUsername` helper** with `user.identities.username.id`

   If you didn't use the `getUsername` helper in your code, you can skip this step.

   This helper changed and it no longer works with the `user` you receive as a prop on a page or through the `context`. You'll need to replace it with `user.identities.username.id`.

   <Tabs>
   <TabItem value="before" label="Before">

   ```tsx title="src/MainPage.tsx"
   import { getUsername, AuthUser } from 'wasp/auth'

   const MainPage = ({ user }: { user: AuthUser }) => {
     const username = getUsername(user)
     // ...
   }
   ```

   ```ts title=src/tasks.ts
   import { getUsername } from 'wasp/auth'

   export const createTask: CreateTask<...>  = async (args, context) => {
       const username = getUsername(context.user)
       // ...
   }
   ```

   </TabItem>
   <TabItem value="after" label="After">

   ```tsx title="src/MainPage.tsx"
   import { AuthUser } from 'wasp/auth'

   const MainPage = ({ user }: { user: AuthUser }) => {
     const username = user.identities.username?.id
     // ...
   }
   ```

   ```ts title=src/tasks.ts
   export const createTask: CreateTask<...>  = async (args, context) => {
       const username = context.user.identities.username?.id
       // ...
   }
   ```

   </TabItem>
   </Tabs>

1. **Replace the `getEmail` helper** with `user.identities.email.id`

   If you didn't use the `getEmail` helper in your code, you can skip this step.

   This helper changed and it no longer works with the `user` you receive as a prop on a page or through the `context`. You'll need to replace it with `user.identities.email.id`.

   <Tabs>
   <TabItem value="before" label="Before">

   ```tsx title="src/MainPage.tsx"
   import { getEmail, AuthUser } from 'wasp/auth'

   const MainPage = ({ user }: { user: AuthUser }) => {
     const email = getEmail(user)
     // ...
   }
   ```

   ```ts title=src/tasks.ts
   import { getEmail } from 'wasp/auth'

   export const createTask: CreateTask<...>  = async (args, context) => {
       const email = getEmail(context.user)
       // ...
   }
   ```

   </TabItem>
   <TabItem value="after" label="After">

   ```tsx title="src/MainPage.tsx"
   import { AuthUser } from 'wasp/auth'

   const MainPage = ({ user }: { user: AuthUser }) => {
     const email = user.identities.email?.id
     // ...
   }
   ```

   ```ts title=src/tasks.ts
   export const createTask: CreateTask<...>  = async (args, context) => {
       const email = context.user.identities.email?.id
       // ...
   }
   ```

   </TabItem>
   </Tabs>

1. **Replace accessing `providerData`** with `user.identities.<provider>.<value>`

   If you didn't use any data from the `providerData` object, you can skip this step.

   Replace `<provider>` with the provider name (for example `username`, `email`, `google`, `github`, etc.) and `<value>` with the field you want to access (for example `isEmailVerified`).

   <Tabs>
   <TabItem value="before" label="Before">

   ```tsx title="src/MainPage.tsx"
   import { findUserIdentity, AuthUser } from 'wasp/auth'

   function getProviderData(user: AuthUser) {
     const emailIdentity = findUserIdentity(user, 'email')
     // We needed this before check for proper type support
     return emailIdentity && 'isEmailVerified' in emailIdentity.providerData
       ? emailIdentity.providerData
       : null
   }

   const MainPage = ({ user }: { user: AuthUser }) => {
     const providerData = getProviderData(user)
     const isEmailVerified = providerData ? providerData.isEmailVerified : null
     // ...
   }
   ```

   </TabItem>
   <TabItem value="after" label="After">

   ```tsx title="src/MainPage.tsx"
   import { AuthUser } from 'wasp/auth'

   const MainPage = ({ user }: { user: AuthUser }) => {
     // The email object is properly typed, so we can access `isEmailVerified` directly
     const isEmailVerified = user.identities.email?.isEmailVerified
     // ...
   }
   ```

   </TabItem>
   </Tabs>

1. **Use `getFirstProviderUserId` directly** on the user object

   If you didn't use `getFirstProviderUserId` in your code, you can skip this step.

   You should replace `getFirstProviderUserId(user)` with `user.getFirstProviderUserId()`.

   <Tabs>
   <TabItem value="before" label="Before">

   ```tsx title="src/MainPage.tsx"
   import { getFirstProviderUserId, AuthUser } from 'wasp/auth'

   const MainPage = ({ user }: { user: AuthUser }) => {
     const userId = getFirstProviderUserId(user)
     // ...
   }
   ```

   ```ts title=src/tasks.ts
   import { getFirstProviderUserId } from 'wasp/auth'

   export const createTask: CreateTask<...>  = async (args, context) => {
       const userId = getFirstProviderUserId(context.user)
       // ...
   }
   ```

   </TabItem>
   <TabItem value="after" label="After">

   ```tsx title="src/MainPage.tsx"
   import { AuthUser } from 'wasp/auth'

   const MainPage = ({ user }: { user: AuthUser }) => {
     const userId = user.getFirstProviderUserId()
     // ...
   }
   ```

   ```ts title=src/tasks.ts
   export const createTask: CreateTask<...>  = async (args, context) => {
       const userId = user.getFirstProviderUserId()
       // ...
   }
   ```

   </TabItem>
   </Tabs>

1. **Replace `findUserIdentity`** with checks on `user.identities.<provider>`

   If you didn't use `findUserIdentity` in your code, you can skip this step.

   Instead of using `findUserIdentity` to get the identity object, you can directly check if the identity exists on the `identities` object.

   <Tabs>
   <TabItem value="before" label="Before">

   ```tsx title="src/MainPage.tsx"
   import { findUserIdentity, AuthUser } from 'wasp/auth'

   const MainPage = ({ user }: { user: AuthUser }) => {
     const usernameIdentity = findUserIdentity(user, 'username')
     if (usernameIdentity) {
       // ...
     }
   }
   ```

   ```ts title=src/tasks.ts
   import { findUserIdentity } from 'wasp/auth'

   export const createTask: CreateTask<...>  = async (args, context) => {
       const usernameIdentity = findUserIdentity(context.user, 'username')
       if (usernameIdentity) {
           // ...
       }
   }
   ```

   </TabItem>
   <TabItem value="after" label="After">

   ```tsx title="src/MainPage.tsx"
   import { AuthUser } from 'wasp/auth'

   const MainPage = ({ user }: { user: AuthUser }) => {
     if (user.identities.username) {
       // ...
     }
   }
   ```

   ```ts title=src/tasks.ts
   export const createTask: CreateTask<...>  = async (args, context) => {
       if (context.user.identities.username) {
           // ...
       }
   }
   ```

   </TabItem>
   </Tabs>

That's it!

You should now be able to run your app with the new Wasp 0.14.0. We recommend reading through the updated [Accessing User Data](./auth/entities/entities.md) section to get a better understanding of the new API.
