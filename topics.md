
## Gerenciamento de Tópicos no Kafka

A funcionalidade de gerenciamento de tópicos no Kafka permite que usuários autenticados criem, listem e consultem tópicos específicos. Essa funcionalidade é implementada por meio de controladores, serviços e esquemas que interagem com o Kafka e MongoDB. Abaixo estão os detalhes de cada componente envolvido.

### Estrutura dos Arquivos

1. **`create-topic.dto.ts`**
   - **Caminho:** `src/users/dto/create-topic.dto.ts`
   - **Função:** Define a estrutura dos dados enviados pelo usuário ao criar um novo tópico.
   - **Conteúdo:**
     ```typescript
     import { IsString } from 'class-validator';

     export class CreateTopicDto {
       @IsString()
       readonly topicName: string;
     }
     ```
   - **Explicação:**
     - `@IsString()`: Valida que o campo `topicName` seja uma string.
     - `CreateTopicDto`: Define o formato esperado do payload para a criação de tópicos.

2. **`topic.schema.ts`**
   - **Caminho:** `src/users/schemas/topic.schema.ts`
   - **Função:** Define o esquema do MongoDB para armazenar informações sobre os tópicos criados pelos usuários.
   - **Conteúdo:**
     ```typescript
     import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
     import { Document } from 'mongoose';

     @Schema()
     export class Topic extends Document {
       @Prop({ required: true })
       topicName: string;

       @Prop({ required: true })
       userId: string;
     }

     export const TopicSchema = SchemaFactory.createForClass(Topic);
     ```
   - **Explicação:**
     - `@Prop({ required: true })`: Define os campos `topicName` e `userId` como obrigatórios no esquema.
     - `Topic`: Classe que representa o documento do MongoDB para um tópico.
     - `TopicSchema`: Fábrica de esquemas Mongoose para a classe `Topic`.

3. **`topics.controller.ts`**
   - **Caminho:** `src/users/topics.controller.ts`
   - **Função:** Controlador responsável por gerenciar as rotas relacionadas a tópicos, incluindo criação e listagem.
   - **Conteúdo:**
     ```typescript
     import { Controller, Post, Get, UseGuards, Request, Body, Param } from '@nestjs/common';
     import { AuthGuard } from '@nestjs/passport';
     import { KafkaService } from './kafka.service';
     import { CreateTopicDto } from './dto/create-topic.dto';

     @Controller('users')
     @UseGuards(AuthGuard('jwt'))
     export class TopicsController {
       constructor(private readonly kafkaService: KafkaService) {}

       @Post('createTopics')
       async createTopic(@Body() createTopicDto: CreateTopicDto, @Request() req) {
         const userId = req.user['id'];
         const result = await this.kafkaService.createTopic(userId, createTopicDto);
         return result;
       }

       @Get('listTopics')
       async listTopics(@Request() req) {
         const userId = req.user['id'];
         const result = await this.kafkaService.listTopics(userId);
         return result;
       }

       @Get('listTopics/:idTopic')
       async getTopicById(@Request() req, @Param('idTopic') idTopic: string) {
         const userId = req.user['id'];
         const result = await this.kafkaService.getTopicById(userId, idTopic);
         return result;
       }
     }
     ```
   - **Explicação:**
     - `@Controller('users')`: Define o controlador para gerenciar rotas dentro do contexto de `users`.
     - `@UseGuards(AuthGuard('jwt'))`: Protege as rotas com autenticação JWT.
     - `createTopic()`: Endpoint para criação de um novo tópico no Kafka.
     - `listTopics()`: Endpoint para listar todos os tópicos criados pelo usuário autenticado.
     - `getTopicById()`: Endpoint para obter informações de um tópico específico por ID.

4. **`kafka.config.ts`**
   - **Caminho:** `src/kafka.config.ts`
   - **Função:** Configura o cliente Kafka e o produtor Kafka, incluindo o tratamento de eventos de prontidão e erro.
   - **Conteúdo:**
     ```typescript
     import { KafkaClient, Producer } from 'kafka-node';

     export const kafkaClient = new KafkaClient({ kafkaHost: 'localhost:29092' });

     export const producer = new Producer(kafkaClient);

     producer.on('ready', () => {
       console.log('Kafka Producer está pronto');
     });

     producer.on('error', (err) => {
       console.error('Erro no Kafka Producer:', err);
     });
     ```
   - **Explicação:**
     - `KafkaClient`: Cria uma instância do cliente Kafka.
     - `Producer`: Inicializa o produtor Kafka para envio de mensagens.
     - `producer.on('ready')`: Evento disparado quando o produtor Kafka está pronto.
     - `producer.on('error')`: Evento disparado em caso de erro no produtor Kafka.

5. **`kafka.service.ts`**
   - **Caminho:** `src/users/kafka.service.ts`
   - **Função:** Serviço que interage com o Kafka para criar, listar e buscar tópicos, além de gerenciar mensagens.
   - **Conteúdo:**
     ```typescript
     import { Injectable } from '@nestjs/common';
     import { Kafka, Admin, Consumer } from 'kafkajs';
     import { InjectModel } from '@nestjs/mongoose';
     import { Model } from 'mongoose';
     import { Topic } from './schemas/topic.schema';

     @Injectable()
     export class KafkaService {
       private kafka: Kafka;
       private admin: Admin;
       private consumer: Consumer;

       constructor(
         @InjectModel(Topic.name) private readonly topicModel: Model<Topic>,
       ) {
         this.kafka = new Kafka({
           clientId: 'my-app',
           brokers: ['localhost:29092'], // Ajuste conforme necessário
         });

         this.admin = this.kafka.admin();
         this.consumer = this.kafka.consumer({ groupId: 'your-group-id' });
       }

       async createTopic(userId: string, createTopicDto: any) {
         const topicName = `${userId}-${createTopicDto.topicName}`;

         // Criação do tópico no Kafka
         await this.admin.connect();
         await this.admin.createTopics({
           topics: [{ topic: topicName, numPartitions: 1, replicationFactor: 1 }],
         });
         await this.admin.disconnect();

         // Salvando a informação do tópico no banco de dados
         const createdTopic = new this.topicModel({ topicName, userId });
         const savedTopic = await createdTopic.save();

         // Retornando o formato desejado
         return {
           message: 'Tópico criado com sucesso',
           topic: savedTopic,
         };
       }

       async listTopics(userId: string) {
         // Buscando os tópicos do usuário no banco de dados
         const userTopics = await this.topicModel.find({ userId }).exec();
         return userTopics;
       }

       async getTopicById(userId: string, topicId: string) {
         // Buscando o tópico do banco de dados para obter o nome
         const topic = await this.topicModel.findOne({ _id: topicId, userId }).exec();
         if (!topic) {
           throw new Error('Tópico não encontrado ou você não tem permissão para acessá-lo');
         }

         // Obtendo mensagens do tópico do Kafka
         const messages = await this.getMessagesFromTopic(topic.topicName);
         return {
           topic: topic.topicName,
           messages,
         };
       }

       private async getMessagesFromTopic(topic: string) {
         const messages = [];
       
         await this.consumer.connect();
         try {
           await this.consumer.subscribe({ topic, fromBeginning: true });
       
           await this.consumer.run({
             eachMessage: async ({ message }) => {
               messages.push({
                 key: message.key?.toString(),
                 value: message.value?.toString(),
               });
             },
           });

           // Espera um pouco para garantir que as mensagens foram consumidas
           await new Promise(resolve => setTimeout(resolve, 1000));
           return messages;
         } catch (error) {
           console.error('Error getting messages from topic:', error);
           throw error;
         } finally {
           await this.consumer.disconnect();
         }
       }
     }
     ```
   - **Explicação:**
     - `KafkaService`: Serviço que gerencia as operações relacionadas ao Kafka.
     - `createTopic()`: Cria um novo tópico no Kafka e salva suas informações no banco de dados.
     - `listTopics()`: Lista todos os tópicos criados por um usuário específico.
     - `getTopicById()`: Retorna as mensagens de um tópico específico por ID.
     - `getMessagesFromTopic()`: Consome e retorna as mensagens de um tópico Kafka.

---

Essa documentação oferece uma visão abrangente do gerenciamento de tópicos no Kafka dentro do seu projeto, facilitando a compreensão e manutenção do código.
