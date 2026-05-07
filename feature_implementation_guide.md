# How to Implement a New Feature in NestJS Boilerplate

## 1. Choosing Your Database: PostgreSQL vs. MongoDB

Before starting, you need to decide whether to build your feature for a Relational DB (PostgreSQL) or Document DB (MongoDB).

### Comparison: PostgreSQL vs MongoDB

| Feature            | PostgreSQL (Relational)                        | MongoDB (Document)                                                            |
| :----------------- | :--------------------------------------------- | :---------------------------------------------------------------------------- |
| **Data Structure** | Strict schemas (Tables & Columns)              | Flexible schemas (JSON-like Documents)                                        |
| **Relationships**  | Strong relationships (Foreign Keys, JOINs)     | Embedded documents or loose references                                        |
| **Transactions**   | ACID compliant, strong consistency             | ACID compliant (multi-document), but optimized for single document operations |
| **Scaling**        | Vertical scaling (bigger server) is easier     | Horizontal scaling (sharding) is built-in                                     |
| **Migration**      | Requires migration scripts when schema changes | Schema-less, so no rigid migrations required                                  |

### Feature Mapping: When to use which?

In this boilerplate, core features (like `Users`, `Session`, `Files`) are actually built to support **both** databases via a Repository pattern. However, if you are designing a system that uses **both databases simultaneously** (Polyglot Persistence), here is a mapping of which features should use which database and why:

#### 🟢 Features using PostgreSQL (Relational)

- **Users, Roles, & Authentication:** Needs strict data integrity, uniqueness constraints (e.g., email), and complex relationships (Users have many Roles).
- **Billing & Subscriptions:** Requires ACID transactions to ensure financial data is perfectly consistent.
- **Inventory & Orders:** Strict relationships are needed (An Order belongs to a User and contains many Products, which deducts from Inventory).

#### 🟡 Features using MongoDB (Document)

- **Activity Logs & Audit Trails:** High write volume, data doesn't change often, and doesn't need to be strictly related to other tables via JOINs.
- **User Preferences / Settings:** Settings can be a deeply nested object that changes frequently. Document DBs handle flexible JSON objects perfectly.
- **Product Catalog with Dynamic Attributes:** If different products have wildly different attributes (e.g., a TV has "screen size" but a Shirt has "fabric type"), MongoDB handles these dynamic schemas better than a rigid SQL table.

---

## 2. Generate the Resource with Hygen

Instead of creating files manually, use the project's built-in script powered by **Hygen**. Hygen is a code generator that helps maintain consistency and speeds up development by scaffolding the necessary boilerplate for a new feature.

### Why use Hygen?

- **Speed**: Instantly create all files needed for a feature (Controller, Service, Repository, etc.).
- **Consistency**: Ensures every feature follows the same folder structure and naming conventions.
- **Auto-registration**: Automatically updates `app.module.ts` to include your new module.
- **Clean Architecture**: Scaffolds files according to the project's architecture (Domain, Infrastructure, DTOs).

### Commands

Depending on your database choice, use one of the following commands:

| Command                                | Usage                                                 |
| :------------------------------------- | :---------------------------------------------------- |
| `npm run generate:resource:relational` | Scaffolds a feature for **PostgreSQL** (TypeORM).     |
| `npm run generate:resource:document`   | Scaffolds a feature for **MongoDB** (Mongoose).       |
| `npm run generate:resource:all-db`     | Scaffolds a feature that supports **both** databases. |

**Example:**

```bash
npm run generate:resource:relational -- --name Inventory
```

### What gets generated?

The script creates a new directory in `src/` (e.g., `src/inventory/`) with the following structure:

- **`domain/`**: Contains the core business entity.
- **`dto/`**: Contains Data Transfer Objects for requests (Create, Update, Query).
- **`infrastructure/`**: Contains database-specific implementations (Entities, Repositories, Mappers).
- **`inventory.controller.ts`**: The entry point for HTTP requests.
- **`inventory.service.ts`**: Contains the business logic.
- **`inventory.module.ts`**: The NestJS module definition.

### Core Concepts (The "Why")

- **🎯 Domain**: The heart of the app. It contains the business rules and "pure" data models, independent of databases or frameworks.
- **🏛️ Infrastructure**: The layer for external tools (DB, APIs). It keeps the rest of the app "clean" from technical details.
- **💾 TypeORM**: The tool that maps your TypeScript code to PostgreSQL tables.
- **🔄 Mapper**: A translator that converts between database data and pure business data. This allows you to change your database without breaking your business logic.
- **📦 Repository**: A standard way to access data. The Service calls the Repository, which then talks to the Database.

---

## 3. Add Properties to the Entity

Once you have the base resource, you can add properties (fields) to your entities and DTOs using the property generator. Define the fields your feature needs (e.g., `sku`, `quantity`).

### Command Parameters

- `--name`: The name of your feature (e.g., `Inventory`).
- `--property`: The name of the field (e.g., `sku`).
- `--kind`: The kind of property (`primitive`, `relation`, etc.).
- `--type`: The TypeScript/Database type (e.g., `string`, `number`, `Date`).
- `--isAddToDto`: (true/false) Whether to add this property to the `Create` and `Update` DTOs.
- `--isOptional`: (true/false) Whether the property is optional in the DTO.
- `--isNullable`: (true/false) Whether the database column can be null.

**Example:**

```bash
npm run add:property:to-relational -- --name Inventory --property sku --kind primitive --type string --isAddToDto true --isOptional false --isNullable false

npm run add:property:to-relational -- --name Inventory --property quantity --kind primitive --type number --isAddToDto true --isOptional false --isNullable false
```

## 4. Coding the Feature: DTOs, Service, and Controller

Once the skeleton is generated, you need to write the actual code for your feature.

### A. Define the DTO (Data Transfer Object)

DTOs validate incoming request payloads. The generator created `create-inventory.dto.ts` for you. You should add `class-validator` decorators to enforce rules.

**Example `src/inventory/dto/create-inventory.dto.ts`:**

```typescript
import { ApiProperty } from '@nestjs/swagger';
import { IsNotEmpty, IsString, IsNumber, Min } from 'class-validator';

export class CreateInventoryDto {
  @ApiProperty({ example: 'LAPTOP-X200' })
  @IsString()
  @IsNotEmpty()
  sku: string;

  @ApiProperty({ example: 50 })
  @IsNumber()
  @Min(0)
  quantity: number;
}
```

### B. Implement the Service Logic

The service handles business logic (e.g., checking stock or updating inventory counts).

**Example `src/inventory/inventory.service.ts`:**

```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { InventoryEntity } from './entities/inventory.entity';
import { CreateInventoryDto } from './dto/create-inventory.dto';

@Injectable()
export class InventoryService {
  constructor(
    @InjectRepository(InventoryEntity)
    private readonly inventoryRepository: Repository<InventoryEntity>,
  ) {}

  async create(
    createInventoryDto: CreateInventoryDto,
  ): Promise<InventoryEntity> {
    // Implement custom logic here, e.g., checking if SKU already exists
    const newInventoryRecord =
      this.inventoryRepository.create(createInventoryDto);
    return await this.inventoryRepository.save(newInventoryRecord);
  }

  async findAll(): Promise<InventoryEntity[]> {
    return await this.inventoryRepository.find();
  }
}
```

### C. Connect to the Controller

The controller handles HTTP requests and calls the service.

**Example `src/inventory/inventory.controller.ts`:**

```typescript
import { Controller, Post, Body, Get } from '@nestjs/common';
import { ApiTags } from '@nestjs/swagger';
import { InventoryService } from './inventory.service';
import { CreateInventoryDto } from './dto/create-inventory.dto';

@ApiTags('Inventory')
@Controller({ path: 'inventory', version: '1' })
export class InventoryController {
  constructor(private readonly inventoryService: InventoryService) {}

  @Post()
  create(@Body() createInventoryDto: CreateInventoryDto) {
    return this.inventoryService.create(createInventoryDto);
  }

  @Get()
  findAll() {
    return this.inventoryService.findAll();
  }
}
```

## 5. Generate and Run Migrations (Relational DB only)

If you chose PostgreSQL, you must create a database migration for your new `Inventory` entity.

**What is a migration and why is it needed?**
In Relational Databases (like PostgreSQL), the database table structure must exactly match your application code. When you created the `Inventory` entity and added properties (like `sku` and `quantity`), you only changed the TypeScript code. The database itself doesn't know about these changes yet.
A migration is an auto-generated file containing SQL commands that safely updates your database schema (e.g., creating the `inventory` table with `sku` and `quantity` columns) without breaking existing data. It also allows you to keep track of database changes over time just like Git tracks code changes.

Generate the migration file:

```bash
npm run migration:generate -- src/database/migrations/CreateInventoryTable
```

Then apply the migration:

```bash
npm run migration:run
```

## 6. Verify App Module

Finally, check `src/app.module.ts` to ensure the generator automatically added `InventoryModule` to the `imports` array. If not, add it manually.

---

## 7. Using gRPC for Inter-Service Communication

If your architecture involves multiple microservices and you need to communicate between them, **gRPC** is highly recommended for its performance and strongly-typed contracts using Protocol Buffers.

### A. Install Required Dependencies

First, ensure you have the NestJS microservices and gRPC packages installed:

```bash
npm install @nestjs/microservices @grpc/grpc-js @grpc/proto-loader
```

### B. Define the Protocol Buffer (.proto) File

Create a `.proto` file to define the service contract.
**Example `src/inventory/inventory.proto`:**

```proto
syntax = "proto3";

package inventory;

service InventoryService {
  rpc CheckStock (StockRequest) returns (StockResponse) {}
}

message StockRequest {
  string sku = 1;
}

message StockResponse {
  bool isAvailable = 1;
  int32 quantity = 2;
}
```

### C. Configure the gRPC Client

To call another service, you need to register a gRPC client in your module.

**Example `src/orders/orders.module.ts`:**

```typescript
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { join } from 'path';
import { OrdersService } from './orders.service';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'INVENTORY_PACKAGE',
        transport: Transport.GRPC,
        options: {
          package: 'inventory',
          protoPath: join(__dirname, '../inventory/inventory.proto'),
          url: 'localhost:5000', // Address of the target gRPC service
        },
      },
    ]),
  ],
  providers: [OrdersService],
})
export class OrdersModule {}
```

### D. Call the gRPC Service

Inject the client into your service and call the remote method. Note that gRPC returns RxJS Observables, so you may want to convert them to Promises using `lastValueFrom`.

**Example `src/orders/orders.service.ts`:**

```typescript
import { Injectable, Inject, OnModuleInit } from '@nestjs/common';
import { ClientGrpc } from '@nestjs/microservices';
import { lastValueFrom } from 'rxjs';

interface InventoryGrpcService {
  checkStock(data: { sku: string }): any; // Ideally, define a proper response interface
}

@Injectable()
export class OrdersService implements OnModuleInit {
  private inventoryService: InventoryGrpcService;

  constructor(@Inject('INVENTORY_PACKAGE') private client: ClientGrpc) {}

  onModuleInit() {
    this.inventoryService =
      this.client.getService<InventoryGrpcService>('InventoryService');
  }

  async verifyInventory(sku: string) {
    const response = await lastValueFrom(
      this.inventoryService.checkStock({ sku }),
    );
    return response;
  }
}
```

---

## 8. Running Tests

After implementing your new feature, it is highly recommended to test it. The boilerplate comes with built-in configurations for Unit and E2E testing.

### Unit Tests

Run your unit tests to verify your feature's isolated logic:

```bash
npm run test
```

### End-to-End (E2E) Tests

Run E2E tests to test the whole application flow:

```bash
npm run test:e2e
```

**Running E2E tests inside Docker:**
If you want to run tests cleanly in an isolated Docker environment, the boilerplate provides dedicated scripts for both databases:

- **Relational DB:** `npm run test:e2e:relational:docker`
- **Document DB:** `npm run test:e2e:document:docker`

---

## 9. Deployment

When your feature is fully implemented and tested, you can deploy the application.

### Local or VPS Deployment (Node.js)

If you are deploying directly to a Node.js server without Docker:

1. Build the production bundle:
   ```bash
   npm run build
   ```
2. Start the application in production mode:
   ```bash
   npm run start:prod
   ```

### Docker Deployment

The boilerplate includes `Dockerfile` and `docker-compose.yaml` configurations, making it extremely easy to deploy via Docker.

1. Make sure your `.env` is configured correctly for production (e.g., strong passwords, changing `MAIL_HOST` from localhost to your provider).
2. Start the application containers in detached mode:

   ```bash
   # For PostgreSQL
   docker compose up -d

   # For MongoDB
   docker compose -f docker-compose.document.yaml up -d
   ```
