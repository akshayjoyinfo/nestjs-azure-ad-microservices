
// server microservice

import { Body, Controller, Get, Param, Post, UseGuards } from '@nestjs/common';
import { AppService } from './app.service';
import { AzureADGuard } from './azure-ad.guard';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }

  @Get('/item/:id')
  //@UseGuards(AzureADGuard)
  getById(@Param('id') id: number) {
    return this.appService.getItemById(id);
  }

  @Post('/create')
  @UseGuards(AzureADGuard)
  create(@Body() createItemDto) {
    return this.appService.createItem(createItemDto);
  }
}




import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { PassportModule } from '@nestjs/passport';
import { AzureADStrategy } from './azure-ad.guard';

@Module({
  imports: [
    PassportModule,
    ClientsModule.register([{ name: 'ITEM_MICROSERVICE', transport: Transport.TCP, options :{
      host: '127.0.0.1',
      port: 8977
    } }])
  ],
  controllers: [AppController],
  providers: [AppService , AzureADStrategy],
})
export class AppModule {}


import { Inject, Injectable } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';

@Injectable()
export class AppService {

  constructor(
    @Inject('ITEM_MICROSERVICE') private readonly client: ClientProxy
  ) {}
  
  getHello(): string {
    return 'Hello World!';
  }

  createItem(createItemDto) {
    return this.client.send({ role: 'item', cmd: 'create' }, createItemDto);
  }

  getItemById(id: number) {
    return this.client.send({ role: 'item', cmd: 'get-by-id' }, id); 
  }
}


import { Injectable } from '@nestjs/common';
import { PassportStrategy, AuthGuard } from '@nestjs/passport';
import { BearerStrategy } from 'passport-azure-ad';

const clientID = '<ClientID>';
const tenantID = '<TenantID>';

/**
 * Extracts ID token from header and validates it.
 */
@Injectable()
export class AzureADStrategy extends PassportStrategy(
  BearerStrategy,
  'azure-ad',
) {
  constructor() {
    super({
      identityMetadata: `https://login.microsoftonline.com/${tenantID}/v2.0/.well-known/openid-configuration`,
      clientID,
    });
  }

  async validate(data) {
    return data;
  }
}

export const AzureADGuard = AuthGuard('azure-ad');



import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();


// client microservice
import { Controller, Get } from '@nestjs/common';
import { MessagePattern } from '@nestjs/microservices';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }

  @MessagePattern({ role: 'item', cmd: 'create' })
  createItem(itemDto) {
    console.log('Create Item ' + JSON.stringify(itemDto));
    return this.appService.createItem(itemDto)
  }
  @MessagePattern({ role: 'item', cmd: 'get-by-id' })
  getItemById(id: number) {
    return this.appService.getItemById(id);
  }
  
}


import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ItemEntity } from './item.entity';
import { ItemRepository } from './item.repository';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: 'localhost',
      port: 5432,
      username: 'postgres',
      password: 'Password2020',
      database: 'enm',
      synchronize: true,
      autoLoadEntities: true,
    }),
    TypeOrmModule.forFeature([ItemRepository, ItemEntity])
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}


import { Inject, Injectable } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import { ItemEntity } from './item.entity';
import { ItemRepository } from './item.repository';

@Injectable()
export class AppService {
  constructor(
    private readonly itemRepository: ItemRepository,
  ) {}

  getHello(): string {
    return 'Hello World!';
  }

  createItem(itemDto) {
    const item = new ItemEntity();
    item.name = itemDto.name;
    return this.itemRepository.save(item);
  }
  getItemById(id) {
    return this.itemRepository.findOne(id);
  }
}

import { BaseEntity, Column, Entity, PrimaryGeneratedColumn } from "typeorm";

@Entity()
export class ItemEntity extends BaseEntity {
    @PrimaryGeneratedColumn()
    id: number;
    @Column()
    name: string;
}

import { EntityRepository, Repository } from "typeorm";
import { ItemEntity } from "./item.entity";

@EntityRepository(ItemEntity)
export class ItemRepository extends Repository<ItemEntity> {}



import { Logger } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

const logger = new Logger('Microservice');

async function bootstrap() {

  const app = await NestFactory.createMicroservice(AppModule, {
    transport: Transport.TCP,
    options: {
      host: '127.0.0.1',
      port: 8977
    }
  });
  
  await app.listen(() => {
    logger.log('Microservice is listening');

  });
}
bootstrap();
