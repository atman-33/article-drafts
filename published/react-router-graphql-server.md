# React Router v7 | GraphQLサーバー実装方法

本記事では、React Router v7とGraphQLを組み合わせて、Webアプリケーションを構築する方法を解説します。具体的には、GraphQL Yogaを用いたサーバーのセットアップから、Pothos GraphQLを使った型安全なスキーマの構築、さらにGraphQL Codegenを利用した型定義の自動生成まで、ステップバイステップで説明していきます。

👇ソースコードはこちら
@[card](https://github.com/atman-33/react-router-v7-boilerplate/tree/keep/graphql)

## 1️⃣ 概要

### 利用するライブラリの関係性と役割

| ライブラリ               | 役割・特徴 | どこで使うか |
|---------------------|----------|------------|
| **GraphQL Yoga**   | GraphQL サーバーの実装。Fastify, Express などと統合可能 | **サーバー** |
| **Pothos GraphQL** | TypeScript で型安全な GraphQL スキーマを構築できるスキーマビルダー | **サーバー** |
| **GraphQL Codegen** | GraphQL スキーマから型定義やクライアントコードを自動生成 | **クライアント & サーバー** |
| **graphql-request** | 軽量な GraphQL クライアント。シンプルなリクエストの送信に特化 | **クライアント** |

![image](https://storage.googleapis.com/zenn-user-upload/7a6ecbdde33c-20250307.png)

以下、ライブラリの関係性について補足となります。

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

まずは必要なパッケージをインストールします。

```sh
npm i graphql graphql-yoga
```

### 2.2 GraphQLサーバーのエンドポイント作成

次に、GraphQLサーバーのエンドポイントを作成します。

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

上記まで作成したら、 Yoga GraphiQL にアクセスしてみます。

```sh
npm run dev
```

ブラウザから`localhost:xxxx/api/graphql`のエンドポイントを開き、以下のようなGraphiQL画面が表示されれば成功です。

![image](https://storage.googleapis.com/zenn-user-upload/839312c715e9-20250225.png)

## 3️⃣ POTHOS GraphQLセットアップ

次は、GraphQLスキーマビルダーである、POTHOSをセットアップしていきます。

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

今回のサンプルでは、Planet（惑星）とStarCluster（星団）が多対多の関係になっているため、中間テーブルも用意しています。
サンプルはやや複雑な内容になっていますので、Planetだけでシンプルに実装してもよいです。
その場合は、以降のコードもPlanetのみの実装に合わせて変更する必要がある点にご注意ください。

### 3.2 パッケージのインストール

必要なパッケージをインストールします。

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

prisma-pothos-types の設定を追加します。

```prisma: prisma/schema.prisma
generator pothos {
  provider     = "prisma-pothos-types"
}
```

追加した設定を反映するために、`prisma generate`を実行しておきます。

```sh
npx prisma generate
```

上記を実行するとPrismaクライアントが作成され、コード内でデータベースにアクセス可能となります。

### 3.4 スキーマビルダーを作成

コンテキストを準備します。このコンテキストには、認証後のユーザー情報が格納されます。

```ts: app/.server/lib/graphql/context.ts
import type { User } from '@prisma/client';
import type { YogaInitialContext } from 'graphql-yoga';

export interface Context extends YogaInitialContext {
  user?: User;
}
```

スキーマビルダーを準備します。スキーマビルダーは、GraphQLスキーマを構築するためのライブラリであり、後で説明するクエリ、ミューテーションの定義にも利用します。

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

ここからPlanetのGraphQLスキーマを定義していきます。

#### Typeを作成

まずはPlanetのTypeを作成します。

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

次にクエリやミューテーションで利用するDtoを定義します。今回はargs（Read系）とinput（Write系）に分けて作成しています。

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

次に、PlanetのResolverを作成します。ここでクエリとミューテーションを定義します。

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

PlanetのModuleを作成します。Moduleでは、上記で作成したTypeとResolver定義を呼び出す役割としています。

```ts: app/.server/lib/graphql/modules/planet/planet.module.ts
import { definePlanetResolver } from './planet.resolver';
import { definePlanetType } from './planet.type';

export const setupPlanetModule = () => {
  definePlanetType();
  definePlanetResolver();
  console.log('Planet module has been defined');
};
```

:::message
ModuleはNestJSの構造を参考にしていますが、必須ではありません。重要なのは、builderからTypeとResolver（クエリやミューテーション）を定義できることです。
:::

#### schema.tsに反映

以下のように、Moduleで定義した呼び出し関数を実行することで、schema.tsにTypeとResolverを反映させます。

```ts: app/.server/lib/graphql/schema.ts
import { builder } from './builder';
import { setupPlanetModule } from './modules/planet/planet.module';

setupPlanetModule();

export const schema = builder.toSchema();
```

### 3.6 StarClusterを実装

続いてStarClusterも実装していきます。繰り返しとなるため、説明は省略します。

:::details コード

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

:::

### 3.7 PlanetStarCluster(中間テーブル)を実装

こちらも繰り返しとなるため、説明は省略します。

:::details コード

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

:::

### 3.8 GraphQL Yogaを変更

今までの説明で作成した`schema`を、`GraphQL Yoga`に適用します。  
ここでは、ユーザー認証を利用するために、`CookieSessionStorage`からユーザー情報を取得しています。

:::message
ここで利用する`context`のgetSessionは、下記の記事で定義した処理を利用していますので、参考に実装してみてください。

@[card](https://zenn.dev/atman/articles/5cd93410772d03)

ユーザー認証も含めて実装すると大変になるため、もし認証を省略する場合は、ContextからUserを除くか、以降で扱う`user`を空で返すなどして対応してください。
:::

```diff ts: app/routes/api.graphql/route.ts
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

### 3.9 動作確認

1. ターミナルで以下のコマンドを実行し、サーバーを起動します。

    ```sh
    npm run dev
    ```

2. ブラウザで以下のURLにアクセスし、GraphQLのエンドポイントが正しく動作しているか確認します。

    ```sh
    http://localhost:5173/api/graphql
    ```

3. 次のクエリやミューテーションを実行して、データの取得や更新ができるか確認してください。

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
      planet(args: { id: "xxx" }) {
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
      updatePlanet(input: { id: "xxx", name: "New Earth" }) {
        id
        name
      }
    }
    
    mutation deletePlanet {
      deletePlanet(input: { id: "xxx" }) {
        id
        name
      }
    }
    ```

上記の`{ id: "xxx" }`の`xxx`は参考です。実際は、生成されたレコードのIDを指定してください。
GraphiQLの扱い方は省略しますが、正常にクエリが実行できれば以下のように結果が表示されます。

![image](https://storage.googleapis.com/zenn-user-upload/c58266d66136-20250309.png)

### 補足. 認証によるアクセス制限の方法

認証状況に応じてエラーを発生させる方法はいくつかあります。

#### 方法1: Type定義のfieldsに制限を設定する

今回は、下記のように`loggedIn`というスコープを設定していますので、

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

以下のように、Typeにスコープを設定することができます（例では`name`に`loggedIn`を設定）。

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

この状態で、ログアウトしたまま`createStarCluster`を試しに実行すると、

```graphql
mutation createStarCluster {
  createStarCluster(input: { name: "Milky Way Cluster" }) {
    id
    name
  }
}
```

エラーが返ってきます。これは、ログインしていない状態で`createStarCluster`を実行したため、`StarCluster.name`の取得が認証エラーとしてブロックされたためです。

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

他には、Resolverのクエリ・ミューテーション定義内でアクセス制限を実装する方法もあります。
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

この状態で、ログアウトしたままcreateStarClusterを試しに実行すると、

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

## 4️⃣ GraphQL codegen セットアップ

ここでは、GraphQL codegenを利用するための設定を追加していきます。

### 4.1 GraphQL スキーマファイル出力機能を追加

まずはGraphQLスキーマ出力処理を追加します。

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

package.json にスクリプトを追加します。

```json: package.json
  "scripts": {
    // ...
    "--- GRAPHQL SECTION ---": "--- --- --- --- ---",
    "graphql:schema": "npx tsx ./tools/graphql-codegen/export-schema.ts",
    // ...
  }
```

これを実行すると、`schema.graphql`というファイルが生成されます。
graphql-codegenでは、このスキーマを読み込んで、ファイルを自動生成するようにします。

### 4.2 graphql-codegenインストール

以下のコマンドを実行して、必要なパッケージをインストールします。

```sh
npm i graphql
npm i -D typescript @graphql-codegen/cli @graphql-codegen/client-preset
npm i -D @parcel/watcher
```

`@parcel/watcher`はwatchモードを利用する際に必要となります（watchモードは後述）。

### 4.3 codegen configファイルを作成

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

:::message
現在はBiomeでのフォーマットを紹介していますが、ESLintでも代用できます。最悪の場合、ファイルが自動生成された後に手動修正でも問題ありません。

もしBiomeを利用する場合は、以下の設定方法を参考にしてください。
@[card](https://zenn.dev/atman/books/b060a0ba47af8c/viewer/3dafd3)
:::

### 4.4 package.jsonにスクリプトを追加

```json: package.json
  "scripts": {
    // ...
    "graphql:codegen": "graphql-codegen -c tools/graphql-codegen/codegen.ts",
    "graphql:codegen-watch": "graphql-codegen -c tools/graphql-codegen/codegen.ts --watch",
    // ...
  }
```

2つスクリプトを追加していますが、`--watch`オプションを付きのスクリプトを実行すると、ファイルが変更されるたびに自動で型生成を再実行するようになります。

どちらのスクリプトでもよいので、実行するとGraphQLの型生成に必要なファイルが自動生成されます。
![image](https://storage.googleapis.com/zenn-user-upload/d9784a29eec2-20250305.png)

:::details 自動生成ファイルのLintエラー無視（biome.json）

自動生成ファイルに対して、Lintエラーは無視するように設定しておきます。ここでは、Biomeの場合の設定を参考に記載しておきます。
自動生成されたファイルでは`any`を利用していますので、警告を無視するように設定しておきました。

```diff json: biome.json
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

:::

## 5️⃣ graphql-request セットアップ

ここからは、クライアント側から GraphQL API にリクエストを送るための実装を進めていきます。

### 5.1 パッケージをインストール

まずは必要なパッケージをインストールします。

```sh
npm i graphql-request
```

### 5.2 GraphQL API URLを環境変数に追加

次に、GraphQLサーバーのAPIエンドポイントを環境変数から取得できるようにしておきます。

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

### 5.3 GraphQL Clientを作成

クライアント側でGraphQLを扱うための処理を作成します。

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

これでクライアント側でGraphQLを扱う準備は完了です！
残りは動作確認となりますので、もう少しです。

## 6️⃣ 動作確認

実際にGraphQLのデータを取得して表示させるデモページを作成してみます。

### 6.1 データ取得用のGraphQLを作成

今回の`graphql-codegen`の設定により、`app`フォルダ内の**ts**や**tsx**ファイルにGraphQL文が含まれている場合、それらが**codegen**の自動生成対象となります。

まずは、`getPlanets`という関数を作成してみましょう。

```tsx: app/routes/__.demo.graphql._index/route.tsx
import { graphql } from '~/.server/lib/graphql/@generated';

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

この時点では、上記の`graphql(...`の部分がエラーとなっているはずです。

### 6.2 graphql-codegenを実行

コード生成のコマンドを実行します。

```sh
npm run graphql:codegen
```

コマンドを実行後、`graphql(...`の参照先にコードが自動生成されるため、エラーがなくなっていれば成功です。

### 6.3 デモページを作成

最後にデモページを作成して完了です！

```tsx: app/routes/__.demo.graphql._index/route.tsx
import { graphql } from '~/.server/lib/graphql/@generated';
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
      <div>
        <h2>A list of planets</h2>
        <ul>
          {planets?.map((planet) => (
            <li key={planet?.node?.id}>
              {planet?.node?.id} 👉 {planet?.node?.name}
            </li>
          ))}
        </ul>
      </div>
    </>
  );
};

export default DemoGraphqlPage;
```

👇完成画面
![image](https://storage.googleapis.com/zenn-user-upload/72c206a75d42-20250309.png)

上記のデモページでは、データ取得のみの機能が実装されています。もしデータの作成や更新機能を追加したい場合は、同じようにGraphQL文を作成し、自動コード生成を行い、画面ページを作成してみてください。

長い記事となってしまいましたが、こちらで実装は完了です！ここまで読んで頂き、ありがとうございました🎉

## 補足

### DBテーブル更新時の改修手順

これまでの作成ステップは、初回のセットアップの場合に該当しますが、実際にはテーブルの追加や変更、スキーマの追加を繰り返すことが多いです。

その場合は、以下のような手順になります。

1. `schema.prisma`を更新 - DBのテーブルやフィールドを変更
2. prisma migrate を実行 - DBスキーマを反映
3. prisma generate を実行 - Prismaクライアントを再生成
4. GraphQL Pothosのスキーマを追加
   - タイプを追加
   - DTOを追加
   - Query/Mutationのリゾルバを追加
   - モジュール（Type、Resolver定義処理実行）を追加
   - `schema.ts`にモジュール呼び出しを追加
5. `npm run graphql:schema`でスキーマファイルをエクスポート
6. tsファイルに、GraphQLのリクエスト文を記載
7. `graphql:codegen`で、型生成を自動生成

### GraphQLフォルダ構造

今回のGraphQLフォルダ構造は、NestJSのモジュール設計を参考にしています。ただし、これはあくまで一つの例であり、正解はありません。
そのため、自分に合ったフォルダ構造を考えて作成して頂ければと思います。

#### フォルダ構造

**ポイント**
✅ **エンティティごとに分離** → 拡張性が高く、新規追加が容易  
✅ **DTO, モデル, リゾルバを整理** → 規模が大きくなってもメンテしやすい  
✅ **モジュールのエントリーポイントを統一** → 各エンティティのスキーマ定義を明確にし、依存関係を整理しやすくする

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
