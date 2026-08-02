# راه‌اندازی Prisma در NestJS (قدم به قدم)

سلام! بر اساس تجربه‌ای که در چت داشتیم، اینجا روش کامل و درست اتصال Prisma 7 به NestJS 11 رو قدم به قدم توضیح می‌دم.

## قدم ۱: نصب پکیج‌های مورد نیاز

```bash
# داخل پروژه NestJS
npm install prisma --save-dev
npm install @prisma/client
npm install @prisma/adapter-pg pg
npm install dotenv
```

## قدم ۲: مقداردهی اولیه Prisma

```bash
npx prisma init
```

این دستور دو چیز می‌سازه:
- پوشه `prisma/` با فایل `schema.prisma`
- فایل `.env` در ریشه پروژه

## قدم ۳: تنظیم فایل .env

فایل `.env` باید کنار `package.json` باشه، نه داخل `src/`:

```text
DATABASE_URL="postgresql://postgres:root@localhost:5432/nest"
```

**نکته مهم:** اگر از PostgreSQL استفاده می‌کنی، فرمت باید اینطور باشه:
```
postgresql://USER:PASSWORD@HOST:PORT/DATABASE
```

## قدم ۴: تنظیم schema.prisma

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client"
  output   = "../generated/prisma"   // مسیر خروجی سفارشی
}

datasource db {
  provider = "postgresql"
}

// مدل نمونه
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

## قدم ۵: اجرای مهاجرت (Migration)

```bash
npx prisma migrate dev --name init
```

این کار:
- فایل‌های migration رو می‌سازه
- دیتابیس رو آپدیت می‌کنه
- PrismaClient رو در مسیر مشخص شده (`src/generated/prisma`) تولید می‌کنه

## قدم ۶: ساخت Prisma Service

بساز: `src/prisma/prisma.service.ts`

```typescript
// src/prisma/prisma.service.ts

import 'dotenv/config';  // ⚠️ خیلی مهم - برای لود .env
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaPg } from '@prisma/adapter-pg';
import { PrismaClient } from '../generated/prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  constructor() {
    // adapter برای PostgreSQL در Prisma 7
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

**توجه:** برای Prisma 7 استفاده از adapter الزامی شده.

## قدم ۷: ساخت Prisma Module

بساز: `src/prisma/prisma.module.ts`

```typescript
// src/prisma/prisma.module.ts

import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global()  // 👈 این باعث می‌شود در همه جای پروژه در دسترس باشد
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

## قدم ۸: ثبت در AppModule

```typescript
// src/app.module.ts

import { Module } from '@nestjs/common';
import { PrismaModule } from './prisma/prisma.module';
import { PostsModule } from './posts/posts.module';

@Module({
  imports: [
    PrismaModule,   // 👈 حتماً اول باشه
    PostsModule,
  ],
})
export class AppModule {}
```

## قدم ۹: استفاده در Service ها

حالا می‌تونی PrismaService رو مثل هر سرویس دیگه‌ای Inject کنی:

```typescript
// src/posts/posts.service.ts

import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';

@Injectable()
export class PostsService {
  constructor(private prisma: PrismaService) {}

  findAll() {
    return this.prisma.post.findMany({
      include: {
        author: true,  // رابطه رو هم بیار
      },
    });
  }

  findOne(id: number) {
    return this.prisma.post.findUnique({
      where: { id },
    });
  }

  create(data: { title: string; body?: string; authorId: number }) {
    return this.prisma.post.create({
      data,
    });
  }

  update(id: number, data: { title?: string; body?: string }) {
    return this.prisma.post.update({
      where: { id },
      data,
    });
  }

  delete(id: number) {
    return this.prisma.post.delete({
      where: { id },
    });
  }
}
```

## قدم ۱۰: Controller (اختیاری)

```typescript
// src/posts/posts.controller.ts

import { Controller, Get, Post, Body, Param, Delete, Put } from '@nestjs/common';
import { PostsService } from './posts.service';

@Controller('posts')
export class PostsController {
  constructor(private readonly postsService: PostsService) {}

  @Get()
  findAll() {
    return this.postsService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.postsService.findOne(+id);
  }

  @Post()
  create(@Body() createPostDto: { title: string; body?: string; authorId: number }) {
    return this.postsService.create(createPostDto);
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updatePostDto: { title?: string; body?: string }) {
    return this.postsService.update(+id, updatePostDto);
  }

  @Delete(':id')
  delete(@Param('id') id: string) {
    return this.postsService.delete(+id);
  }
}
```

## قدم ۱۱: ساختار نهایی پروژه

```text
my-nest-project/
├── .env                      ✅
├── package.json
├── tsconfig.json
├── prisma/
│   └── schema.prisma         ✅
├── src/
│   ├── generated/            ← توسط Prisma ساخته می‌شود
│   │   └── prisma/
│   │       └── client/
│   ├── prisma/
│   │   ├── prisma.module.ts  ✅
│   │   └── prisma.service.ts ✅
│   ├── posts/
│   │   ├── posts.module.ts
│   │   ├── posts.service.ts  ✅
│   │   └── posts.controller.ts
│   └── app.module.ts         ✅
```

---

## نکات کلیدی

1. **مسیر خروجی Prisma Client** در `schema.prisma` به `../src/generated/prisma` تنظیم شده تا فایل‌های تولید شده در پروژه مدیریت شوند.

2. **adapter اجباری** در Prisma 7 برای PostgreSQL از پکیج `@prisma/adapter-pg` استفاده می‌شود.

3. **`dotenv/config`** در بالای فایل سرویس برای اطمینان از بارگذاری متغیرهای محیطی ضروری است.

4. **`@Global()`** در PrismaModule باعث می‌شود که PrismaService بدون نیاز به import در ماژول‌های دیگر در دسترس باشد.

5. **مدیریت اتصال** با پیاده‌سازی `OnModuleInit` و `OnModuleDestroy`، اتصال به دیتابیس در زمان راه‌اندازی و قطع آن در زمان خاموش شدن برنامه مدیریت می‌شود.
