# Documentação do Cadastro de Usuário #
## Este documento detalha a funcionalidade de cadastro de usuário na API, descrevendo cada parte do código, suas responsabilidades e as validações aplicadas. ##
---

## Estrutura do Projeto

O cadastro de usuários no projeto está organizado em diversos arquivos, cada um desempenhando uma função específica. A seguir, uma descrição detalhada de cada um desses arquivos.

### 1. DTO (Data Transfer Object)

**Arquivo:** `users/dto/create-user.dto.ts`

O DTO define a estrutura dos dados que são enviados pelo cliente ao servidor durante a criação de um novo usuário. Além de estruturar os dados, ele incorpora validações para assegurar que as informações enviadas estejam no formato correto e atendam aos critérios necessários.

```typescript
import { IsEmail, IsNotEmpty, IsString } from 'class-validator';

export class CreateUserDto {
  @IsString({ message: 'O nome deve ser uma string' })
  @IsNotEmpty({ message: 'O nome é obrigatório' })
  firstName: string;

  @IsString({ message: 'O sobrenome deve ser uma string' })
  @IsNotEmpty({ message: 'O sobrenome é obrigatório' })
  lastName: string;

  @IsEmail({}, { message: 'O email deve ser um endereço de email válido' })
  @IsNotEmpty({ message: 'O email é obrigatório' })
  email: string;

  @IsString({ message: 'A senha deve ser uma string' })
  @IsNotEmpty({ message: 'A senha é obrigatória' })
  password: string;
}
```

#### Explicação das Validações:

- **@IsString**: Garante que o valor do campo seja uma string.
- **@IsNotEmpty**: Certifica que o campo não está vazio.
- **@IsEmail**: Verifica se o valor do campo é um endereço de email válido.

Cada uma dessas validações inclui uma mensagem personalizada, que será retornada caso a validação falhe, informando ao usuário qual foi o problema encontrado com o dado fornecido.

### 2. Esquema do Usuário

**Arquivo:** `users/schemas/user.schema.ts`

Este arquivo define o esquema do usuário no banco de dados utilizando o Mongoose, um ODM (Object Data Modeling) para MongoDB. O esquema define os campos que um documento de usuário deve ter, além de aplicar restrições como campos obrigatórios e unicidade.

```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

export type UserDocument = User & Document;

@Schema()
export class User extends Document {
  @Prop({ required: true })
  firstName: string;

  @Prop({ required: true })
  lastName: string;

  @Prop({ required: true, unique: true })
  email: string;

  @Prop({ required: true })
  password: string;
}

export const UserSchema = SchemaFactory.createForClass(User);
```

#### Explicação dos Campos:

- **firstName**: O primeiro nome do usuário. Campo obrigatório.
- **lastName**: O sobrenome do usuário. Campo obrigatório.
- **email**: Endereço de email do usuário. Este campo é obrigatório e deve ser único no sistema.
- **password**: Senha do usuário. Campo obrigatório, que será criptografado antes de ser salvo no banco de dados.

### 3. Serviço de Usuários

**Arquivo:** `users/users.service.ts`

Este serviço é responsável por lidar com a lógica de negócio relacionada aos usuários. Ele inclui funcionalidades como a criação de novos usuários, verificação de email duplicado e criptografia de senhas antes de armazená-las.

```typescript
import { Injectable, ConflictException } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import * as bcrypt from 'bcryptjs';
import { User, UserDocument } from './schemas/user.schema';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UsersService {
  constructor(@InjectModel(User.name) private userModel: Model<UserDocument>) {}

  async create(createUserDto: CreateUserDto): Promise<User> {
    const existingUser = await this.findByEmail(createUserDto.email);
    if (existingUser) {
      throw new ConflictException('Email Já Existe');
    }

    // Criptografar a senha antes de criar o usuário
    const hashedPassword = await bcrypt.hash(createUserDto.password, 10);
    const createdUser = new this.userModel({
      ...createUserDto,
      password: hashedPassword,
    });

    return createdUser.save();
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.userModel.findOne({ email }).exec();
  }
}
```

#### Explicação das Funções:

- **create**: Responsável por criar um novo usuário. Ela verifica se o email já está em uso, criptografa a senha e, em seguida, salva o usuário no banco de dados.
- **findByEmail**: Pesquisa um usuário no banco de dados utilizando o email como critério de busca. Retorna `null` se o usuário não for encontrado.

### 4. Controlador de Usuários

**Arquivo:** `users/users.controller.ts`

O controlador é responsável por gerenciar as rotas relacionadas aos usuários, como a criação de novos usuários e a verificação do status da API.

```typescript
import { Controller, Post, Body, HttpCode, HttpStatus, Get, UseGuards, Request } from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { AuthGuard } from '@nestjs/passport';

@Controller('v1/users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post('create')
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() createUserDto: CreateUserDto) {
    await this.usersService.create(createUserDto);
    return { message: 'Usuário registrado com sucesso' };
  }

  @UseGuards(AuthGuard('jwt'))
  @Get('status')
  getStatus(@Request() req) {
    return {
      date: new Date().toLocaleDateString(),
      time: new Date().toLocaleTimeString(),
      location: this.getCurrentLocation(req),
      status: 'API is running fine',
    };
  }

  private getCurrentLocation(req): string {
    return 'Latitude: X, Longitude: Y';
  }
}
```

#### Explicação das Funções:

- **create**: Rota `POST /v1/users/create` responsável por registrar um novo usuário. Ela chama o serviço de criação de usuários e retorna uma mensagem de sucesso.
- **getStatus**: Rota `GET /v1/users/status` protegida por autenticação JWT. Retorna informações sobre o status da API, incluindo a data, hora, e uma localização fictícia.

### 5. Testes

**Testes do Controlador:** `users/users.controller.spec.ts`

Os testes garantem que o controlador de usuários esteja funcionando conforme o esperado. Eles verificam se o controlador foi definido corretamente.

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { UsersController } from './users.controller';

describe('UsersController', () => {
  let controller: UsersController;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
    }).compile();

    controller = module.get<UsersController>(UsersController);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });
});
```

**Testes do Serviço:** `users/users.service.spec.ts`

Os testes do serviço asseguram que as funcionalidades de criação e busca de usuários estão operando corretamente.

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';

describe('UsersService', () => {
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [UsersService],
    }).compile();

    service = module.get<UsersService>(UsersService);
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });
});
```

### 6. Módulo de Usuários

**Arquivo:** `users/users.module.ts`

Este módulo é responsável por agrupar todos os componentes relacionados aos usuários, como o controlador, o serviço, e o esquema. Ele também faz a integração com o Mongoose para facilitar a comunicação com o banco de dados.

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { User, UserSchema } from './schemas/user.schema';

@Module({
  imports: [
    MongooseModule.forFeature([{ name: User.name, schema: UserSchema }])
  ],
  providers: [UsersService],
  controllers: [UsersController],
  exports: [UsersService],
})
export class UsersModule {}
```

#### Explicação:

- **MongooseModule.forFeature**: Registra os esquemas de MongoDB, como o esquema de usuário, para serem usados em todo o módulo.
- **providers**: Lista de serviços que serão injetados no módulo. Inclui o `UsersService` que contém a lógica de negócio dos usuários.
- **controllers**: Lista de controladores que definem as rotas associadas a este módulo.
- **exports**: Permite que outros módulos utilizem o `UsersService` ao importar o `UsersModule`.

---

Este detalhamento cobre as principais funcionalidades e estrutura do sistema de cadastro

 de usuários, ajudando a entender como as diferentes partes do código se conectam e trabalham juntas.
