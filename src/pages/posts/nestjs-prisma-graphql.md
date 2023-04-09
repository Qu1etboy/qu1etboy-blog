---
layout: ../../layouts/PostLayout.astro
title: Learn how to used Nest.js with Prisma and GraphQL
author: Weerawong Vonggatunyu
authorImg:
slug: nestjs-prisma-graphql
categories: [nest.js, prisma, GraphQL]
createdAt: 4-9-2023
editedAt: 4-9-2023
thumbnail: https://repository-images.githubusercontent.com/160067449/cdc5a000-b305-11e9-9f4e-e2273bfb33fe
description:
---

This article will teach you how to install and setup Nest.js with Prisma and GraphQL for creating backend server.

## Setup from scratch

### 1. Create a new project

Install nest cli if you haven't

```bash
pnpm i -g @nest/cli
```

then create a new project with this command.

```bash
nest new <project_name>
```

In this tutorial I choose `pnpm` as a package manager but you can choose what ever you want.

### 2. Connect to Database with Docker

If you not have docker desktop installed, download [here](https://www.docker.com/)

after you installed setup `docker-compose.yml` and copy the code below

```yml
version: '3.9'
services:
  postgres:
    image: postgres
    ports:
      - 5432:5432
    volumes:
      - postgres:/var/liv/postgresql/data
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_DB=${POSTGRES_DB}
volumes:
  postgres:
```

I choose Postgresql as our database, but you can choose anything that suit your project.

Create `.env` and config your database password, user, database name (you can write it as anything)

```shell
POSTGRES_USER=
POSTGRES_PASSWORD=
POSTGRES_DB=
```

and then run this command to start a postgres server

```bash
docker-compose up
```

### 3. Prisma

Download Prisma

```bash
pnpm add -D prisma
```

Initialize prisma, this will create `./prisma/schema.prisma` for you.

```bash
npx prisma init
```

In your `.env` change username, password, and database name to what you have config.

```shell
DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@localhost:5432/${POSTGRES_DB}?schema=public
```

In `prisma/schema.prisma` modified the file as followed

```prisma
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Pet {
  id   Int    @id @default(autoincrement())
  name String
}

```

Here, we defined a Pet model with `id` and `name`.

Then we run prisma migrate to create table in our database

```
npx prisma migrate dev --name init
```

Install prisma client and generate prisma schema to prisma client

```bash
pnpm add @prisma/client && npx prisma generate
```

Create a Prisma Service inside our Nest.js application

```ts
// src/prisma/prisma.service.ts
import { Injectable } from '@nestjs/common'
import { PrismaClient } from '@prisma/client'

@Injectable()
export class PrismaService extends PrismaClient {
  constructor() {
    super()
  }
}
```

```ts
// src/prisma/prisma.module.ts
import { Global, Module } from '@nestjs/common'
import { PrismaService } from './prisma.service'

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

Don't forget to import `PrismaModule` to `app.module.ts`

## 4. GraphQL

Now for our main part GraphQL, install required dependencies as followed.

```bash
 pnpm add @nestjs/graphql @nestjs/apollo @apollo/server graphql
```

For this tutorial we will used a schema first approach. If you want to used a code first you can see [the offical doc](https://docs.nestjs.com/graphql/quick-start#code-first) for a guide.

Now define `GraphQLModule` in `app.module.ts`

```ts
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  typePaths: ['./**/*.graphql'],
  // used apollo playground
  playground: false,
  plugins: [ApolloServerPluginLandingPageLocalDefault()],
  // this will auto generate a graphql types
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
    outputAs: 'class',
  },
}),
```

run `pnpm start:dev` to generate a `graphql.ts` which will contain GraphQL types

## 5. Create our Pet resources

You can do that easily by using nest built in CLI.

```
nest generate resource pet
```

Then modified the files as needed

```ts
// pet.graphql
type Pet {
  id: Int!
  name: String!
}

input CreatePetInput {
  name: String!
}

input UpdatePetInput {
  id: Int!
  name: String!
}

type Query {
  pets: [Pet]!
  pet(id: Int!): Pet
}

type Mutation {
  createPet(createPetInput: CreatePetInput!): Pet!
  updatePet(updatePetInput: UpdatePetInput!): Pet!
  removePet(id: Int!): Pet
}
```

```ts
// pet.service.ts
import { Injectable } from '@nestjs/common'
import { CreatePetInput, UpdatePetInput } from 'src/graphql'
import { PrismaService } from 'src/prisma/prisma.service'

@Injectable()
export class PetService {
  constructor(private prisma: PrismaService) {}

  create(createPetInput: CreatePetInput) {
    return this.prisma.pet.create({
      data: { name: createPetInput.name },
    })
  }

  findAll() {
    return this.prisma.pet.findMany()
  }

  findOne(id: number) {
    return this.prisma.pet.findUnique({
      where: { id },
    })
  }

  update(id: number, updatePetInput: UpdatePetInput) {
    return this.prisma.pet.update({
      where: { id },
      data: {
        name: updatePetInput.name,
      },
    })
  }

  remove(id: number) {
    return this.prisma.pet.delete({
      where: { id },
    })
  }
}
```

```ts
// pet.resolver.ts
import { Resolver, Query, Mutation, Args } from '@nestjs/graphql'
import { CreatePetInput, UpdatePetInput } from 'src/graphql'
import { PetService } from './pet.service'

@Resolver('Pet')
export class PetResolver {
  constructor(private readonly petService: PetService) {}

  @Mutation('createPet')
  create(@Args('createPetInput') createPetInput: CreatePetInput) {
    return this.petService.create(createPetInput)
  }

  @Query('pet')
  findAll() {
    return this.petService.findAll()
  }

  @Query('pet')
  findOne(@Args('id') id: number) {
    return this.petService.findOne(id)
  }

  @Mutation('updatePet')
  update(@Args('updatePetInput') updatePetInput: UpdatePetInput) {
    return this.petService.update(updatePetInput.id, updatePetInput)
  }

  @Mutation('removePet')
  remove(@Args('id') id: number) {
    return this.petService.remove(id)
  }
}
```

```ts
// pet.module.ts
import { Module } from '@nestjs/common'
import { PetService } from './pet.service'
import { PetResolver } from './pet.resolver'

@Module({
  providers: [PetResolver, PetService],
})
export class PetModule {}
```

## 6. Running the app

```bash
# development
$ pnpm run start

# watch mode
$ pnpm run start:dev

# production mode
$ pnpm run start:prod
```

go to `localhost:3000/graphql` to see the graphql playground. You can start playing around with it. We will cover how to used GraphQL in next chapter so stay tuned.

You can see your database table with [Prisma Studio](https://www.prisma.io/studio)

```bash
npx prisma studio
```

## 7. Test

In this tutorial we didn't write any tests but if you want you can. Run those command to execute tests.

```bash
# unit tests
$ pnpm run test

# e2e tests
$ pnpm run test:e2e

# test coverage
$ pnpm run test:cov
```

## Conclusion

We have learn how to install and setup a Nest.js project with Prisma and GraphQL. I hope you learned somthing from it. The source code can be found in my [GitHub Repo](https://github.com/Qu1etboy/nestjs-prisma-graphql.git)

## References

- https://docs.nestjs.com/graphql/quick-start#schema-first
- https://www.youtube.com/watch?v=twi33GVRazE&t=631s&ab_channel=RichardHong
