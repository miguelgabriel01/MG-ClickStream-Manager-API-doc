## Autenticação de Usuário

A autenticação de usuário no projeto é gerenciada através de uma série de arquivos que implementam o login, validação de credenciais, geração de tokens JWT e a estratégia para proteger rotas. Abaixo, você encontra uma descrição detalhada de cada arquivo, suas funções, e como elas se conectam.

### Estrutura dos Arquivos

1. **`auth.controller.ts`**
   - **Caminho:** `src/auth/auth.controller.ts`
   - **Função:** Este controlador gerencia as rotas relacionadas à autenticação de usuários.
   - **Conteúdo:**
     ```typescript
     import { Controller, Post, Body } from '@nestjs/common';
     import { AuthService } from './auth.service';
     import { LoginDto } from './dto/login.dto';

     @Controller('v1/users')
     export class AuthController {
       constructor(private authService: AuthService) {}

       @Post('auth')
       async login(@Body() loginDto: LoginDto) {
         return this.authService.login(loginDto);
       }
     }
     ```
   - **Explicação:**
     - `AuthController`: Define o controlador para autenticação.
     - `@Post('auth')`: Rota HTTP POST para autenticação do usuário.
     - `login()`: Recebe as credenciais de login e chama o serviço de autenticação.

2. **`auth.module.ts`**
   - **Caminho:** `src/auth/auth.module.ts`
   - **Função:** Define o módulo de autenticação, registrando os serviços, controladores e módulos necessários.
   - **Conteúdo:**
     ```typescript
     import { Module } from '@nestjs/common';
     import { JwtModule } from '@nestjs/jwt';
     import { PassportModule } from '@nestjs/passport';
     import { AuthService } from './auth.service';
     import { AuthController } from './auth.controller';
     import { UsersModule } from '../users/users.module';
     import { JwtStrategy } from './jwt.strategy';

     @Module({
       imports: [
         UsersModule,
         PassportModule,
         JwtModule.register({
           secret: 'secretKey', // Substitua por uma chave secreta mais segura em produção
           signOptions: { expiresIn: '60m' },
         }),
       ],
       providers: [AuthService, JwtStrategy],
       controllers: [AuthController],
     })
     export class AuthModule {}
     ```
   - **Explicação:**
     - `JwtModule.register`: Configura o módulo JWT com uma chave secreta e tempo de expiração.
     - `AuthService`, `JwtStrategy`: Provedores que lidam com autenticação e estratégias JWT.
     - `UsersModule`: Importa o módulo de usuários para validar as credenciais.

3. **`auth.service.ts`**
   - **Caminho:** `src/auth/auth.service.ts`
   - **Função:** Serviço que lida com a lógica de autenticação, como validação de credenciais e geração de tokens JWT.
   - **Conteúdo:**
     ```typescript
     import { Injectable, UnauthorizedException } from '@nestjs/common';
     import { JwtService } from '@nestjs/jwt';
     import * as bcrypt from 'bcryptjs';
     import { UsersService } from '../users/users.service';
     import { LoginDto } from './dto/login.dto';

     @Injectable()
     export class AuthService {
       constructor(
         private usersService: UsersService,
         private jwtService: JwtService
       ) {}

       async validateUser(email: string, password: string): Promise<any> {
         console.log(`Validando usuário: ${email}`);
         
         const user = await this.usersService.findByEmail(email);
         if (!user) {
           console.log('Usuário não encontrado');
           throw new UnauthorizedException('Email ou senha invalidos');
         }

         const passwordMatch = await bcrypt.compare(password, user.password);
         if (passwordMatch) {
           console.log('Senha validada com sucesso');
           return { id: user.id, email: user.email, firstName: user.firstName, lastName: user.lastName };
         } else {
           console.log('Falha na validação da senha');
           throw new UnauthorizedException('Email ou senha invalidos');
         }
       }

       async login(loginDto: LoginDto) {
         const user = await this.validateUser(loginDto.email, loginDto.password);
         const payload = { email: user.email, sub: user.id };
         return {
           access_token: this.jwtService.sign(payload),
           user: {
             id: user.id,
             email: user.email,
             firstName: user.firstName,
             lastName: user.lastName,
           },
         };
       }
     }
     ```
   - **Explicação:**
     - `validateUser()`: Valida as credenciais do usuário. Verifica se o email existe e se a senha é correta.
     - `login()`: Gera o token JWT após validação e retorna as informações do usuário juntamente com o token.

4. **`jwt.strategy.ts`**
   - **Caminho:** `src/auth/jwt.strategy.ts`
   - **Função:** Implementa a estratégia JWT para proteger rotas, extraindo e validando o token JWT.
   - **Conteúdo:**
     ```typescript
     import { Injectable } from '@nestjs/common';
     import { PassportStrategy } from '@nestjs/passport';
     import { ExtractJwt, Strategy } from 'passport-jwt';

     @Injectable()
     export class JwtStrategy extends PassportStrategy(Strategy) {
       constructor() {
         super({
           jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
           ignoreExpiration: false,
           secretOrKey: 'secretKey', // A mesma chave usada para assinar os tokens
         });
       }

       async validate(payload: any) {
         return { id: payload.sub, email: payload.email };
       }
     }
     ```
   - **Explicação:**
     - `JwtStrategy`: Extende a estratégia Passport para JWT.
     - `jwtFromRequest`: Extrai o token JWT do cabeçalho de autorização.
     - `validate()`: Valida o payload do token JWT, retornando os dados do usuário.

5. **`login.dto.ts`**
   - **Caminho:** `src/auth/dto/login.dto.ts`
   - **Função:** Define a estrutura e as validações dos dados de login fornecidos pelo usuário.
   - **Conteúdo:**
     ```typescript
     import { IsEmail, IsNotEmpty, IsString } from 'class-validator';

     export class LoginDto {
       @IsEmail({}, { message: 'O email deve ser um endereço de email válido' })
       @IsNotEmpty({ message: 'O email é obrigatório' })
       email: string;

       @IsString({ message: 'A senha deve ser uma string' })
       @IsNotEmpty({ message: 'A senha é obrigatória' })
       password: string;
     }
     ```
   - **Explicação:**
     - `@IsEmail`: Valida que o campo email é um endereço de email válido.
     - `@IsNotEmpty`: Garante que os campos email e senha não estejam vazios.
     - `LoginDto`: Define o formato dos dados de login esperados pelo servidor.

---

Essa documentação detalhada fornece uma visão clara de como a autenticação de usuário é implementada no projeto, facilitando o entendimento e a manutenção do código.
