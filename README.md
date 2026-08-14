# راه‌اندازی Prisma 7 در NestJS 11

در این راهنما نحوه‌ی اتصال **Prisma 7** به **NestJS 11** را به‌صورت قدم‌به‌قدم یاد می‌گیریم.

این راهنما هر دو دیتابیس زیر را پوشش می‌دهد:

* PostgreSQL
* MySQL

> نکته: در Prisma 7 برای اتصال به دیتابیس از **Driver Adapter** استفاده می‌کنیم.

---

# فهرست مطالب

1. نصب پکیج‌ها
2. مقداردهی اولیه Prisma
3. تنظیم `.env`
4. تنظیم `schema.prisma`
5. اتصال به PostgreSQL
6. اتصال به MySQL
7. اجرای Migration
8. ساخت PrismaService
9. ساخت PrismaModule
10. ثبت PrismaModule در AppModule
11. استفاده از Prisma در Service
12. Controller
13. ساختار نهایی پروژه
14. دستورات مهم Prisma
15. نکات مهم Prisma 7

---

# قدم ۱: نصب پکیج‌های مورد نیاز

## برای PostgreSQL

داخل پروژه NestJS:

```bash
npm install prisma --save-dev
npm install @prisma/client
npm install @prisma/adapter-pg pg
npm install dotenv
```

---

## برای MySQL

اگر از MySQL استفاده می‌کنی:

```bash
npm install prisma --save-dev
npm install @prisma/client
npm install @prisma/adapter-mariadb mariadb
npm install dotenv
```

> در این راهنما برای MySQL از `@prisma/adapter-mariadb` استفاده می‌کنیم.

---

# قدم ۲: مقداردهی اولیه Prisma

در ریشه‌ی پروژه NestJS اجرا کن:

```bash
npx prisma init
```

ساختار اولیه:

```text
my-nest-project/
├── .env
├── prisma/
│   └── schema.prisma
├── src/
├── package.json
└── ...
```

دستور `prisma init` فایل‌های اولیه Prisma را ایجاد می‌کند.

---

# قدم ۳: تنظیم فایل `.env`

فایل `.env` باید در ریشه‌ی پروژه و کنار `package.json` قرار داشته باشد:

```text
my-nest-project/
├── .env
├── package.json
├── prisma/
└── src/
```

---

# اتصال به PostgreSQL

اگر PostgreSQL استفاده می‌کنی:

```env
DATABASE_URL="postgresql://postgres:root@localhost:5432/nest"
```

فرمت کلی:

```text
postgresql://USER:PASSWORD@HOST:PORT/DATABASE
```

مثال:

```env
DATABASE_URL="postgresql://postgres:123456@localhost:5432/my_database"
```

---

# اتصال به MySQL

اگر MySQL استفاده می‌کنی:

```env
DATABASE_URL="mysql://root:root@localhost:3306/nest"
```

فرمت کلی:

```text
mysql://USER:PASSWORD@HOST:PORT/DATABASE
```

مثال:

```env
DATABASE_URL="mysql://root:123456@localhost:3306/my_database"
```

اگر برای MySQL رمز عبور نداری، معمولاً می‌تواند به شکل زیر باشد:

```env
DATABASE_URL="mysql://root@localhost:3306/nest"
```

---

# قدم ۴: تنظیم `schema.prisma`

فایل:

```text
prisma/schema.prisma
```

در Prisma 7 می‌توانیم Client را در مسیر دلخواه تولید کنیم.

مثلاً:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}
```

---

# تنظیم برای PostgreSQL

اگر PostgreSQL استفاده می‌کنی:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

---

# تنظیم برای MySQL

اگر MySQL استفاده می‌کنی:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "mysql"
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

تفاوت اصلی فقط این قسمت است:

### PostgreSQL

```prisma
provider = "postgresql"
```

### MySQL

```prisma
provider = "mysql"
```

---

# قدم ۵: اجرای Migration

بعد از تنظیم `schema.prisma`:

```bash
npx prisma migrate dev --name init
```

این دستور:

* Migration ایجاد می‌کند
* دیتابیس را به‌روزرسانی می‌کند
* Prisma Client را Generate می‌کند

مثلاً:

```text
prisma/
├── migrations/
│   └── 20260814_init/
│       └── migration.sql
└── schema.prisma
```

---

# قدم ۶: Generate کردن Prisma Client

در صورت نیاز می‌توانی Client را دستی Generate کنی:

```bash
npx prisma generate
```

با توجه به تنظیم زیر:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}
```

فایل‌ها در این مسیر قرار می‌گیرند:

```text
src/generated/prisma/
```

---

# قدم ۷: ساخت PrismaService

حالا باید Prisma را به NestJS متصل کنیم.

مسیر فایل:

```text
src/prisma/prisma.service.ts
```

---

# PrismaService برای PostgreSQL

```typescript
import 'dotenv/config';

import {
  Injectable,
  OnModuleDestroy,
  OnModuleInit,
} from '@nestjs/common';

import { PrismaPg } from '@prisma/adapter-pg';
import { PrismaClient } from '../generated/prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  constructor() {
    const adapter = new PrismaPg({
      connectionString: process.env.DATABASE_URL!,
    });

    super({
      adapter,
    });
  }

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

> مسیر import را با توجه به محل `generated` در پروژه‌ی خودت تنظیم کن.

---

# PrismaService برای MySQL

اگر MySQL استفاده می‌کنی:

```typescript
import 'dotenv/config';

import {
  Injectable,
  OnModuleDestroy,
  OnModuleInit,
} from '@nestjs/common';

import { PrismaMariaDb } from '@prisma/adapter-mariadb';
import { PrismaClient } from '../generated/prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  constructor() {
    const adapter = new PrismaMariaDb({
      host: process.env.DATABASE_HOST!,
      port: Number(process.env.DATABASE_PORT ?? 3306),
      user: process.env.DATABASE_USER!,
      password: process.env.DATABASE_PASSWORD!,
      database: process.env.DATABASE_NAME!,
      connectionLimit: 5,
    });

    super({
      adapter,
    });
  }

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

برای MySQL می‌توانی تنظیمات اتصال را در `.env` به شکل زیر قرار بدهی:

```env
DATABASE_HOST="localhost"
DATABASE_PORT="3306"
DATABASE_USER="root"
DATABASE_PASSWORD="root"
DATABASE_NAME="nest"
```

---

# نکته مهم درباره MySQL

در PostgreSQL می‌توانیم مستقیماً `DATABASE_URL` را به adapter بدهیم:

```typescript
const adapter = new PrismaPg({
  connectionString: process.env.DATABASE_URL!,
});
```

اما در این روش MySQL با `PrismaMariaDb` تنظیمات اتصال را به‌صورت جداگانه دریافت می‌کند:

```typescript
const adapter = new PrismaMariaDb({
  host: process.env.DATABASE_HOST!,
  port: Number(process.env.DATABASE_PORT ?? 3306),
  user: process.env.DATABASE_USER!,
  password: process.env.DATABASE_PASSWORD!,
  database: process.env.DATABASE_NAME!,
});
```

---

# قدم ۸: ساخت PrismaModule

فایل:

```text
src/prisma/prisma.module.ts
```

```typescript
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

---

# چرا از `@Global()` استفاده می‌کنیم؟

با:

```typescript
@Global()
```

ماژول Prisma به‌صورت Global در برنامه در دسترس قرار می‌گیرد.

در نتیجه اگر `PrismaModule` یک‌بار در `AppModule` ثبت شود، می‌توانیم `PrismaService` را در ماژول‌های مختلف Inject کنیم.

مثلاً:

```typescript
constructor(private readonly prisma: PrismaService) {}
```

---

# قدم ۹: ثبت PrismaModule در AppModule

فایل:

```text
src/app.module.ts
```

```typescript
import { Module } from '@nestjs/common';
import { PrismaModule } from './prisma/prisma.module';
import { PostsModule } from './posts/posts.module';

@Module({
  imports: [
    PrismaModule,
    PostsModule,
  ],
})
export class AppModule {}
```

---

# قدم ۱۰: استفاده از Prisma در Service

مثلاً می‌خواهیم یک سیستم Posts داشته باشیم.

فرض کنیم در `schema.prisma` این مدل را داریم:

```prisma
model Post {
  id        Int      @id @default(autoincrement())
  title     String
  body      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

بعد از تغییر Schema:

```bash
npx prisma migrate dev --name create_posts
```

---

# ساخت PostsService

فایل:

```text
src/posts/posts.service.ts
```

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class PostsService {
  constructor(private readonly prisma: PrismaService) {}

  findAll() {
    return this.prisma.post.findMany();
  }

  findOne(id: number) {
    return this.prisma.post.findUnique({
      where: {
        id,
      },
    });
  }

  create(data: {
    title: string;
    body?: string;
  }) {
    return this.prisma.post.create({
      data,
    });
  }

  update(
    id: number,
    data: {
      title?: string;
      body?: string;
    },
  ) {
    return this.prisma.post.update({
      where: {
        id,
      },
      data,
    });
  }

  delete(id: number) {
    return this.prisma.post.delete({
      where: {
        id,
      },
    });
  }
}
```

---

# قدم ۱۱: ساخت Controller

فایل:

```text
src/posts/posts.controller.ts
```

```typescript
import {
  Body,
  Controller,
  Delete,
  Get,
  Param,
  Post,
  Put,
} from '@nestjs/common';

import { PostsService } from './posts.service';

@Controller('posts')
export class PostsController {
  constructor(
    private readonly postsService: PostsService,
  ) {}

  @Get()
  findAll() {
    return this.postsService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.postsService.findOne(Number(id));
  }

  @Post()
  create(
    @Body()
    createPostDto: {
      title: string;
      body?: string;
    },
  ) {
    return this.postsService.create(createPostDto);
  }

  @Put(':id')
  update(
    @Param('id') id: string,
    @Body()
    updatePostDto: {
      title?: string;
      body?: string;
    },
  ) {
    return this.postsService.update(
      Number(id),
      updatePostDto,
    );
  }

  @Delete(':id')
  delete(@Param('id') id: string) {
    return this.postsService.delete(Number(id));
  }
}
```

---

# قدم ۱۲: ساخت PostsModule

فایل:

```text
src/posts/posts.module.ts
```

```typescript
import { Module } from '@nestjs/common';

import { PostsController } from './posts.controller';
import { PostsService } from './posts.service';

@Module({
  controllers: [PostsController],
  providers: [PostsService],
})
export class PostsModule {}
```

چون `PrismaModule` با `@Global()` تعریف شده، نیازی نیست آن را دوباره داخل `PostsModule` import کنیم.

---

# ساختار نهایی پروژه

ساختار پروژه می‌تواند به شکل زیر باشد:

```text
my-nest-project/
│
├── .env
├── package.json
├── tsconfig.json
│
├── prisma/
│   ├── migrations/
│   │   └── ...
│   │
│   └── schema.prisma
│
└── src/
    │
    ├── generated/
    │   └── prisma/
    │       └── ...
    │
    ├── prisma/
    │   ├── prisma.module.ts
    │   └── prisma.service.ts
    │
    ├── posts/
    │   ├── posts.module.ts
    │   ├── posts.service.ts
    │   └── posts.controller.ts
    │
    ├── app.module.ts
    └── main.ts
```

---

# PostgreSQL یا MySQL؟

اگر PostgreSQL استفاده می‌کنی:

### نصب

```bash
npm install @prisma/adapter-pg pg
```

### Schema

```prisma
datasource db {
  provider = "postgresql"
}
```

### Environment

```env
DATABASE_URL="postgresql://postgres:root@localhost:5432/nest"
```

### Adapter

```typescript
import { PrismaPg } from '@prisma/adapter-pg';

const adapter = new PrismaPg({
  connectionString: process.env.DATABASE_URL!,
});
```

---

اگر MySQL استفاده می‌کنی:

### نصب

```bash
npm install @prisma/adapter-mariadb mariadb
```

### Schema

```prisma
datasource db {
  provider = "mysql"
}
```

### Environment

```env
DATABASE_HOST="localhost"
DATABASE_PORT="3306"
DATABASE_USER="root"
DATABASE_PASSWORD="root"
DATABASE_NAME="nest"
```

### Adapter

```typescript
import { PrismaMariaDb } from '@prisma/adapter-mariadb';

const adapter = new PrismaMariaDb({
  host: process.env.DATABASE_HOST!,
  port: Number(process.env.DATABASE_PORT ?? 3306),
  user: process.env.DATABASE_USER!,
  password: process.env.DATABASE_PASSWORD!,
  database: process.env.DATABASE_NAME!,
});
```

---

# دستورات مهم Prisma

## ساخت Prisma Client

```bash
npx prisma generate
```

---

## ساخت Migration

```bash
npx prisma migrate dev --name init
```

مثلاً:

```bash
npx prisma migrate dev --name create_users
```

---

## مشاهده دیتابیس با Prisma Studio

```bash
npx prisma studio
```

---

## بررسی Schema

```bash
npx prisma validate
```

---

## فرمت کردن Schema

```bash
npx prisma format
```

---

## مشاهده وضعیت Migrationها

```bash
npx prisma migrate status
```

---

# نکته مهم درباره Prisma 7

در نسخه‌های جدید Prisma، نحوه‌ی اتصال Prisma Client به دیتابیس با نسخه‌های قدیمی متفاوت است.

در پروژه‌های Prisma 7 بهتر است Driver Adapter متناسب با دیتابیس استفاده شود.

### PostgreSQL

```text
@prisma/adapter-pg
```

### MySQL

```text
@prisma/adapter-mariadb
```

بنابراین این ساختار:

```typescript
const adapter = new PrismaPg({
  connectionString: process.env.DATABASE_URL!,
});

super({
  adapter,
});
```

برای PostgreSQL است.

و این ساختار:

```typescript
const adapter = new PrismaMariaDb({
  host: process.env.DATABASE_HOST!,
  port: Number(process.env.DATABASE_PORT ?? 3306),
  user: process.env.DATABASE_USER!,
  password: process.env.DATABASE_PASSWORD!,
  database: process.env.DATABASE_NAME!,
});

super({
  adapter,
});
```

برای MySQL است.

---

# نکته درباره `dotenv`

در `prisma.service.ts` می‌توانیم از:

```typescript
import 'dotenv/config';
```

استفاده کنیم تا متغیرهای `.env` در زمان اجرای سرویس در دسترس باشند.

مثلاً:

```typescript
process.env.DATABASE_URL
```

یا:

```typescript
process.env.DATABASE_HOST
```

---

# نکته درباره مسیر Prisma Client

اگر در `schema.prisma` نوشته باشیم:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}
```

Prisma Client در این مسیر تولید می‌شود:

```text
src/generated/prisma/
```

پس باید import را نیز مطابق همین مسیر انجام دهیم:

```typescript
import { PrismaClient } from '../generated/prisma/client';
```

اگر ساختار پروژه متفاوت است، مسیر import را اصلاح کن.

---

# نکته درباره Migration

هر زمان مدل‌های `schema.prisma` را تغییر دادی، در محیط Development بهتر است Migration جدید بسازی.

مثلاً مدل:

```prisma
model User {
  id    Int    @id @default(autoincrement())
  name  String
  email String @unique
}
```

بعداً یک فیلد اضافه می‌کنی:

```prisma
model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  phone     String?
  createdAt DateTime @default(now())
}
```

سپس:

```bash
npx prisma migrate dev --name add_phone_to_user
```

---

# نکته درباره Production

برای Development:

```bash
npx prisma migrate dev
```

برای Production معمولاً:

```bash
npx prisma migrate deploy
```

استفاده می‌شود.

---

# جمع‌بندی

برای راه‌اندازی Prisma 7 در NestJS 11 مراحل کلی این است:

```text
1. نصب Prisma
        ↓
2. npx prisma init
        ↓
3. تنظیم .env
        ↓
4. تنظیم schema.prisma
        ↓
5. نصب Driver Adapter
        ↓
6. npx prisma migrate dev
        ↓
7. npx prisma generate
        ↓
8. ساخت PrismaService
        ↓
9. ساخت PrismaModule
        ↓
10. ثبت PrismaModule
        ↓
11. Inject کردن PrismaService
        ↓
12. استفاده از Prisma در Service
```

## PostgreSQL

```text
PostgreSQL
    ↓
@prisma/adapter-pg
    ↓
PrismaPg
    ↓
PrismaClient
    ↓
NestJS
```

## MySQL

```text
MySQL
    ↓
@prisma/adapter-mariadb
    ↓
PrismaMariaDb
    ↓
PrismaClient
    ↓
NestJS
```

به این ترتیب می‌توانی Prisma 7 را در NestJS 11 با هر دو دیتابیس **PostgreSQL** و **MySQL** راه‌اندازی و استفاده کنی.
