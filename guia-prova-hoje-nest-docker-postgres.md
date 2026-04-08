# Guia da prova de hoje — Nest + Docker + PostgreSQL

Este guia junta, em um único lugar, o passo a passo ideal para fazer **hoje** a parte mais importante da prova: **backend + banco**, usando **NestJS + Prisma + PostgreSQL no Docker**.

A proposta da prova pede que as **4 operações do CRUD** estejam funcionando e que, para cada uma, você **mostre no banco via SQL** que o dado foi inserido, alterado ou removido.

---

## 1. O que abrir hoje

Abra nesta ordem:

1. **Docker Desktop**
2. **VSCode**
3. **Terminal do VSCode**
4. **Postman** ou navegador

Hoje, foque em:
- subir o banco com Docker
- criar o backend Nest
- configurar Prisma
- fazer o CRUD de `Customer`
- testar no Postman/Swagger
- provar tudo no banco com SQL

---

## 2. Estrutura inicial da pasta

No terminal do VSCode:

```bash
mkdir prova-crud
cd prova-crud
mkdir backend
mkdir admin
git init
```

Hoje você vai usar **só o `backend`**. O `admin` pode ficar para amanhã.

---

## 3. Arquivo `docker-compose.yml`

Na raiz do projeto, crie um arquivo chamado `docker-compose.yml`:

```yaml
services:
  db:
    image: postgres:16
    container_name: prova_postgres
    environment:
      POSTGRES_DB: provadb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - prova_data:/var/lib/postgresql/data

volumes:
  prova_data:
```

### Subir o banco

Na raiz do projeto:

```bash
docker compose up -d
```

Para conferir se subiu:

```bash
docker ps
```

---

## 4. Criar o backend Nest

Entre na pasta do backend:

```bash
cd backend
npx @nestjs/cli new .
```

Escolha `npm` quando pedir.

---

## 5. Instalar Prisma e PostgreSQL

Ainda dentro de `backend`:

```bash
npm install prisma @prisma/client pg
npm install -D prisma
npx prisma init
```

---

## 6. Arquivo `.env`

No arquivo `backend/.env`, deixe assim:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/provadb?schema=public"
```

---

## 7. Arquivo `prisma/schema.prisma`

Substitua pelo seguinte:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Customer {
  id        Int      @id @default(autoincrement())
  name      String
  age       Int
  email     String   @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

Agora rode:

```bash
npx prisma generate
npx prisma db push
```

---

## 8. Gerar a estrutura de customers

Ainda em `backend`:

```bash
nest g module customers
nest g controller customers
nest g service customers
```

Depois crie a pasta do Prisma:

```bash
mkdir src\prisma
mkdir src\customers\dto
```

---

## 9. Arquivo `src/prisma/prisma.service.ts`

```ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

---

## 10. DTOs

### `src/customers/dto/create-customer.dto.ts`

```ts
export class CreateCustomerDto {
  name: string;
  age: number;
  email: string;
}
```

### `src/customers/dto/update-customer.dto.ts`

```ts
export class UpdateCustomerDto {
  name?: string;
  age?: number;
  email?: string;
}
```

---

## 11. Arquivo `src/customers/customers.service.ts`

```ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { CreateCustomerDto } from './dto/create-customer.dto';
import { UpdateCustomerDto } from './dto/update-customer.dto';

@Injectable()
export class CustomersService {
  constructor(private readonly prisma: PrismaService) {}

  async create(createCustomerDto: CreateCustomerDto) {
    return this.prisma.customer.create({
      data: createCustomerDto,
    });
  }

  async findAll() {
    return this.prisma.customer.findMany({
      orderBy: { id: 'asc' },
    });
  }

  async findOne(id: number) {
    const customer = await this.prisma.customer.findUnique({
      where: { id },
    });

    if (!customer) {
      throw new NotFoundException('Cliente não encontrado');
    }

    return customer;
  }

  async update(id: number, updateCustomerDto: UpdateCustomerDto) {
    await this.findOne(id);

    return this.prisma.customer.update({
      where: { id },
      data: updateCustomerDto,
    });
  }

  async remove(id: number) {
    await this.findOne(id);

    return this.prisma.customer.delete({
      where: { id },
    });
  }
}
```

---

## 12. Arquivo `src/customers/customers.controller.ts`

```ts
import { Body, Controller, Delete, Get, Param, Patch, Post } from '@nestjs/common';
import { CustomersService } from './customers.service';
import { CreateCustomerDto } from './dto/create-customer.dto';
import { UpdateCustomerDto } from './dto/update-customer.dto';

@Controller('customers')
export class CustomersController {
  constructor(private readonly customersService: CustomersService) {}

  @Post()
  create(@Body() createCustomerDto: CreateCustomerDto) {
    return this.customersService.create(createCustomerDto);
  }

  @Get()
  findAll() {
    return this.customersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.customersService.findOne(Number(id));
  }

  @Patch(':id')
  update(@Param('id') id: string, @Body() updateCustomerDto: UpdateCustomerDto) {
    return this.customersService.update(Number(id), updateCustomerDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.customersService.remove(Number(id));
  }
}
```

---

## 13. Arquivo `src/customers/customers.module.ts`

```ts
import { Module } from '@nestjs/common';
import { CustomersController } from './customers.controller';
import { CustomersService } from './customers.service';
import { PrismaService } from '../prisma/prisma.service';

@Module({
  controllers: [CustomersController],
  providers: [CustomersService, PrismaService],
})
export class CustomersModule {}
```

---

## 14. Arquivo `src/app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { CustomersModule } from './customers/customers.module';

@Module({
  imports: [CustomersModule],
})
export class AppModule {}
```

---

## 15. Opcional — Swagger no `src/main.ts`

Se quiser testar mais fácil pelo navegador:

```ts
import { NestFactory } from '@nestjs/core';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.enableCors();

  const config = new DocumentBuilder()
    .setTitle('CRUD Customers')
    .setDescription('API de clientes para a prova')
    .setVersion('1.0')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('docs', app, document);

  await app.listen(3000);
}
bootstrap();
```

Instale o Swagger:

```bash
npm install @nestjs/swagger swagger-ui-express
```

---

## 16. Rodar o backend

Dentro de `backend`:

```bash
npm run start:dev
```

Se der tudo certo:
- API: `http://localhost:3000/customers`
- Swagger: `http://localhost:3000/docs`

---

## 17. Testes no Postman

### Criar cliente
**POST** `http://localhost:3000/customers`

Body → raw → JSON:

```json
{
  "name": "Patrick",
  "age": 20,
  "email": "patrick@email.com"
}
```

### Listar
**GET** `http://localhost:3000/customers`

### Buscar por id
**GET** `http://localhost:3000/customers/1`

### Atualizar
**PATCH** `http://localhost:3000/customers/1`

```json
{
  "name": "Patrick Melo",
  "age": 21
}
```

### Deletar
**DELETE** `http://localhost:3000/customers/1`

---

## 18. Como provar no banco via SQL

Na **raiz do projeto**, abra outro terminal e rode:

```bash
docker compose exec db psql -U postgres -d provadb
```

Quando aparecer:

```text
provadb=#
```

você já está dentro do banco.

### Ver tabelas

```sql
\dt
```

### Ver tudo da tabela

```sql
SELECT * FROM "Customer" ORDER BY id;
```

### Ver um registro específico

```sql
SELECT * FROM "Customer" WHERE id = 1;
```

### Sair do psql

```sql
\q
```

---

## 19. Como apresentar cada operação

### CREATE
1. Faça o POST no Postman/Swagger
2. No banco, rode:

```sql
SELECT * FROM "Customer" ORDER BY id;
```

### READ
1. Faça o GET
2. Mostre que o retorno bate com:

```sql
SELECT * FROM "Customer" ORDER BY id;
```

### UPDATE
1. Faça o PATCH
2. Depois rode:

```sql
SELECT * FROM "Customer" WHERE id = 1;
```

### DELETE
1. Faça o DELETE
2. Depois rode:

```sql
SELECT * FROM "Customer" WHERE id = 1;
```

Se aparecer:

```text
(0 rows)
```

então o delete foi provado.

---

## 20. O que fazer se der erro

### O banco não sobe

```bash
docker compose up -d
```

### A tabela não existe

Dentro de `backend`:

```bash
npx prisma generate
npx prisma db push
```

### A API não responde

```bash
npm run start:dev
```

### Erro de conexão com banco
Confira:
- se o Docker Desktop está aberto
- se o container está rodando
- se o `.env` está certo
- se a porta `5432` não está ocupada

---

## 21. Checklist final de hoje

Antes de encerrar o dia, veja se você já fez tudo isso:

- [ ] Docker Desktop aberto
- [ ] `docker compose up -d` rodou
- [ ] backend Nest criado
- [ ] Prisma instalado
- [ ] `.env` configurado
- [ ] `schema.prisma` pronto
- [ ] `npx prisma generate` rodou
- [ ] `npx prisma db push` rodou
- [ ] `prisma.service.ts` criado
- [ ] DTOs criados
- [ ] `customers.service.ts` criado
- [ ] `customers.controller.ts` criado
- [ ] `customers.module.ts` criado
- [ ] `app.module.ts` ajustado
- [ ] `npm run start:dev` funcionando
- [ ] POST testado
- [ ] GET testado
- [ ] PATCH testado
- [ ] DELETE testado
- [ ] SQL comprovando cada operação
- [ ] `git add .`
- [ ] `git commit -m "feat: backend crud customers"`

---

## 22. O que deixar para amanhã

Amanhã você pode fazer:
- o Angular simples
- integração do front com o backend
- tela de listagem
- formulário de cadastro/edição
- botão de excluir
- revisão da apresentação

Hoje, o foco ideal é deixar o **backend e o banco 100% funcionais**.
