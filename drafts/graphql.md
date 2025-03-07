React Router v7に、GraphQLサーバーを実装する方法を解説します。

## 概要

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

## 1️⃣ GraphQL Yogaセットアップ

### 1.1 パッケージをインストール

```sh
npm i graphql graphql-yoga
```

### 1.2 GraphQLサーバーのエンドポイント作成

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

## 2️⃣ POTHOS GraphQLセットアップ

GraphQLスキーマビルダーである、POTHOSをセットアップしていきます。

### サンプルのモデル作成

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

### パッケージのインストール

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

### prisma.schemaにprisma-pothos-typesの設定を追加

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

### スキーマビルダーを作成

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

### Planetを実装

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

### StarClusterを実装

#### Typeを作成

```ts: app/.server/lib/graphql/modules/star-cluster/star-cluster.type.ts
import { builder } from '../../builder';

export const defineStarClusterType = () => {
  builder.prismaNode('StarCluster', {
    id: { field: 'id' },
    findUnique: (id) => ({ id }),
    fields: (t) => ({
      name: t.exposeString('name'),
      planets: t.relation('planets'),
    }),
  });
};

```

#### Dtoを作成

```ts: app/.server/lib/graphql/modules/star-cluster/dto/args/get-star-cluster-args.dto.ts
import { builder } from '~/.server/lib/graphql/builder';

export const GetStarClusterArgs = builder.inputType('GetStarClusterArgs', {
  fields: (t) => ({
    id: t.string({ required: true }),
  }),
});
```

```ts: app/.server/lib/graphql/modules/star-cluster/dto/input/create-star-cluster-input.dto.ts
import { builder } from '~/.server/lib/graphql/builder';

export const CreateStarClusterInput = builder.inputType(
  'CreateStarClusterInput',
  {
    fields: (t) => ({
      name: t.string({ required: true }),
    }),
  },
);

```

```ts: app/.server/lib/graphql/modules/star-cluster/dto/input/update-star-cluster-input.dto.ts
import { builder } from '~/.server/lib/graphql/builder';

export const UpdateStarClusterInput = builder.inputType(
  'UpdateStarClusterInput',
  {
    fields: (t) => ({
      id: t.string({ required: true }),
      name: t.string({ required: true }),
    }),
  },
);

```

```ts: app/.server/lib/graphql/modules/star-cluster/dto/input/delete-star-cluster-input.dto.ts
import { builder } from '~/.server/lib/graphql/builder';

export const DeleteStarClusterInput = builder.inputType(
  'DeleteStarClusterInput',
  {
    fields: (t) => ({
      id: t.string({ required: true }),
    }),
  },
);

```

#### Resolverを作成

```ts: app/.server/lib/graphql/modules/star-cluster/star-cluster.resolver.ts
import { decodeGlobalID } from '@pothos/plugin-relay';
import { prisma } from '~/.server/lib/prisma-client';
import { builder } from '../../builder';
import { GetStarClusterArgs } from './dto/args/get-star-cluster-args.dto';
import { CreateStarClusterInput } from './dto/input/create-star-cluster-input.dto';
import { DeleteStarClusterInput } from './dto/input/delete-star-cluster-input.dto';
import { UpdateStarClusterInput } from './dto/input/update-star-cluster-input.dto';

export const defineStarClusterResolver = () => {
  // --- Query
  builder.queryField('starClusters', (t) =>
    t.prismaConnection({
      type: 'StarCluster',
      cursor: 'id',
      resolve: (query) => prisma.starCluster.findMany({ ...query }),
      totalCount: () => prisma.starCluster.count(),
    }),
  );

  builder.queryField('starCluster', (t) =>
    t.prismaField({
      type: 'StarCluster',
      args: { args: t.arg({ type: GetStarClusterArgs, required: true }) },
      resolve: async (query, _parent, { args }) => {
        const { id: rawId } = decodeGlobalID(args.id);
        return await prisma.starCluster.findUnique({
          ...query,
          where: { id: rawId },
        });
      },
    }),
  );

  // --- Mutation
  builder.mutationField('createStarCluster', (t) =>
    t.prismaField({
      type: 'StarCluster',
      nullable: false,
      args: { input: t.arg({ type: CreateStarClusterInput, required: true }) },
      resolve: async (query, _parent, { input }, _ctx) => {
        const createdStarCluster = await prisma.starCluster.create({
          ...query,
          data: {
            name: input.name,
          },
        });

        return createdStarCluster;
      },
    }),
  );

  builder.mutationField('updateStarCluster', (t) =>
    t.prismaField({
      type: 'StarCluster',
      nullable: true,
      args: { input: t.arg({ type: UpdateStarClusterInput, required: true }) },
      resolve: async (query, _parent, { input }, _ctx) => {
        const { id: rawId } = decodeGlobalID(input.id);
        const updatedStarCluster = await prisma.starCluster.update({
          ...query,
          where: {
            id: rawId,
          },
          data: {
            name: input.name,
          },
        });

        return updatedStarCluster;
      },
    }),
  );

  builder.mutationField('deleteStarCluster', (t) =>
    t.prismaField({
      type: 'StarCluster',
      nullable: true,
      args: { input: t.arg({ type: DeleteStarClusterInput, required: true }) },
      resolve: async (query, _parent, { input }) => {
        const { id: rawId } = decodeGlobalID(input.id);
        return await prisma.starCluster.delete({
          ...query,
          where: { id: rawId },
        });
      },
    }),
  );
};
```

#### Moduleを作成

```ts: app/.server/lib/graphql/modules/star-cluster/star-cluster.module.ts
import { defineStarClusterResolver } from './star-cluster.resolver';
import { defineStarClusterType } from './star-cluster.type';

export const setupStarClusterModule = () => {
  defineStarClusterType();
  defineStarClusterResolver();
  console.log('StarCluster module has been defined');
};

```

#### schema.tsに反映

```diff ts: app/.server/lib/graphql/schema.ts
import { builder } from './builder';
import { setupPlanetModule } from './modules/planet/planet.module';
+ import { setupStarClusterModule } from './modules/star-cluster/star-cluster.module';

setupPlanetModule();
+ setupStarClusterModule();

export const schema = builder.toSchema();

```

### PlanetStarCluster(中間テーブル)を実装

#### Typeを作成

```ts: app/.server/lib/graphql/modules/planet-star-cluster/planet-star-cluster.type.ts
import { builder } from '../../builder';

export const definePlanetStarClusterType = () => {
  builder.prismaNode('PlanetStarCluster', {
    id: { field: 'planetId_starClusterId' },
    fields: (t) => ({
      planet: t.relation('planet'),
      starCluster: t.relation('starCluster'),
    }),
  });
};

```

#### Moduleを作成

```ts: app/.server/lib/graphql/modules/planet-star-cluster/planet-star-cluster.module.ts
import { definePlanetStarClusterType } from './planet-star-cluster.type';

export const setupPlanetStarClusterModule = () => {
  definePlanetStarClusterType();
  console.log('PlanetStarCluster module has been defined');
};

```

#### schema.tsに反映

```diff ts: app/.server/lib/graphql/schema.ts
import { builder } from './builder';
import { setupPlanetStarClusterModule } from './modules/planet-star-cluster/planet-star-cluster.module';
import { setupPlanetModule } from './modules/planet/planet.module';
+ import { setupStarClusterModule } from './modules/star-cluster/star-cluster.module';

setupPlanetModule();
setupStarClusterModule();
+ setupPlanetStarClusterModule();

export const schema = builder.toSchema();

```

### GraphQL Yogaを変更

:::message
ここで利用する`context`のgetSessionは、下記の記事で定義した処理を利用していますので、参考に実装してみてください。

<https://zenn.dev/atman/articles/5cd93410772d03>
:::

- 上記のschemaを反映
- contextを実装

```ts: app/routes/api.graphql/route.ts
+ import { createYoga } from 'graphql-yoga';
+ import { schema } from '~/.server/lib/graphql/schema';
+ import { getSession } from '~/sessions.server';
import type { Route } from './+types/route';

// NOTE: createYogaで生成したインスタンスはシングルトンとして利用される。
const yoga = createYoga({
  schema,
  graphqlEndpoint: '/api/graphql', // GraphQL のエンドポイントを指定
+   context: async (ctx) => {
+     // ログイン中のユーザー取得
+     const session = await getSession(ctx.request.headers.get('Cookie'));
+     const user = session.get('user');
+     // console.log(user);
+     return { ...ctx, user };
  },
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

### 動作確認

`npm run dev`でサーバーを起動後、`http://localhost:5173/api/graphql`にアクセスする。

```graphql
query getPlanets {
  planets {
    edges {
      node {
        id
        name
      }
    }
    totalCount
  }
}

query getPlanet {
  planet(args: { id: "UGxhbmV0OjExNjlhYTc5LTNlNDctNDk2ZS1hNWJhLWM1YzgzY2U5OThmMg==" }) {
    id
    name
  }
}

mutation createPlanet {
  createPlanet(input: {name: "earth"}) {
    id
  }
}

mutation updatePlanet {
  updatePlanet(input: { id: "UGxhbmV0OmVlZmY4YzE0LWYxM2ItNDk5ZC1iNjNiLTM5ZGFjOTRmY2RhNw==", name: "New Earth" }) {
    id
    name
  }
}

mutation deletePlanet {
  deletePlanet(input: { id: "UGxhbmV0OmVlZmY4YzE0LWYxM2ItNDk5ZC1iNjNiLTM5ZGFjOTRmY2RhNw==" }) {
    id
    name
  }
}
```

```graphql
query getStarClusters {
  starClusters {
    edges {
      node {
        id
        name
      }
    }
    totalCount
  }
}

query getStarCluster {
  starCluster(args: { id: "U3RhckNsdXN0ZXI6MmU2YWQ2MWMtZWI1ZC00NjQ1LWE2YjAtN2QyM2ZjNThmMjFk" }) {
    id
    name
  }
}

mutation createStarCluster {
  createStarCluster(input: { name: "Milky Way Cluster" }) {
    id
    name
  }
}

mutation updateStarCluster {
  updateStarCluster(input: { id: "U3RhckNsdXN0ZXI6MmU2YWQ2MWMtZWI1ZC00NjQ1LWE2YjAtN2QyM2ZjNThmMjFk", name: "Andromeda Cluster" }) {
    id
    name
  }
}

mutation deleteStarCluster {
  deleteStarCluster(input: { id: "U3RhckNsdXN0ZXI6MmU2YWQ2MWMtZWI1ZC00NjQ1LWE2YjAtN2QyM2ZjNThmMjFk" }) {
    id
    name
  }
}

```

### 補足. 認証によるアクセス制限の方法

認証状況に応じてエラーを発生させる方法はいくつかあります。

#### 方法1: Type定義のfieldsに制限を設定する

下記のように、loggeInを設定しているので、

```ts: app/.server/lib/graphql/builder.ts
export const builder = new SchemaBuilder<{
  // ...
}>({
  plugins: [
    // ...
  ],
  scopeAuth: {
    authorizeOnSubscribe: true,
    authScopes: async (ctx) => ({
      loggedIn: !!ctx.user,
    }),
  },
```

このように、Typeに指定することができる。

```diff ts: app/.server/lib/graphql/modules/star-cluster/star-cluster.type.ts
import { builder } from '../../builder';

export const defineStarClusterType = () => {
  builder.prismaNode('StarCluster', {
    id: { field: 'id' },
    findUnique: (id) => ({ id }),
    fields: (t) => ({
+     name: t.exposeString('name', {authScopes: {loggedIn: true}}),
      planets: t.relation('planets'),
    }),
  });
};

```

この状態で、ログアウトした状態で、createStarClusterを試しに実行すると、

```graphql
mutation createStarCluster {
  createStarCluster(input: { name: "Milky Way Cluster" }) {
    id
    name
  }
}
```

エラーが返ってきます。

```json
{
  "errors": [
    {
      "message": "Unexpected error.",
      "path": [
        "createStarCluster",
        "name"
      ],
      "extensions": {
        "code": "INTERNAL_SERVER_ERROR",
        "originalError": {
          "message": "Not authorized to resolve StarCluster.name",...
```

#### 方法2: Resolverのctxを使って制限する

例えば、`createStarCluster`のmutationに、ログインしていなければエラーを発生させます。

```diff ts: app/.server/lib/graphql/modules/star-cluster/star-cluster.resolver.ts
 builder.mutationField('createStarCluster', (t) =>
    t.prismaField({
      type: 'StarCluster',
      nullable: false,
      args: { input: t.arg({ type: CreateStarClusterInput, required: true }) },
-     resolve: async (query, _parent, { input }, _ctx) => {
+     resolve: async (query, _parent, { input }, ctx) => {
+       if (!ctx.user) {
+         throw new Error('Unauthorized');
+       }

        const createdStarCluster = await prisma.starCluster.create({
```

この状態で、ログアウトした状態で、createStarClusterを試しに実行すると、

```graphql
mutation createStarCluster {
  createStarCluster(input: { name: "Milky Way Cluster" }) {
    id
    name
  }
}
```

エラーが返ってきます。

```json
{
  "errors": [
    {
      "message": "Unexpected error.",
      "path": [
        "createStarCluster"
      ],
      "extensions": {
        "code": "INTERNAL_SERVER_ERROR",
        "originalError": {
          "message": "Unauthorized",
```

## 3️⃣ GraphQL codegen セットアップ

### GraphQL スキーマファイル出力機能を追加

- GraphQLスキーマ出力処理を追加する。

```ts: tools/graphql-codegen/export-schema.ts
import fs from 'node:fs';
import path from 'node:path';
import { lexicographicSortSchema, printSchema } from 'graphql';
import { schema } from '~/.server/lib/graphql/schema';

const main = async () => {
  const outputFile = './tools/graphql-codegen/schema.graphql';

  const schemaAsString = printSchema(lexicographicSortSchema(schema));
  await fs.writeFileSync(outputFile, schemaAsString);
  console.log(`🌙${path.resolve(outputFile)} is created!`);
};

main();
```

- package.json にスクリプトを追加する。

```json: package.json
  "scripts": {
    // ...
    "--- GRAPHQL SECTION ---": "--- --- --- --- ---",
    "graphql:schema": "npx tsx ./tools/graphql-codegen/export-schema.ts",
```

これを実行すると、`schema.graphql`というファイルが生成されます。
graphql-codegenでは、このスキーマを読み込んで、ファイルを自動生成するようにします。

### graphql-codegenインストール

```sh
npm i graphql
npm i -D typescript @graphql-codegen/cli @graphql-codegen/client-preset
npm i -D @parcel/watcher
```

`@parcel/watcher`はwatchモードを利用する際に必要となる。

### codegen configファイルを作成

```ts: tools/graphql-codegen/codegen.ts
import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  schema: 'tools/graphql-codegen/schema.graphql',
  overwrite: true,
  documents: ['app/**/*.tsx', 'app/**/*.ts'],
  ignoreNoDocuments: true, // for better experience with the watcher
  hooks: { afterAllFileWrite: ['biome check --write'] },
  generates: {
    './app/.server/lib/graphql/@generated/': {
      preset: 'client',
      plugins: [],
      config: {
        scalars: { DateTime: 'string' },
      },
    },
  },
};

export default config;
```

自動生成されたファイルをそのまま利用するとエラーになるので、`hooks: { afterAllFileWrite: ['biome check --write'] }`でフォーマットするようにしています。

### package.jsonにスクリプトを追加

```json: package.json
  "scripts": {
    // ...
    "graphql:codegen": "graphql-codegen -c tools/graphql-codegen/codegen.ts",
    "graphql:codegen-watch": "graphql-codegen -c tools/graphql-codegen/codegen.ts --watch",
```

このスクリプトを実行すると、GraphQLの型生成に必要なファイルが自動生成されます。
![image](https://storage.googleapis.com/zenn-user-upload/d9784a29eec2-20250305.png)

:::details 補足. 自動生成ファイルのLintエラー無視

自動生成ファイルに対して、Lintエラーは無視するように設定しておきます。
ここでは、Biomeの場合の設定を参考に記載しておきます。

```diff biome.json
{
  "$schema": "./node_modules/@biomejs/biome/configuration_schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true
  },
  "files": {
    "include": ["**/*.ts", "**/*.tsx", "**/*.js"],
    "ignore": ["**/.react-router", "**/.vscode", "**/node_modules"]
  },
  "formatter": {
    "indentStyle": "space",
    "ignore": []
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "nursery": {
        "useSortedClasses": "warn"
      },
      "correctness": {
        "noUnusedImports": "warn",
        "noUnusedVariables": "warn"
      }
    }
  },
  "organizeImports": {
    "enabled": true
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "single",
      "semicolons": "always"
    }
  },
  "overrides": [
    {
      "include": ["app/components/shadcn/ui/**"],
      "linter": {
        "rules": {
          "nursery": {
            "useSortedClasses": "off"
          }
        }
      }
    },
+   {
+     "include": ["app/.server/lib/graphql/@generated/**"],
+     "linter": {
+       "rules": {
+         "suspicious": {
+           "noExplicitAny": "off"
+         }
+       }
+     }
+   }
  ]
}
```

自動生成されたファイルでは`any`を利用していますので、警告を無視するように設定しておきます。

:::

## 4️⃣ graphql-request セットアップ

### パッケージをインストール

```sh
npm i graphql-request
```

### GraphQL API URLを環境変数に追加

```sh: .env
# API GraphQL URL
API_GQL_URL='http://localhost:5173/api/graphql'
```

```ts: app/config/env.ts
export const env = {
  // ...
+ API_GQL_URL: process.env.API_GQL_URL as string,
};
```

### GraphQL Clientを作成

```ts: app/lib/graphql-client/index.ts
import { ClientError, GraphQLClient } from 'graphql-request';
import { env } from '~/config/env';

/**
 * オプションのリクエストヘッダーを使用してGraphQLクライアントを初期化します。
 *
 * @param {Request | undefined} [request] - ヘッダーを抽出するためのオプションのリクエストオブジェクト。
 * @returns {Promise<GraphQLClient>} 初期化されたGraphQLクライアントを解決するPromise。
 * @example
 * ```typescript
 * const client = await initializeClient(request);
 * ```
 */
export const initializeClient = async (
  request: Request | undefined = undefined,
) => {
  const headers: Record<string, string> = {};

  if (request) {
    try {
      // 特定の重要なヘッダーのみを選択的に追加
      const selectHeaders = [
        // NOTE: cookieはGraphQLサーバーでユーザー情報取得に必要
        'cookie',
      ];

      for (const headerName of selectHeaders) {
        const headerValue = request.headers.get(headerName);
        if (headerValue) {
          headers[headerName] = headerValue;
        }
      }
    } catch (error) {
      console.error('An error occurred while retrieving headers:', error);
    }
  }

  // GraphQLClientの初期化
  const client = new GraphQLClient(env.API_GQL_URL, {
    fetch: fetch,
    headers: {
      // NOTE: Content-Typeを指定しないとGqphQLの構文エラーとなる。
      'Content-Type': 'application/json',
      ...headers,
    },
  });

  return client;
};

/**
 * `ClientError`インスタンスから元のエラーメッセージを抽出します。
 *
 * @param error - 元のエラーメッセージを抽出する`ClientError`インスタンス。
 * @returns 元のエラーメッセージが存在する場合はそれを返し、存在しない場合は`null`を返します。
 */
export const getOriginalErrorMessage = (error: ClientError): string | null => {
  if (error instanceof ClientError) {
    const originalError = error.response?.errors?.[0]?.extensions
      ?.originalError as { message: string } | undefined;

    if (originalError?.message) {
      return originalError.message;
    }
  }
  return null;
};

```

## 5 動作確認

### データ取得用のGraphQLを作成

```tsx: app/routes/__.demo.graphql._index/route.tsx
const getPlanets = graphql(`
query getPlanets {
  planets {
    edges {
      node {
        id
        name
      }
    }
    totalCount
  }
}
`);
```

### graphql-codegenを実行

```sh
npm run graphql:codegen
```

### ページを作成

```tsx: app/routes/__.demo.graphql._index/route.tsx
import { graphql } from '~/.server/lib/graphql/@generated';
import {
  Table,
  TableBody,
  TableCaption,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '~/components/shadcn/ui/table';
import {
  getOriginalErrorMessage,
  initializeClient,
} from '~/lib/graphql-client';
import type { Route } from './+types/route';

const getPlanets = graphql(`
query getPlanets {
  planets {
    edges {
      node {
        id
        name
      }
    }
    totalCount
  }
}
`);

export const loader = async ({ request }: Route.LoaderArgs) => {
  const client = await initializeClient(request);
  return await client
    .request(getPlanets)
    .then((planets) => planets)
    .catch((error) => {
      const errorMessage = getOriginalErrorMessage(error);
      return { errorMessage: errorMessage ?? (error.message as string) };
    });
};

const DemoGraphqlPage = ({ loaderData }: Route.ComponentProps) => {
  if ('errorMessage' in loaderData) {
    return <div>{loaderData.errorMessage}</div>;
  }

  const planets = loaderData.planets?.edges;

  return (
    <>
      <Table>
        <TableCaption>A list of planets</TableCaption>
        <TableHeader>
          <TableRow>
            <TableHead>ID</TableHead>
            <TableHead>Name</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          {planets?.map((planet) => (
            <TableRow key={planet?.node?.id}>
              <TableCell>{planet?.node?.id}</TableCell>
              <TableCell>{planet?.node?.name}</TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </>
  );
};

export default DemoGraphqlPage;
```

完成画面
![image](https://storage.googleapis.com/zenn-user-upload/4250017e8dd2-20250306.png)

## 補足

### DBテーブル更新時の改修手順の流れ

1. `schema.prisma`を更新
2. prisma migrate を実行
3. prisma generate を実行
4. GraphQL Pothosのスキーマを追加
   - タイプを追加
   - DTOを追加
   - リゾルバ（Query, Mutation）を追加
   - モジュール（Type、Resolver定義処理実行）を追加
   - `schema.ts`にモジュール呼び出しを追加
5. `npm run graphql:schema`で、スキーマファイルをエクスポート
6. tsファイルに、GraphQLを記載
7. `graphql:codegen`で、型生成を自動生成

### GraphQLフォルダ構造

今回のGraphQLフォルダ構造は、NestJSのモジュール設計を参考にしています。

#### モジュール管理のポイント

✅ **エンティティごとに分離** → 拡張性が高く、新規追加が容易  
✅ **DTO, モデル, リゾルバを整理** → 規模が大きくなってもメンテしやすい  
✅ **モジュールのエントリーポイントを統一** → 各エンティティのスキーマ定義を明確にし、依存関係を整理しやすくする

#### フォルダ構造

```text
app/
  ├── .server/
  │   ├── lib/
  │   │   ├── graphql/
  │   │   │   ├── builder.ts
  │   │   │   ├── context.ts
  │   │   │   ├── modules/
  │   │   │   │   ├── post/
  │   │   │   │   │   ├── dto/
  │   │   │   │   │   │   ├── args/
  │   │   │   │   │   │   │   ├── get-post-args.dto.ts
  │   │   │   │   │   │   ├── input/
  │   │   │   │   │   │   │   ├── create-post-input.dto.ts
  │   │   │   │   │   │   │   ├── update-post-input.dto.ts
  │   │   │   │   │   │   │   ├── delete-post-input.dto.ts
  │   │   │   │   │   ├── post.type.ts
  │   │   │   │   │   ├── post.resolver.ts
  │   │   │   │   │   ├── post.module.ts
  │   │   │   ├── schema.ts
```
