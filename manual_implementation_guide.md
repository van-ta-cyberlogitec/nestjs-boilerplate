# Manual Feature Implementation Guide (PostgreSQL)

This guide walks you through the manual steps to implement a new feature (e.g., **Inventory**) using PostgreSQL in this NestJS boilerplate. This is useful if you want to understand the underlying architecture or if you need to deviate from the standard scaffolding provided by Hygen.

## Overview
The boilerplate follows a Clean Architecture approach with three main layers for each feature:
1.  **Domain Layer**: Core business logic and interfaces (POJO/Typescript classes).
2.  **Infrastructure Layer**: Database-specific implementations (TypeORM entities, Mappers, Repositories).
3.  **Application Layer**: Entry points (Controllers, Services, DTOs).

---

## 1. Define the Domain Layer
The domain layer contains the core business entity.

### Create `src/inventory/domain/inventory.ts`
```typescript
import { ApiProperty } from '@nestjs/swagger';

export class Inventory {
  @ApiProperty({ type: Number })
  id: number | string;

  @ApiProperty()
  sku: string;

  @ApiProperty()
  quantity: number;

  @ApiProperty()
  createdAt: Date;

  @ApiProperty()
  updatedAt: Date;
}
```

---

## 2. Define the Infrastructure Layer (Persistence)
This layer handles the actual database communication using TypeORM.

### A. Create the TypeORM Entity
`src/inventory/infrastructure/persistence/relational/entities/inventory.entity.ts`
```typescript
import {
  Column,
  CreateDateColumn,
  Entity,
  PrimaryGeneratedColumn,
  UpdateDateColumn,
} from 'typeorm';
import { EntityRelationalHelper } from 'src/utils/relational-entity-helper';

@Entity({ name: 'inventory' })
export class InventoryEntity extends EntityRelationalHelper {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ type: String })
  sku: string;

  @Column({ type: Number })
  quantity: number;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

### B. Create the Mapper
Mappers ensure that the application logic only works with the Domain Entity, keeping it decoupled from TypeORM.
`src/inventory/infrastructure/persistence/relational/mappers/inventory.mapper.ts`
```typescript
import { Inventory } from 'src/inventory/domain/inventory';
import { InventoryEntity } from '../entities/inventory.entity';

export class InventoryMapper {
  static toDomain(raw: InventoryEntity): Inventory {
    const domainEntity = new Inventory();
    domainEntity.id = raw.id;
    domainEntity.sku = raw.sku;
    domainEntity.quantity = raw.quantity;
    domainEntity.createdAt = raw.createdAt;
    domainEntity.updatedAt = raw.updatedAt;
    return domainEntity;
  }

  static toPersistence(domainEntity: Inventory): InventoryEntity {
    const persistenceEntity = new InventoryEntity();
    if (domainEntity.id && typeof domainEntity.id === 'number') {
      persistenceEntity.id = domainEntity.id;
    }
    persistenceEntity.sku = domainEntity.sku;
    persistenceEntity.quantity = domainEntity.quantity;
    return persistenceEntity;
  }
}
```

### C. Define the Repository Interface
The interface belongs to the domain (but usually stored near persistence for convenience in this boilerplate).
`src/inventory/infrastructure/persistence/inventory.repository.ts`
```typescript
import { Inventory } from '../domain/inventory';
import { NullableType } from 'src/utils/types/nullable.type';
import { IPaginationOptions } from 'src/utils/types/pagination-options';

export abstract class InventoryRepository {
  abstract create(
    data: Omit<Inventory, 'id' | 'createdAt' | 'updatedAt'>,
  ): Promise<Inventory>;

  abstract findAllWithPagination({
    paginationOptions,
  }: {
    paginationOptions: IPaginationOptions;
  }): Promise<Inventory[]>;

  abstract findById(id: Inventory['id']): Promise<NullableType<Inventory>>;

  abstract update(
    id: Inventory['id'],
    payload: Partial<Inventory>,
  ): Promise<NullableType<Inventory>>;

  abstract remove(id: Inventory['id']): Promise<void>;
}
```

### D. Implement the Repository (Relational)
`src/inventory/infrastructure/persistence/relational/repositories/inventory.repository.ts`
```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { InventoryEntity } from '../entities/inventory.entity';
import { InventoryRepository } from '../../inventory.repository';
import { Inventory } from 'src/inventory/domain/inventory';
import { InventoryMapper } from '../mappers/inventory.mapper';
import { IPaginationOptions } from 'src/utils/types/pagination-options';

@Injectable()
export class InventoryRelationalRepository implements InventoryRepository {
  constructor(
    @InjectRepository(InventoryEntity)
    private readonly repository: Repository<InventoryEntity>,
  ) {}

  async create(data: Inventory): Promise<Inventory> {
    const persistenceModel = InventoryMapper.toPersistence(data);
    const newEntity = await this.repository.save(
      this.repository.create(persistenceModel),
    );
    return InventoryMapper.toDomain(newEntity);
  }

  async findAllWithPagination({
    paginationOptions,
  }: {
    paginationOptions: IPaginationOptions;
  }): Promise<Inventory[]> {
    const entities = await this.repository.find({
      skip: (paginationOptions.page - 1) * paginationOptions.limit,
      take: paginationOptions.limit,
    });
    return entities.map((item) => InventoryMapper.toDomain(item));
  }

  async findById(id: Inventory['id']): Promise<Inventory | null> {
    const entity = await this.repository.findOne({ where: { id: Number(id) } });
    return entity ? InventoryMapper.toDomain(entity) : null;
  }

  async update(id: Inventory['id'], payload: Partial<Inventory>): Promise<Inventory | null> {
    const entity = await this.repository.findOne({ where: { id: Number(id) } });
    if (!entity) return null;

    const updatedEntity = await this.repository.save(
      this.repository.create(
        InventoryMapper.toPersistence({
          ...InventoryMapper.toDomain(entity),
          ...payload,
        }),
      ),
    );
    return InventoryMapper.toDomain(updatedEntity);
  }

  async remove(id: Inventory['id']): Promise<void> {
    await this.repository.delete(id);
  }
}
```

### E. Set up the Persistence Module
`src/inventory/infrastructure/persistence/relational/relational-persistence.module.ts`
```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { InventoryEntity } from './entities/inventory.entity';
import { InventoryRepository } from '../../inventory.repository';
import { InventoryRelationalRepository } from './repositories/inventory.repository';

@Module({
  imports: [TypeOrmModule.forFeature([InventoryEntity])],
  providers: [
    {
      provide: InventoryRepository,
      useClass: InventoryRelationalRepository,
    },
  ],
  exports: [InventoryRepository],
})
export class RelationalInventoryPersistenceModule {}
```

---

## 3. Define the Application Layer (Service and Controller)

### A. Create DTOs
`src/inventory/dto/create-inventory.dto.ts`
```typescript
import { ApiProperty } from '@nestjs/swagger';
import { IsNotEmpty, IsNumber, IsString } from 'class-validator';

export class CreateInventoryDto {
  @ApiProperty({ example: 'LAPTOP-001' })
  @IsNotEmpty()
  @IsString()
  sku: string;

  @ApiProperty({ example: 100 })
  @IsNotEmpty()
  @IsNumber()
  quantity: number;
}
```

### B. Create the Service
`src/inventory/inventory.service.ts`
```typescript
import { Injectable } from '@nestjs/common';
import { InventoryRepository } from './infrastructure/persistence/inventory.repository';
import { CreateInventoryDto } from './dto/create-inventory.dto';
import { Inventory } from './domain/inventory';
import { IPaginationOptions } from 'src/utils/types/pagination-options';

@Injectable()
export class InventoryService {
  constructor(private readonly inventoryRepository: InventoryRepository) {}

  create(createInventoryDto: CreateInventoryDto) {
    return this.inventoryRepository.create(createInventoryDto);
  }

  findAllWithPagination(paginationOptions: IPaginationOptions) {
    return this.inventoryRepository.findAllWithPagination({ paginationOptions });
  }

  findOne(id: Inventory['id']) {
    return this.inventoryRepository.findById(id);
  }

  update(id: Inventory['id'], updateInventoryDto: Partial<Inventory>) {
    return this.inventoryRepository.update(id, updateInventoryDto);
  }

  remove(id: Inventory['id']) {
    return this.inventoryRepository.remove(id);
  }
}
```

### C. Create the Controller
`src/inventory/inventory.controller.ts`
```typescript
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Delete,
  Query,
  HttpStatus,
  HttpCode,
} from '@nestjs/common';
import { InventoryService } from './inventory.service';
import { CreateInventoryDto } from './dto/create-inventory.dto';
import { ApiTags } from '@nestjs/swagger';

@ApiTags('Inventory')
@Controller({ path: 'inventory', version: '1' })
export class InventoryController {
  constructor(private readonly inventoryService: InventoryService) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createInventoryDto: CreateInventoryDto) {
    return this.inventoryService.create(createInventoryDto);
  }

  @Get()
  @HttpCode(HttpStatus.OK)
  findAll(@Query('page') page: number, @Query('limit') limit: number) {
    return this.inventoryService.findAllWithPagination({ page, limit });
  }

  @Get(':id')
  @HttpCode(HttpStatus.OK)
  findOne(@Param('id') id: string) {
    return this.inventoryService.findOne(id);
  }

  @Patch(':id')
  @HttpCode(HttpStatus.OK)
  update(@Param('id') id: string, @Body() updateInventoryDto: any) {
    return this.inventoryService.update(id, updateInventoryDto);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id') id: string) {
    return this.inventoryService.remove(id);
  }
}
```

---

## 4. Assemble the Module
`src/inventory/inventory.module.ts`
```typescript
import { Module } from '@nestjs/common';
import { InventoryService } from './inventory.service';
import { InventoryController } from './inventory.controller';
import { RelationalInventoryPersistenceModule } from './infrastructure/persistence/relational/relational-persistence.module';

@Module({
  imports: [RelationalInventoryPersistenceModule],
  controllers: [InventoryController],
  providers: [InventoryService],
  exports: [InventoryService, RelationalInventoryPersistenceModule],
})
export class InventoryModule {}
```

---

## 5. Final Registration and Database Update

### A. Register in `AppModule`
Open `src/app.module.ts` and import `InventoryModule`.

### B. Generate and Run Migration
Since you manually created the `InventoryEntity`, you must tell the database to create the table.
```bash
npm run migration:generate -- src/database/migrations/CreateInventoryTable
npm run migration:run
```
