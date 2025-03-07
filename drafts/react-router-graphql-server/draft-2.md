### 3.6 StarClusterを実装

繰り返しとなるため、説明は省略します。

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

繰り返しとなるため、説明は省略します。

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
