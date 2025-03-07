# React Router v7 | GraphQLサーバー実装方法

React Router v7に、GraphQLサーバーを実装する方法を解説します。

## 1️⃣ 概要

### 利用するライブラリの関係性と役割

| ツール               | 役割・特徴 | どこで使うか |
|---------------------|----------|------------|
| **GraphQL Yoga**   | GraphQL サーバーの実装。Fastify, Express などと統合可能 | **サーバー** |
| **Pothos GraphQL** | TypeScript で型安全な GraphQL スキーマを構築できるスキーマビルダー | **サーバー** |
| **GraphQL Codegen** | GraphQL スキーマから型定義やクライアントコードを自動生成 | **クライアント & サーバー** |
| **graphql-request** | 軽量な GraphQL クライアント。シンプルなリクエストの送信に特化 | **クライアント** |

![image](https://storage.googleapis.com/zenn-user-upload/7a6ecbdde33c-20250307.png)

### **関係性**

1. **GraphQL YOGA**  
   → GraphQL サーバーを提供する。  
   → **Pothos GraphQL** を利用して型安全なスキーマを構築できる。  

2. **Pothos GraphQL**  
   → TypeScript でスキーマを構築し、GraphQL YOGA で提供する。  
   → `t.relation()` などを使い、Prisma との統合も可能。  

3. **GraphQL Codegen**  
   → GraphQL スキーマ（Pothos や Yoga で定義されたもの）を元に型やクエリ関数を自動生成。  
   → クライアントでは `graphql-request` や `urql` などと連携できる。  

4. **graphql-request**  
   → クライアント側で GraphQL API にリクエストを送るためのライブラリ。  
   → **GraphQL Codegen** を使うと、型安全なリクエストコードを自動生成可能。  

## 2️⃣ GraphQL Yogaセットアップ

### 2.1 パッケージをインストール

```sh
npm i graphql graphql-yoga
```

### 2.2 GraphQLサーバーのエンドポイント作成

```ts: app/routes/api.graphql/route.ts
import { createSchema, createYoga } from 'graphql-yoga';
import type { Route } from './+types/route';

// Define your schema and resolvers
const typeDefs = /* GraphQL */ `
  type Query {
    hello: String
  }
`;

const resolvers = {
  Query: {
    hello: () => 'Hello from Yoga!',
  },
};

const schema = createSchema({
  typeDefs,
  resolvers,
});

const yoga = createYoga({
  schema,
  graphqlEndpoint: '/api/graphql', // GraphQL のエンドポイントを指定
});

export const loader = async ({ request, context }: Route.LoaderArgs) => {
  const response = await yoga.handleRequest(request, context);
  return new Response(response.body, response);
};

export const action = async ({ request, context }: Route.ActionArgs) => {
  const response = await yoga.handleRequest(request, context);
  return new Response(response.body, response);
};

```

Yoga GraphiQL にアクセスしてみる。  

```sh
npm run dev
```

ブラウザから`localhost:xxxx/api/graphql`のエンドポイントを開く。
![image](https://storage.googleapis.com/zenn-user-upload/839312c715e9-20250225.png)

## 3️⃣ POTHOS GraphQLセットアップ

GraphQLスキーマビルダーである、POTHOSをセットアップしていきます。

### 3.1 サンプルのモデル作成

prismaスキーマに各モデルを追加します。

```prisma: prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

// ...

model Planet {
  id        String              @id @default(uuid())
  name      String
  createdAt DateTime            @default(now())
  updatedAt DateTime            @updatedAt
  clusters  PlanetStarCluster[] // 中間モデルを通じたリレーション
}

model StarCluster {
  id        String              @id @default(uuid())
  name      String
  createdAt DateTime            @default(now())
  updatedAt DateTime            @updatedAt
  planets   PlanetStarCluster[] // 中間モデルを通じたリレーション
}

model PlanetStarCluster {
  planet        Planet      @relation(fields: [planetId], references: [id])
  planetId      String
  starCluster   StarCluster @relation(fields: [starClusterId], references: [id])
  starClusterId String
  assignedAt    DateTime    @default(now()) // リレーションが作成された日時

  @@id([planetId, starClusterId]) // 複合主キー
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  password  String
  name      String
  image     String?
  provider  String   @default("Credentials")
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### 3.2 パッケージのインストール

```sh
npm i @pothos/core @pothos/plugin-prisma @pothos/plugin-relay @pothos/plugin-simple-objects @pothos/plugin-scope-auth graphql-scalars
```

:::details 各パッケージの解説

- **@pothos/core**: Pothosは、GraphQLスキーマを構築するためのフレームワークです。型安全で柔軟なスキーマ作成をサポートします。

- **@pothos/plugin-prisma**: Prismaの型を利用して、型安全なGraphQLスキーマを生成します。

- **@pothos/plugin-relay**: Relay規約に基づいたページネーションを簡単に実装できます。

- **@pothos/plugin-simple-objects**: シンプルなオブジェクト型（例えば、tokenだけを持つオブジェクト）を簡単に定義できます。

- **@pothos/plugin-scope-auth**: スコープに基づいた認証をサポートし、GraphQLリクエストに認証を追加できます。

- **graphql-scalars**: `DateTime`型など、標準にはないスカラ型を利用でき、日時データを型安全に扱えます。
:::

### 3.3 prisma.schemaにprisma-pothos-typesの設定を追加

- prisma-pothos-types の設定を追加する。

```prisma: prisma/schema.prisma
generator pothos {
  provider     = "prisma-pothos-types"
}
```

- 追加した設定を反映するために、`prisma generate`を実行しておく。

```sh
npx prisma generate
```

### 3.4 スキーマビルダーを作成

コンテキストを準備する。  

```ts: app/.server/lib/graphql/context.ts
import type { User } from '@prisma/client';
import type { YogaInitialContext } from 'graphql-yoga';

export interface Context extends YogaInitialContext {
  user?: User;
}

```

スキーマビルダーを準備する。

```ts: app/.server/lib/graphql/builder.ts
import SchemaBuilder from '@pothos/core';
import PrismaPlugin from '@pothos/plugin-prisma';
import type PrismaTypes from '@pothos/plugin-prisma/generated';
import RelayPlugin from '@pothos/plugin-relay';
import ScopeAuthPlugin from '@pothos/plugin-scope-auth';
import PothosSimpleObjectsPlugin from '@pothos/plugin-simple-objects';
import { Prisma } from '@prisma/client';
import { DateTimeResolver } from 'graphql-scalars';
import { prisma } from '../prisma-client';
import type { Context } from './context';

export const builder = new SchemaBuilder<{
  Scalars: {
    DateTime: {
      Input: Date;
      Output: Date;
    };
  };
  AuthScopes: {
    loggedIn: boolean;
  };
  Connection: {
    totalCount: number | (() => number | Promise<number>);
  };
  PrismaTypes: PrismaTypes;
  Context: Context;
}>({
  plugins: [
    ScopeAuthPlugin,
    PrismaPlugin,
    RelayPlugin,
    PothosSimpleObjectsPlugin,
  ],
  scopeAuth: {
    authorizeOnSubscribe: true,
    authScopes: async (ctx) => ({
      loggedIn: !!ctx.user,
    }),
  },
  relay: {},
  prisma: {
    client: prisma,
    dmmf: Prisma.dmmf,
  },
});

builder.queryType();
builder.mutationType();

builder.addScalarType('DateTime', DateTimeResolver, {});

```

### 3.5 Planetを実装

#### Typeを作成

```ts: app/.server/lib/graphql/modules/planet/planet.type.ts
import { builder } from '../../builder';

export const definePlanetType = () => {
  builder.prismaNode('Planet', {
    id: { field: 'id' },
    findUnique: (id) => ({ id }),
    fields: (t) => ({
      name: t.exposeString('name'),
      clusters: t.relation('clusters'),
    }),
  });
};

```

#### Dtoを作成

argsとinputに分けて作成します。

```ts: app/.server/lib/graphql/modules/planet/dto/args/get-planet-args.dto.ts
import { builder } from '~/.server/lib/graphql/builder';

export const GetPlanetArgs = builder.inputType('GetPlanetArgs', {
  fields: (t) => ({
    id: t.string({ required: true }),
  }),
});

```

```ts: app/.server/lib/graphql/modules/planet/dto/input/create-planet-input.dto.ts
import { builder } from '~/.server/lib/graphql/builder';

export const CreatePlanetInput = builder.inputType('CreatePlanetInput', {
  fields: (t) => ({
    name: t.string({ required: true }),
    starClusterIds: t.stringList({ required: false }),
  }),
});

```

```ts: app/.server/lib/graphql/modules/planet/dto/input/update-planet-input.dto.ts
import { builder } from '~/.server/lib/graphql/builder';

export const UpdatePlanetInput = builder.inputType('UpdatePlanetInput', {
  fields: (t) => ({
    id: t.string({ required: true }),
    name: t.string({ required: true }),
    starClusterIds: t.stringList({ required: false }),
  }),
});

```

```ts: app/.server/lib/graphql/modules/planet/dto/input/delete-planet-input.dto.ts
import { builder } from '~/.server/lib/graphql/builder';

export const DeletePlanetInput = builder.inputType('DeletePlanetInput', {
  fields: (t) => ({
    id: t.string({ required: true }),
  }),
});

```

#### Resolverを作成

```ts: app/.server/lib/graphql/modules/planet/planet.resolver.ts
import { decodeGlobalID } from '@pothos/plugin-relay';
import { prisma } from '~/.server/lib/prisma-client';
import { builder } from '../../builder';
import { GetPlanetArgs } from './dto/args/get-planet-args.dto';
import { CreatePlanetInput } from './dto/input/create-planet-input.dto';
import { DeletePlanetInput } from './dto/input/delete-planet-input.dto';
import { UpdatePlanetInput } from './dto/input/update-planet-input.dto';

export const definePlanetResolver = () => {
  // --- Query
  builder.queryField('planets', (t) =>
    t.prismaConnection({
      type: 'Planet',
      cursor: 'id',
      resolve: (query) => prisma.planet.findMany({ ...query }),
      totalCount: () => prisma.planet.count(),
    }),
  );

  builder.queryField('planet', (t) =>
    t.prismaField({
      type: 'Planet',
      args: { args: t.arg({ type: GetPlanetArgs, required: true }) },
      resolve: async (query, _parent, { args }) => {
        const { id: rawId } = decodeGlobalID(args.id);
        return await prisma.planet.findUnique({
          ...query,
          where: { id: rawId },
        });
      },
    }),
  );

  // --- Mutation
  builder.mutationField('createPlanet', (t) =>
    t.prismaField({
      type: 'Planet',
      nullable: false,
      args: { input: t.arg({ type: CreatePlanetInput, required: true }) },
      resolve: async (query, _parent, { input }, _ctx) => {
        const createdPlanet = await prisma.planet.create({
          ...query,
          data: {
            name: input.name,
          },
        });

        if (input.starClusterIds && input.starClusterIds.length > 0) {
          const starClusterInputs = input.starClusterIds.map(
            (starClusterId) => ({
              planetId: createdPlanet.id,
              starClusterId,
            }),
          );

          await prisma.planetStarCluster.createMany({
            data: starClusterInputs,
          });
        }

        return createdPlanet;
      },
    }),
  );

  builder.mutationField('updatePlanet', (t) =>
    t.prismaField({
      type: 'Planet',
      nullable: true,
      args: { input: t.arg({ type: UpdatePlanetInput, required: true }) },
      resolve: async (query, _parent, { input }, _ctx) => {
        const { id: rawId } = decodeGlobalID(input.id);
        const updatedPlanet = await prisma.planet.update({
          ...query,
          where: {
            id: rawId,
          },
          data: {
            name: input.name,
          },
        });

        if (input.starClusterIds) {
          await prisma.planetStarCluster.deleteMany({
            where: {
              planetId: rawId,
            },
          });

          if (input.starClusterIds.length > 0) {
            const starClusterInputs = input.starClusterIds.map(
              (starClusterId) => ({
                planetId: updatedPlanet.id,
                starClusterId,
              }),
            );

            await prisma.planetStarCluster.createMany({
              data: starClusterInputs,
            });
          }
        }

        return updatedPlanet;
      },
    }),
  );

  builder.mutationField('deletePlanet', (t) =>
    t.prismaField({
      type: 'Planet',
      nullable: true,
      args: { input: t.arg({ type: DeletePlanetInput, required: true }) },
      resolve: async (query, _, { input }) => {
        const { id: rawId } = decodeGlobalID(input.id);
        return await prisma.planet.delete({
          ...query,
          where: { id: rawId },
        });
      },
    }),
  );
};

```

#### Moduleを作成

```ts: app/.server/lib/graphql/modules/planet/planet.module.ts
import { definePlanetResolver } from './planet.resolver';
import { definePlanetType } from './planet.type';

export const setupPlanetModule = () => {
  definePlanetType();
  definePlanetResolver();
  console.log('Planet module has been defined');
};

```

#### schema.tsに反映

```ts: app/.server/lib/graphql/schema.ts
import { builder } from './builder';
import { setupPlanetModule } from './modules/planet/planet.module';

setupPlanetModule();

export const schema = builder.toSchema();

```
