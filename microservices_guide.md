# Microservices with API Gateway & gRPC Guide

This guide demonstrates how to build a microservices architecture using NestJS with:

- **1 API Gateway** — public-facing HTTP entry point
- **Product Service** — manages product catalog
- **Inventory Service** — manages stock levels

The services communicate internally via **gRPC**, while the gateway exposes a REST API to clients.

---

## Architecture Overview

```
┌──────────┐       HTTP        ┌──────────────┐      gRPC       ┌───────────────────┐
│  Client  │ ───────────────►  │  API Gateway │ ──────────────► │  Product Service  │
│ (Browser │                   │  (port 3000) │                 │  (port 5001)      │
│  / App)  │                   │              │      gRPC       ├───────────────────┤
└──────────┘                   │              │ ──────────────► │ Inventory Service │
                               └──────────────┘                 │  (port 5002)      │
                                                                └───────────────────┘
```

### Key Feature: Get Products Available in a Specific Inventory

The gateway calls the **Inventory Service** to get inventory items for a location, then calls the **Product Service** to enrich each item with product details — returning a combined response.

---

## Project Structure

```
microservices-root/
├── gateway/                    # API Gateway (HTTP → gRPC)
│   ├── src/
│   │   ├── product/
│   │   │   ├── product.controller.ts
│   │   │   ├── product.module.ts
│   │   │   └── product.service.ts
│   │   ├── inventory/
│   │   │   ├── inventory.controller.ts
│   │   │   ├── inventory.module.ts
│   │   │   └── inventory.service.ts
│   │   ├── app.module.ts
│   │   └── main.ts
│   └── package.json
│
├── product-service/            # Product microservice (gRPC server)
│   ├── src/
│   │   ├── product/
│   │   │   ├── domain/product.ts
│   │   │   ├── dto/
│   │   │   ├── infrastructure/persistence/...
│   │   │   ├── product.controller.ts
│   │   │   ├── product.service.ts
│   │   │   └── product.module.ts
│   │   ├── app.module.ts
│   │   └── main.ts
│   └── package.json
│
├── inventory-service/          # Inventory microservice (gRPC server)
│   ├── src/
│   │   ├── inventory/
│   │   │   ├── domain/inventory.ts
│   │   │   ├── dto/
│   │   │   ├── infrastructure/persistence/...
│   │   │   ├── inventory.controller.ts
│   │   │   ├── inventory.service.ts
│   │   │   └── inventory.module.ts
│   │   ├── app.module.ts
│   │   └── main.ts
│   └── package.json
│
└── proto/                      # Shared Proto definitions
    ├── product.proto
    └── inventory.proto
```

---

## 1. Shared Proto Definitions

Protocol Buffers (protobuf) are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data. They serve as the contract between the microservices.

**Key Concepts:**

- `syntax = "proto3";`: Specifies the protocol buffer version being used.
- `package`: Provides a namespace to prevent name conflicts between different projects.
- `service`: Defines a set of RPC (Remote Procedure Call) methods that can be called remotely.
- `rpc`: Defines a single remote method, specifying its request and response types.
- `message`: Defines the structure of the data being passed (similar to a class or interface).
- `int32`, `string`, `double`, `bool`: Basic data types for fields.
- `repeated`: Indicates an array or list of the specified type.
- `= 1`, `= 2`: These are unique "tag numbers" used to identify fields in the binary message format. They must be unique within a message and should not be changed once in use.

### `proto/product.proto`

```proto
syntax = "proto3"; // Use proto3 syntax

package product; // Namespace for the generated code

// Defines the interface for the Product service
service ProductService {
  rpc FindOne (ProductById) returns (Product) {} // Fetch a single product by ID
  rpc FindMany (ProductByIds) returns (ProductList) {} // Fetch multiple products by their IDs
  rpc FindAll (Empty) returns (ProductList) {} // Fetch all products
}

// An empty message used when a request or response has no data
message Empty {}

// Request message for finding a product by ID
message ProductById {
  int32 id = 1; // Field 1: The ID of the product
}

// Request message for finding multiple products
message ProductByIds {
  repeated int32 ids = 1; // Field 1: An array of product IDs
}

// Response message containing product details
message Product {
  int32 id = 1; // Field 1: Product ID
  string name = 2; // Field 2: Product name
  string description = 3; // Field 3: Product description
  double price = 4; // Field 4: Product price
  string sku = 5; // Field 5: Stock Keeping Unit (unique identifier)
}

// Response message containing a list of products
message ProductList {
  repeated Product products = 1; // Field 1: An array of Product messages
}
```

### `proto/inventory.proto`

```proto
syntax = "proto3"; // Use proto3 syntax

package inventory; // Namespace for the generated code

// Defines the interface for the Inventory service
service InventoryService {
  rpc GetInventoryByLocation (LocationRequest) returns (InventoryList) {} // Fetch inventory items for a given location
  rpc CheckStock (StockRequest) returns (StockResponse) {} // Check stock availability for a specific SKU
}

// Request message containing a location
message LocationRequest {
  string location = 1; // Field 1: The location name
}

// Request message for checking stock
message StockRequest {
  string sku = 1; // Field 1: The SKU of the product
}

// Response message detailing stock availability
message StockResponse {
  bool isAvailable = 1; // Field 1: True if stock is available
  int32 quantity = 2; // Field 2: The current quantity in stock
}

// Represents a single inventory item
message InventoryItem {
  int32 id = 1; // Field 1: Inventory record ID
  string sku = 2; // Field 2: Product SKU
  int32 quantity = 3; // Field 3: Quantity available
  string location = 4; // Field 4: Location where the item is stored
  int32 productId = 5; // Field 5: Associated Product ID
}

// Response message containing a list of inventory items
message InventoryList {
  repeated InventoryItem items = 1; // Field 1: An array of InventoryItem messages
}
```

---

## 2. Product Service (gRPC Server on port 5001)

### `product-service/src/main.ts`

```typescript
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { join } from 'path';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.GRPC,
      options: {
        package: 'product',
        protoPath: join(__dirname, '../../proto/product.proto'),
        url: '0.0.0.0:5001',
      },
    },
  );
  await app.listen();
  console.log('Product Service is running on port 5001 (gRPC)');
}
bootstrap();
```

### `product-service/src/product/product.controller.ts`

The controller uses `@GrpcMethod` instead of HTTP decorators:

```typescript
import { Controller } from '@nestjs/common';
import { GrpcMethod } from '@nestjs/microservices';
import { ProductService } from './product.service';

@Controller()
export class ProductController {
  constructor(private readonly productService: ProductService) {}

  @GrpcMethod('ProductService', 'FindOne')
  async findOne(data: { id: number }) {
    return this.productService.findOne(data.id);
  }

  @GrpcMethod('ProductService', 'FindMany')
  async findMany(data: { ids: number[] }) {
    const products = await this.productService.findByIds(data.ids);
    return { products };
  }

  @GrpcMethod('ProductService', 'FindAll')
  async findAll() {
    const products = await this.productService.findAll();
    return { products };
  }
}
```

### `product-service/src/product/product.service.ts`

Uses the same Repository pattern from the boilerplate:

```typescript
import { Injectable } from '@nestjs/common';
import { ProductRepository } from './infrastructure/persistence/product.repository';

@Injectable()
export class ProductService {
  constructor(private readonly productRepository: ProductRepository) {}

  async findOne(id: number) {
    return this.productRepository.findById(id);
  }

  async findByIds(ids: number[]) {
    return this.productRepository.findByIds(ids);
  }

  async findAll() {
    return this.productRepository.findAllWithPagination({
      paginationOptions: { page: 1, limit: 100 },
    });
  }
}
```

---

## 3. Inventory Service (gRPC Server on port 5002)

### `inventory-service/src/main.ts`

```typescript
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { join } from 'path';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.GRPC,
      options: {
        package: 'inventory',
        protoPath: join(__dirname, '../../proto/inventory.proto'),
        url: '0.0.0.0:5002',
      },
    },
  );
  await app.listen();
  console.log('Inventory Service is running on port 5002 (gRPC)');
}
bootstrap();
```

### `inventory-service/src/inventory/inventory.controller.ts`

```typescript
import { Controller } from '@nestjs/common';
import { GrpcMethod } from '@nestjs/microservices';
import { InventoryService } from './inventory.service';

@Controller()
export class InventoryController {
  constructor(private readonly inventoryService: InventoryService) {}

  @GrpcMethod('InventoryService', 'GetInventoryByLocation')
  async getByLocation(data: { location: string }) {
    const items = await this.inventoryService.findByLocation(data.location);
    return { items };
  }

  @GrpcMethod('InventoryService', 'CheckStock')
  async checkStock(data: { sku: string }) {
    return this.inventoryService.checkStock(data.sku);
  }
}
```

### `inventory-service/src/inventory/inventory.service.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { InventoryRepository } from './infrastructure/persistence/inventory.repository';

@Injectable()
export class InventoryService {
  constructor(private readonly inventoryRepository: InventoryRepository) {}

  async findByLocation(location: string) {
    return this.inventoryRepository.findByLocation(location);
  }

  async checkStock(sku: string) {
    const item = await this.inventoryRepository.findBySku(sku);
    return {
      isAvailable: item ? item.quantity > 0 : false,
      quantity: item?.quantity ?? 0,
    };
  }
}
```

---

## 4. API Gateway (HTTP Server on port 3000)

The gateway is the only service exposed to the outside. It receives HTTP requests and forwards them to the appropriate microservice via gRPC.

### `gateway/src/main.ts`

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Microservices API Gateway')
    .setDescription('Gateway for Product & Inventory services')
    .setVersion('1.0')
    .build();
  SwaggerModule.setup('docs', app, SwaggerModule.createDocument(app, config));

  await app.listen(3000);
  console.log('API Gateway is running on http://localhost:3000');
}
bootstrap();
```

### `gateway/src/app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { ProductModule } from './product/product.module';
import { InventoryModule } from './inventory/inventory.module';

@Module({
  imports: [ProductModule, InventoryModule],
})
export class AppModule {}
```

### `gateway/src/product/product.module.ts`

Register the gRPC client to connect to the Product Service:

```typescript
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { join } from 'path';
import { ProductController } from './product.controller';
import { ProductService } from './product.service';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'PRODUCT_PACKAGE',
        transport: Transport.GRPC,
        options: {
          package: 'product',
          protoPath: join(__dirname, '../../../proto/product.proto'),
          url: 'localhost:5001',
        },
      },
    ]),
  ],
  controllers: [ProductController],
  providers: [ProductService],
  exports: [ProductService],
})
export class ProductModule {}
```

### `gateway/src/inventory/inventory.module.ts`

Register gRPC clients for **both** Inventory and Product services, since the "get available products" feature needs to call both:

```typescript
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { join } from 'path';
import { InventoryController } from './inventory.controller';
import { InventoryService } from './inventory.service';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'INVENTORY_PACKAGE',
        transport: Transport.GRPC,
        options: {
          package: 'inventory',
          protoPath: join(__dirname, '../../../proto/inventory.proto'),
          url: 'localhost:5002',
        },
      },
      {
        name: 'PRODUCT_PACKAGE',
        transport: Transport.GRPC,
        options: {
          package: 'product',
          protoPath: join(__dirname, '../../../proto/product.proto'),
          url: 'localhost:5001',
        },
      },
    ]),
  ],
  controllers: [InventoryController],
  providers: [InventoryService],
})
export class InventoryModule {}
```

### `gateway/src/inventory/inventory.service.ts`

This is where the **cross-service orchestration** happens — calling Inventory then Product via gRPC:

```typescript
import { Injectable, Inject, OnModuleInit } from '@nestjs/common';
import { ClientGrpc } from '@nestjs/microservices';
import { lastValueFrom } from 'rxjs';

interface InventoryGrpcService {
  getInventoryByLocation(data: { location: string }): any;
  checkStock(data: { sku: string }): any;
}

interface ProductGrpcService {
  findOne(data: { id: number }): any;
  findMany(data: { ids: number[] }): any;
}

@Injectable()
export class InventoryService implements OnModuleInit {
  private inventoryService: InventoryGrpcService;
  private productService: ProductGrpcService;

  constructor(
    @Inject('INVENTORY_PACKAGE') private inventoryClient: ClientGrpc,
    @Inject('PRODUCT_PACKAGE') private productClient: ClientGrpc,
  ) {}

  onModuleInit() {
    this.inventoryService =
      this.inventoryClient.getService<InventoryGrpcService>('InventoryService');
    this.productService =
      this.productClient.getService<ProductGrpcService>('ProductService');
  }

  /**
   * KEY FEATURE: Get products available in a specific inventory location.
   *
   * Flow:
   * 1. Call Inventory Service → get items at location
   * 2. Extract productIds from inventory items
   * 3. Call Product Service → get product details for those IDs
   * 4. Merge inventory quantities with product details
   */
  async getAvailableProducts(location: string) {
    // Step 1: Get inventory items for this location
    const inventoryResponse = await lastValueFrom(
      this.inventoryService.getInventoryByLocation({ location }),
    );
    const items = inventoryResponse.items || [];

    // Filter only items with stock > 0
    const availableItems = items.filter((item) => item.quantity > 0);
    if (availableItems.length === 0) return [];

    // Step 2: Get product details for all available items
    const productIds = availableItems.map((item) => item.productId);
    const productResponse = await lastValueFrom(
      this.productService.findMany({ ids: productIds }),
    );
    const products = productResponse.products || [];

    // Step 3: Merge product details with inventory quantities
    const productMap = new Map(products.map((p) => [p.id, p]));

    return availableItems.map((item) => {
      const product = productMap.get(item.productId);
      return {
        inventoryId: item.id,
        sku: item.sku,
        quantityInStock: item.quantity,
        location: item.location,
        product: product
          ? { id: product.id, name: product.name, price: product.price }
          : null,
      };
    });
  }

  async checkStock(sku: string) {
    return lastValueFrom(this.inventoryService.checkStock({ sku }));
  }
}
```

### `gateway/src/inventory/inventory.controller.ts`

```typescript
import {
  Controller,
  Get,
  Param,
  Query,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';
import { ApiTags, ApiOperation } from '@nestjs/swagger';
import { InventoryService } from './inventory.service';

@ApiTags('Inventory')
@Controller({ path: 'inventory', version: '1' })
export class InventoryController {
  constructor(private readonly inventoryService: InventoryService) {}

  @Get('location/:location/products')
  @HttpCode(HttpStatus.OK)
  @ApiOperation({
    summary: 'Get available products in a specific inventory location',
  })
  getAvailableProducts(@Param('location') location: string) {
    return this.inventoryService.getAvailableProducts(location);
  }

  @Get('check-stock')
  @HttpCode(HttpStatus.OK)
  @ApiOperation({ summary: 'Check stock availability by SKU' })
  checkStock(@Query('sku') sku: string) {
    return this.inventoryService.checkStock(sku);
  }
}
```

---

## 5. How the Request Flows

```
GET /v1/inventory/location/warehouse-A/products

  1. Client → Gateway (HTTP)
  2. Gateway → Inventory Service (gRPC: GetInventoryByLocation)
  3. Inventory Service returns items with productIds
  4. Gateway → Product Service (gRPC: FindMany with productIds)
  5. Product Service returns product details
  6. Gateway merges data and responds to Client (HTTP/JSON)
```

**Example Response:**

```json
[
  {
    "inventoryId": 1,
    "sku": "LAPTOP-X200",
    "quantityInStock": 25,
    "location": "warehouse-A",
    "product": {
      "id": 10,
      "name": "Laptop X200",
      "price": 999.99
    }
  },
  {
    "inventoryId": 2,
    "sku": "MOUSE-M100",
    "quantityInStock": 150,
    "location": "warehouse-A",
    "product": {
      "id": 22,
      "name": "Wireless Mouse M100",
      "price": 29.99
    }
  }
]
```

---

## 6. Running the Services

Install dependencies in each project, then start them in order:

```bash
# Terminal 1 — Product Service
cd product-service
npm install
npm run start:dev

# Terminal 2 — Inventory Service
cd inventory-service
npm install
npm run start:dev

# Terminal 3 — API Gateway
cd gateway
npm install
npm run start:dev
```

Visit `http://localhost:3000/docs` for the Swagger UI.

---

## 7. Docker Compose (All Services)

For production or local testing, use Docker Compose to run everything together:

```yaml
# docker-compose.yml
version: '3.8'

services:
  product-service:
    build: ./product-service
    ports:
      - '5001:5001'
    environment:
      - DATABASE_URL=postgres://user:pass@product-db:5432/products
    depends_on:
      - product-db

  inventory-service:
    build: ./inventory-service
    ports:
      - '5002:5002'
    environment:
      - DATABASE_URL=postgres://user:pass@inventory-db:5432/inventory
    depends_on:
      - inventory-db

  gateway:
    build: ./gateway
    ports:
      - '3000:3000'
    environment:
      - PRODUCT_SERVICE_URL=product-service:5001
      - INVENTORY_SERVICE_URL=inventory-service:5002
    depends_on:
      - product-service
      - inventory-service

  product-db:
    image: postgres:16
    environment:
      POSTGRES_DB: products
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass

  inventory-db:
    image: postgres:16
    environment:
      POSTGRES_DB: inventory
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
```

---

## 8. Key Takeaways

| Concept           | Description                                                           |
| :---------------- | :-------------------------------------------------------------------- |
| **API Gateway**   | Single HTTP entry point; clients never talk to microservices directly |
| **gRPC**          | High-performance binary protocol for inter-service communication      |
| **Proto files**   | Shared contracts that define service methods and message shapes       |
| **@GrpcMethod**   | NestJS decorator to expose a method as a gRPC handler                 |
| **ClientsModule** | NestJS module to register gRPC clients for calling remote services    |
| **OnModuleInit**  | Lifecycle hook to initialize gRPC service stubs                       |
| **lastValueFrom** | Converts RxJS Observable (from gRPC) to a Promise                     |
| **Orchestration** | Gateway combines data from multiple services before responding        |
