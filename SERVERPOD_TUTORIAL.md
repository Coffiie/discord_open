# Обучающее руководство по Serverpod

> Полное руководство по созданию full-stack приложений с использованием Serverpod на примере Discord Clone проекта

## Содержание

1. [Введение в Serverpod](#введение-в-serverpod)
2. [Начало работы](#начало-работы)
3. [Структура проекта](#структура-проекта)
4. [Создание моделей данных](#создание-моделей-данных)
5. [Разработка API эндпоинтов](#разработка-api-эндпоинтов)
6. [Работа с базой данных](#работа-с-базой-данных)
7. [Аутентификация и авторизация](#аутентификация-и-авторизация)
8. [Real-time коммуникация](#real-time-коммуникация)
9. [Интеграция с Flutter](#интеграция-с-flutter)
10. [Миграции базы данных](#миграции-базы-данных)
11. [Развертывание](#развертывание)
12. [Best Practices](#best-practices)

---

## Введение в Serverpod

### Что такое Serverpod?

**Serverpod** — это современный open-source фреймворк для создания backend на языке Dart. Он предоставляет все необходимое для разработки масштабируемых серверных приложений:

- **Type-safe API** — автоматическая генерация клиентского кода
- **ORM** для работы с базами данных (PostgreSQL)
- **Real-time коммуникация** через WebSockets
- **Встроенная аутентификация**
- **Кэширование** через Redis
- **Миграции базы данных**
- **Автоматическая сериализация/десериализация**

### Преимущества Serverpod

1. **Единый язык** — Dart на frontend (Flutter) и backend
2. **Type safety** — полная типизация от клиента до базы данных
3. **Быстрая разработка** — автогенерация кода экономит время
4. **Масштабируемость** — готов к production нагрузкам
5. **Простая интеграция** — идеально работает с Flutter

---

## Начало работы

### Установка

#### Шаг 1: Установка Serverpod CLI

```bash
dart pub global activate serverpod_cli
```

#### Шаг 2: Создание нового проекта

```bash
serverpod create myapp
```

Эта команда создаст три модуля:
- `myapp_server` — серверная часть
- `myapp_client` — автогенерируемый клиент
- `myapp_flutter` — Flutter приложение

#### Шаг 3: Настройка Docker для базы данных

В директории `myapp_server` создается `docker-compose.yaml`:

```yaml
services:
  postgres:
    image: postgres:16.3
    ports:
      - '8090:5432'
    environment:
      POSTGRES_USER: postgres
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: "your_secure_password"
    volumes:
      - myapp_data:/var/lib/postgresql/data

  redis:
    image: redis:6.2.6
    ports:
      - '8091:6379'
    command: redis-server --requirepass "your_redis_password"

volumes:
  myapp_data:
```

#### Шаг 4: Запуск базы данных

```bash
cd myapp_server
docker-compose up --build --detach
```

#### Шаг 5: Конфигурация сервера

Файл `config/development.yaml`:

```yaml
# API сервер
apiServer:
  port: 8080
  publicHost: localhost
  publicPort: 8080
  publicScheme: http

# Insights сервер (мониторинг)
insightsServer:
  port: 8081
  publicHost: localhost
  publicPort: 8081
  publicScheme: http

# Web сервер
webServer:
  port: 8082
  publicHost: localhost
  publicPort: 8082
  publicScheme: http

# База данных
database:
  host: localhost
  port: 8090
  name: myapp
  user: postgres

# Redis
redis:
  enabled: true
  host: localhost
  port: 8091
```

---

## Структура проекта

### Типичная структура Serverpod проекта

```
myapp_server/
├── bin/
│   └── main.dart                 # Точка входа
├── lib/
│   ├── server.dart              # Конфигурация сервера
│   ├── src/
│   │   ├── endpoints/           # API эндпоинты
│   │   │   ├── user_endpoint.dart
│   │   │   └── message_endpoint.dart
│   │   ├── models/              # Модели данных (.spy.yaml)
│   │   │   ├── user.spy.yaml
│   │   │   └── message.spy.yaml
│   │   └── generated/           # Автогенерируемый код
│   │       ├── protocol.dart
│   │       └── endpoints.dart
├── config/
│   ├── development.yaml         # Настройки разработки
│   ├── production.yaml          # Production настройки
│   └── passwords.yaml           # Пароли (не коммитить!)
├── migrations/                  # Миграции БД
├── docker-compose.yaml          # Docker конфигурация
└── pubspec.yaml                # Зависимости
```

### Точка входа сервера

`bin/main.dart`:
```dart
import 'package:myapp_server/server.dart';

void main(List<String> args) {
  run(args);
}
```

`lib/server.dart`:
```dart
import 'package:serverpod/serverpod.dart';
import 'src/generated/protocol.dart';
import 'src/generated/endpoints.dart';

void run(List<String> args) async {
  // Инициализация Serverpod
  final pod = Serverpod(
    args,
    Protocol(),
    Endpoints(),
  );

  // Настройка web routes
  pod.webServer.addRoute(RouteRoot(), '/');

  // Запуск сервера
  await pod.start();
}
```

---

## Создание моделей данных

### Формат .spy.yaml

Serverpod использует файлы `.spy.yaml` для определения моделей данных. Из них автоматически генерируется код для сервера, клиента и базы данных.

### Пример 1: Простая модель сервера

`lib/src/models/discord_server.spy.yaml`:
```yaml
class: DiscordServer
table: discord_server
fields:
  name: String
  newMessagesCount: int
  newMessagesChats: int
  serverBackground: String
  defaultChannelId: int?, relation(parent=discord_channel)
```

**Объяснение полей:**
- `class` — имя класса в Dart
- `table` — имя таблицы в PostgreSQL
- `fields` — поля модели
- `int?` — nullable поле
- `relation(parent=...)` — foreign key связь

### Пример 2: Модель с отношениями

`lib/src/models/message.spy.yaml`:
```yaml
class: Message
table: message
fields:
  senderInfo: module:auth:UserInfo?, relation
  content: String
  contentType: String
  timeStamp: DateTime
  channelId: int, relation(parent=discord_channel)
  isDelivered: bool
  isDeleted: bool
```

**Особенности:**
- `module:auth:UserInfo?` — использование модели из модуля аутентификации
- `relation` — связь с другой таблицей
- `DateTime` — автоматическая конвертация типов

### Пример 3: Модель с enum

`lib/src/models/enums/activity_status.spy.yaml`:
```yaml
enum: ActivityStatus
values:
  - online
  - idle
  - dnd
  - invisible
```

### Пример 4: Сложная модель пользователя

`lib/src/models/discord_user.spy.yaml`:
```yaml
class: DiscordUser
table: discord_user
fields:
  userInfoId: int, relation(parent=serverpod_user_info)
  userInfo: module:auth:UserInfo?, relation
  status: ActivityStatus
  members: List<ServerMembership>?, relation
```

### Генерация кода из моделей

После создания `.spy.yaml` файлов, выполните:

```bash
serverpod generate
```

Эта команда:
1. Создает Dart классы в `src/generated/`
2. Генерирует SQL схему для миграций
3. Обновляет клиентскую библиотеку
4. Создает сериализаторы/десериализаторы

### Типы данных в Serverpod

| YAML тип | Dart тип | PostgreSQL тип |
|----------|----------|----------------|
| `int` | `int` | `INTEGER` |
| `double` | `double` | `DOUBLE PRECISION` |
| `bool` | `bool` | `BOOLEAN` |
| `String` | `String` | `TEXT` |
| `DateTime` | `DateTime` | `TIMESTAMP` |
| `ByteData` | `ByteData` | `BYTEA` |
| `Duration` | `Duration` | `BIGINT` |
| `UuidValue` | `UuidValue` | `UUID` |
| `List<T>` | `List<T>` | `JSON` (если не relation) |

---

## Разработка API эндпоинтов

### Создание простого эндпоинта

`lib/src/endpoints/discord_server_endpoint.dart`:

```dart
import 'package:serverpod/serverpod.dart';
import '../generated/protocol.dart';

class DiscordServerEndpoint extends Endpoint {

  /// Получить все серверы
  Future<List<DiscordServer>> getServers(Session session) async {
    return await DiscordServer.db.find(session);
  }

  /// Получить сервер по ID
  Future<DiscordServer?> getServerById(Session session, int id) async {
    return await DiscordServer.db.findById(session, id);
  }
}
```

**Ключевые моменты:**
- Наследование от `Endpoint`
- Параметр `Session session` обязателен в каждом методе
- Методы должны быть `Future<T>` или `Stream<T>`
- Все публичные методы автоматически становятся API эндпоинтами

### Эндпоинт с параметрами

```dart
class DiscordServerEndpoint extends Endpoint {

  /// Получить серверы с фильтрацией
  Future<List<DiscordServer>> getAllServers(
    Session session, {
    List<int>? serverIds,
    int? userId,
  }) async {
    return await session.db.transaction((transaction) async {
      // Если указаны конкретные ID
      if (serverIds != null && serverIds.isNotEmpty) {
        return await _findServerFromIds(session, serverIds);
      }

      // Если указан пользователь
      if (userId != null) {
        final user = await DiscordUser.db.findById(
          session,
          userId,
          include: DiscordUser.include(
            members: ServerMembership.includeList(),
          ),
        );

        if (user != null) {
          final ids = user.members?.map((e) => e.serverId).toList();
          if (ids != null && ids.isNotEmpty) {
            return await _findServerFromIds(session, ids);
          }
        }
      }

      // Вернуть все серверы
      return await DiscordServer.db.find(session);
    });
  }
}
```

### Эндпоинт с транзакциями

```dart
class DiscordServerEndpoint extends Endpoint {

  /// Создать новый сервер с каналами
  Future<DiscordServer> createServer(
    Session session, {
    required String serverName,
    required String serverBackground,
    required int creatorId,
  }) async {
    return await session.db.transaction((transaction) async {
      // 1. Создать сервер
      final server = DiscordServer(
        name: serverName,
        serverBackground: serverBackground,
        newMessagesCount: 0,
        newMessagesChats: 0,
      );
      final createdServer = await DiscordServer.db.insertRow(session, server);

      // 2. Добавить создателя как участника
      await DiscordUserEndpoint().addMemberToServer(
        session,
        createdServer.id!,
        creatorId,
      );

      // 3. Создать группу каналов
      final group = Group(
        discordServerId: createdServer.id!,
        name: 'Text Channels',
        type: GroupType.text,
      );
      final createdGroup = await Group.db.insertRow(session, group);

      // 4. Создать канал по умолчанию
      final channel = DiscordChannel(
        groupId: createdGroup.id!,
        name: 'general',
        icon: 'hash',
        discordServerId: createdServer.id!,
        type: GroupType.text,
      );
      final createdChannel = await DiscordChannel.db.insertRow(session, channel);

      // 5. Обновить сервер с ID канала по умолчанию
      await DiscordServer.db.updateRow(
        session,
        createdServer.copyWith(
          defaultChannelId: createdChannel.id,
        ),
      );

      // Вернуть созданный сервер
      return (await DiscordServer.db.findById(session, createdServer.id!))!;
    });
  }
}
```

**Важно:**
- Транзакции гарантируют атомарность операций
- При ошибке все изменения откатываются
- Используйте `session.db.transaction()` для связанных операций

### Обработка ошибок

```dart
class MessageEndpoint extends Endpoint {

  Future<Message> deleteMessage(Session session, int messageId) async {
    return await session.db.transaction((transaction) async {
      final message = await Message.db.findById(
        session,
        messageId,
        include: Message.include(
          senderInfo: UserInfo.include(),
        ),
      );

      // Проверка существования
      if (message == null) {
        throw NotFoundException(
          message: 'Message not found',
          status: 404,
        );
      }

      // Soft delete
      await Message.db.updateRow(
        session,
        message.copyWith(isDeleted: true)
      );

      final updatedMessage = await Message.db.findById(
        session,
        messageId,
        include: Message.include(
          senderInfo: UserInfo.include(),
        ),
      );

      if (updatedMessage == null) {
        throw NotFoundException(
          message: 'Message not found',
          status: 404,
        );
      }

      return updatedMessage;
    });
  }
}
```

---

## Работа с базой данных

### CRUD операции

#### Create (INSERT)

```dart
// Создать новую запись
final server = DiscordServer(
  name: 'My Server',
  newMessagesCount: 0,
  newMessagesChats: 0,
  serverBackground: 'https://example.com/bg.jpg',
);

final createdServer = await DiscordServer.db.insertRow(session, server);
print('Created server with ID: ${createdServer.id}');
```

#### Read (SELECT)

```dart
// Найти по ID
final server = await DiscordServer.db.findById(session, 1);

// Найти все
final allServers = await DiscordServer.db.find(session);

// Найти с условием
final servers = await DiscordServer.db.find(
  session,
  where: (server) => server.newMessagesCount.greaterThan(0),
);

// Найти с сортировкой и лимитом
final recentMessages = await Message.db.find(
  session,
  where: (msg) => msg.channelId.equals(channelId),
  orderBy: (msg) => msg.timeStamp,
  orderDescending: true,
  limit: 20,
);
```

#### Update (UPDATE)

```dart
// Обновить запись
final server = await DiscordServer.db.findById(session, 1);
if (server != null) {
  final updatedServer = server.copyWith(
    newMessagesCount: server.newMessagesCount + 1,
  );
  await DiscordServer.db.updateRow(session, updatedServer);
}
```

#### Delete (DELETE)

```dart
// Удалить запись
await DiscordServer.db.deleteRow(session, server);

// Удалить по условию
await Message.db.deleteWhere(
  session,
  where: (msg) => msg.isDeleted.equals(true),
);
```

### Работа с отношениями (Relations)

#### Include отношений при запросе

```dart
// Загрузить сообщение с информацией об отправителе
final message = await Message.db.findById(
  session,
  messageId,
  include: Message.include(
    senderInfo: UserInfo.include(),
  ),
);

print('Message from: ${message.senderInfo?.userName}');
```

#### Вложенные include

```dart
final user = await DiscordUser.db.findById(
  session,
  userId,
  include: DiscordUser.include(
    userInfo: UserInfo.include(),
    members: ServerMembership.includeList(
      include: ServerMembership.include(
        // Вложенные отношения
      ),
    ),
  ),
);
```

### Пагинация

```dart
class MessageEndpoint extends Endpoint {

  /// Получить сообщения с пагинацией
  Future<List<Message>> fetchChatsPaginated(
    Session session,
    int channelId,
    int cursor,
  ) async {
    return await Message.db.find(
      session,
      where: (message) =>
          message.channelId.equals(channelId) &
          (message.id < cursor),
      include: Message.include(
        senderInfo: UserInfo.include(),
      ),
      orderBy: (message) => message.timeStamp,
      limit: 15,
      orderDescending: true,
    );
  }
}
```

### Сложные запросы

```dart
// Комбинирование условий с AND
final messages = await Message.db.find(
  session,
  where: (msg) =>
    msg.channelId.equals(channelId) &
    msg.isDeleted.equals(false) &
    msg.timeStamp.greaterThan(DateTime.now().subtract(Duration(days: 7))),
);

// Комбинирование условий с OR
final messages = await Message.db.find(
  session,
  where: (msg) =>
    msg.channelId.equals(1) | msg.channelId.equals(2),
);
```

---

## Аутентификация и авторизация

### Настройка аутентификации

`lib/server.dart`:

```dart
import 'package:serverpod_auth_server/serverpod_auth_server.dart' as auth;

void run(List<String> args) async {
  // Конфигурация аутентификации
  auth.AuthConfig.set(auth.AuthConfig(
    // Отправка кода подтверждения email
    sendValidationEmail: (session, email, validationCode) async {
      print('Validation code for $email: $validationCode');
      // В production отправляйте email через SendGrid, AWS SES и т.д.
      return true;
    },

    // Callback при создании пользователя
    onUserCreated: (session, userInfo) async {
      final userId = userInfo.id;
      if (userId != null) {
        // Создать профиль пользователя
        final user = DiscordUser(
          userInfoId: userId,
          userInfo: userInfo,
          status: ActivityStatus.online,
          members: [
            ServerMembership(
              serverId: 1,  // Добавить на сервер по умолчанию
              userId: userId,
              discordUserId: userId,
            )
          ],
        );
        await DiscordUser.db.insertRow(session, user);
      }
    },
  ));

  // Инициализация Serverpod с аутентификацией
  final pod = Serverpod(
    args,
    Protocol(),
    Endpoints(),
    authenticationHandler: auth.authenticationHandler,  // Добавить handler
  );

  await pod.start();
}
```

### Защита эндпоинтов

```dart
class MessageEndpoint extends Endpoint {

  /// Редактировать сообщение (требуется аутентификация)
  Future<Message> editMessage(
    Session session,
    int messageId,
    String content,
  ) async {
    // Проверка аутентификации
    final authenticatedUser = await session.authenticated;
    if (authenticatedUser == null) {
      throw Exception('User not authenticated');
    }

    return await session.db.transaction((transaction) async {
      final message = await Message.db.findById(
        session,
        messageId,
        include: Message.include(
          senderInfo: UserInfo.include(),
        ),
      );

      if (message == null) {
        throw NotFoundException(
          message: 'Message not found',
          status: 404,
        );
      }

      // Проверка владения
      if (message.senderInfoId != authenticatedUser.userId) {
        throw Exception('You are not the sender of this message');
      }

      // Обновление
      await Message.db.updateRow(
        session,
        message.copyWith(content: content),
      );

      return (await Message.db.findById(
        session,
        messageId,
        include: Message.include(
          senderInfo: UserInfo.include(),
        ),
      ))!;
    });
  }
}
```

### Получение информации о пользователе

```dart
class UserEndpoint extends Endpoint {

  Future<DiscordUser?> getCurrentUser(Session session) async {
    final authenticatedUser = await session.authenticated;

    if (authenticatedUser == null) {
      return null;
    }

    return await DiscordUser.db.findById(
      session,
      authenticatedUser.userId,
      include: DiscordUser.include(
        userInfo: UserInfo.include(),
        members: ServerMembership.includeList(),
      ),
    );
  }
}
```

---

## Real-time коммуникация

### Создание стримов

Serverpod предоставляет встроенную поддержку WebSocket для real-time коммуникации.

#### Server-side: Отправка сообщений в stream

```dart
class MessageEndpoint extends Endpoint {

  /// Отправить сообщение и транслировать всем клиентам
  Future<Message> sendMessage(Session session, Message message) async {
    return await session.db.transaction((transaction) async {
      // Сохранить сообщение в БД
      final createdMessage = await Message.db.insertRow(session, message);

      // Транслировать сообщение всем подписанным клиентам
      final updatedMessage = createdMessage.copyWith(isDelivered: true);
      final success = await session.messages.postMessage(
        updatedMessage.channelId.toString(),  // Название канала
        updatedMessage,                        // Данные
      );

      if (success) {
        return await Message.db.updateRow(session, updatedMessage);
      } else {
        throw MessageNotSentException(
          message: 'Message not sent',
          status: 500,
        );
      }
    });
  }

  /// Stream для получения сообщений
  Stream<Message> listenToMessages(Session session, int channelId) async* {
    // Создать stream для конкретного канала
    final messages = session.messages.createStream<Message>(
      channelId.toString()
    );

    // Транслировать сообщения клиенту
    await for (var message in messages) {
      yield message;
    }
  }
}
```

**Как это работает:**
1. `session.messages.postMessage()` — отправляет сообщение всем подписанным клиентам
2. `session.messages.createStream<T>()` — создает stream для прослушивания
3. Каналы идентифицируются строковым ключом (`channelId.toString()`)

### Множественные каналы

```dart
class LiveStreamEndpoint extends Endpoint {

  /// Отправить уведомление о присоединении к комнате
  Future<void> notifyRoomJoin(
    Session session,
    String roomId,
    String userName,
  ) async {
    final notification = {
      'type': 'user_joined',
      'userName': userName,
      'timestamp': DateTime.now().toIso8601String(),
    };

    await session.messages.postMessage(
      'room_$roomId',
      notification,
    );
  }

  /// Прослушивать события комнаты
  Stream<Map<String, dynamic>> listenToRoom(
    Session session,
    String roomId,
  ) async* {
    final events = session.messages.createStream<Map<String, dynamic>>(
      'room_$roomId'
    );

    await for (var event in events) {
      yield event;
    }
  }
}
```

---

## Интеграция с Flutter

### Настройка клиента

`lib/locator.dart`:

```dart
import 'package:discord_client/discord_client.dart';
import 'package:get_it/get_it.dart';
import 'package:serverpod_flutter/serverpod_flutter.dart';

final locator = GetIt.instance;

void setupLocator() {
  // Создать Serverpod клиент
  final client = Client(
    'http://localhost:8080/',  // URL сервера
    authenticationKeyManager: FlutterAuthenticationKeyManager(),
  )..connectivityMonitor = FlutterConnectivityMonitor();

  // Зарегистрировать клиент
  locator.registerSingleton<Client>(client);

  // Зарегистрировать SessionManager
  locator.registerSingleton<SessionManager>(
    SessionManager(caller: client.modules.auth),
  );

  // Зарегистрировать репозитории
  locator.registerLazySingleton<ChatRepository>(
    () => ChatRepository(client: locator<Client>()),
  );

  locator.registerLazySingleton<ServerRepository>(
    () => ServerRepository(client: locator<Client>()),
  );
}
```

### Создание репозитория

`lib/infrastructure/chat/chat_repository.dart`:

```dart
import 'package:discord_client/discord_client.dart';
import 'package:flutter/material.dart';

class ChatRepository {
  ChatRepository({required this.client});

  final Client client;

  /// Получить сообщения с пагинацией
  Future<List<Message>> fetchChatsPaginated(int channelId, int cursor) async {
    try {
      final messages = await client.message.fetchChatsPaginated(
        channelId,
        cursor,
      );
      return messages;
    } on Exception catch (e) {
      debugPrint('[ChatRepository - fetchChatsPaginated] ERROR: $e');
      rethrow;
    }
  }

  /// Слушать новые сообщения (real-time)
  Stream<Message> listenToMessages(int channelId) {
    return client.message.listenToMessages(channelId);
  }

  /// Отправить сообщение
  Future<Message> sendMessage(Message message) async {
    try {
      final updatedMessage = await client.message.sendMessage(message);
      return updatedMessage;
    } on Exception catch (e) {
      debugPrint('[ChatRepository - sendMessage] ERROR: $e');
      rethrow;
    }
  }

  /// Удалить сообщение
  Future<Message> deleteMessage(Message message) async {
    try {
      final updatedMessage = await client.message.deleteMessage(message.id!);
      return updatedMessage;
    } on Exception catch (e) {
      debugPrint('[ChatRepository - deleteMessage] ERROR: $e');
      rethrow;
    }
  }

  /// Редактировать сообщение
  Future<Message> editMessage(Message message) async {
    try {
      final updatedMessage = await client.message.editMessage(
        message.id!,
        message.content,
      );
      return updatedMessage;
    } on Exception catch (e) {
      debugPrint('[ChatRepository - editMessage] ERROR: $e');
      rethrow;
    }
  }
}
```

### Использование в Cubit (State Management)

```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:discord_client/discord_client.dart';

part 'chat_cubit.freezed.dart';
part 'chat_state.dart';

class ChatCubit extends Cubit<ChatState> {
  ChatCubit({required this.chatRepository}) : super(ChatState.initial());

  final ChatRepository chatRepository;
  StreamSubscription<Message>? _messageSubscription;

  /// Подписаться на сообщения канала
  Future<void> listenToMessages(int channelId) async {
    await _messageSubscription?.cancel();

    _messageSubscription = chatRepository
        .listenToMessages(channelId)
        .listen((message) {
      // Добавить новое сообщение в состояние
      final updatedMessages = [message, ...state.messages];
      emit(state.copyWith(messages: updatedMessages));
    });
  }

  /// Загрузить сообщения с пагинацией
  Future<void> fetchChatsPaginated(int channelId, int cursor) async {
    try {
      emit(state.copyWith(isLoadingMore: true));

      final messages = await chatRepository.fetchChatsPaginated(
        channelId,
        cursor,
      );

      final updatedMessages = [...state.messages, ...messages];

      emit(state.copyWith(
        messages: updatedMessages,
        isLoadingMore: false,
        hasMoreMessages: messages.length >= 15,
      ));
    } catch (e) {
      emit(state.copyWith(
        isLoadingMore: false,
        error: e.toString(),
      ));
    }
  }

  /// Отправить сообщение
  Future<void> sendMessage(Message message) async {
    try {
      await chatRepository.sendMessage(message);
      // Сообщение будет добавлено через stream
    } catch (e) {
      emit(state.copyWith(error: e.toString()));
    }
  }

  @override
  Future<void> close() {
    _messageSubscription?.cancel();
    return super.close();
  }
}
```

### Использование в UI

```dart
class ChatView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return BlocBuilder<ChatCubit, ChatState>(
      builder: (context, state) {
        if (state.isLoading) {
          return CircularProgressIndicator();
        }

        return ListView.builder(
          reverse: true,  // Новые сообщения внизу
          itemCount: state.messages.length,
          itemBuilder: (context, index) {
            final message = state.messages[index];
            return MessageTile(message: message);
          },
        );
      },
    );
  }
}
```

---

## Миграции базы данных

### Создание миграции

После изменения моделей в `.spy.yaml` файлах:

```bash
# 1. Генерация кода
serverpod generate

# 2. Создание миграции
serverpod create-migration
```

Serverpod автоматически создаст SQL файлы в `migrations/`:

```
migrations/
├── 20240101120000-initial.sql
├── 20240102153000-add_messages.sql
└── 20240103094500-add_user_status.sql
```

### Применение миграций

```bash
# Применить все миграции
dart bin/main.dart --apply-migrations

# Применить в production
dart bin/main.dart --apply-migrations --role=production
```

### Пример миграции

`migrations/20240102153000-add_messages.sql`:

```sql
BEGIN;

-- Create message table
CREATE TABLE message (
    id SERIAL PRIMARY KEY,
    "senderInfoId" INTEGER,
    content TEXT NOT NULL,
    "contentType" TEXT NOT NULL,
    "timeStamp" TIMESTAMP NOT NULL,
    "channelId" INTEGER NOT NULL,
    "isDelivered" BOOLEAN NOT NULL DEFAULT false,
    "isDeleted" BOOLEAN NOT NULL DEFAULT false
);

-- Add foreign key constraints
ALTER TABLE message
    ADD CONSTRAINT message_fk_0
    FOREIGN KEY ("senderInfoId")
    REFERENCES serverpod_user_info (id)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION;

ALTER TABLE message
    ADD CONSTRAINT message_fk_1
    FOREIGN KEY ("channelId")
    REFERENCES discord_channel (id)
    ON DELETE CASCADE
    ON UPDATE NO ACTION;

-- Create indexes
CREATE INDEX message_channel_idx ON message ("channelId");
CREATE INDEX message_timestamp_idx ON message ("timeStamp" DESC);

COMMIT;
```

### Откат миграций

Если миграция вызвала проблемы:

```bash
# Вручную откатить через psql
psql -h localhost -p 8090 -U postgres -d myapp

# Выполнить DROP TABLE или ALTER TABLE для отката
```

**Важно:** Serverpod пока не поддерживает автоматический откат миграций. Делайте резервные копии БД перед миграциями в production!

---

## Развертывание

### Подготовка к production

#### 1. Создать production конфигурацию

`config/production.yaml`:

```yaml
apiServer:
  port: 8080
  publicHost: api.myapp.com
  publicPort: 443
  publicScheme: https

database:
  host: db.myapp.com
  port: 5432
  name: myapp_prod
  user: postgres

redis:
  enabled: true
  host: redis.myapp.com
  port: 6379
```

#### 2. Настроить пароли

`config/passwords.yaml`:

```yaml
production:
  database: 'your_strong_db_password'
  redis: 'your_strong_redis_password'
```

**Важно:** Не коммитьте `passwords.yaml` в git!

#### 3. Собрать Docker образ

`Dockerfile`:

```dockerfile
FROM dart:stable AS build

WORKDIR /app
COPY pubspec.* ./
RUN dart pub get

COPY . .
RUN dart compile exe bin/main.dart -o bin/server

FROM debian:bookworm-slim
COPY --from=build /app/bin/server /app/bin/server
COPY --from=build /app/config /app/config
COPY --from=build /app/migrations /app/migrations

EXPOSE 8080 8081 8082

CMD ["/app/bin/server", "--mode=production"]
```

#### 4. Docker Compose для production

`docker-compose.prod.yaml`:

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
      - "8081:8081"
      - "8082:8082"
    environment:
      - MODE=production
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  postgres:
    image: postgres:16.3
    environment:
      POSTGRES_USER: postgres
      POSTGRES_DB: myapp_prod
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:6.2.6
    command: redis-server --requirepass ${REDIS_PASSWORD}
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app
    restart: unless-stopped

volumes:
  postgres_data:
```

#### 5. Nginx конфигурация

`nginx.conf`:

```nginx
events {
    worker_connections 1024;
}

http {
    upstream api {
        server app:8080;
    }

    server {
        listen 80;
        server_name api.myapp.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name api.myapp.com;

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        location / {
            proxy_pass http://api;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### Развертывание на облачных платформах

#### AWS (Amazon Web Services)

1. **ECS (Elastic Container Service)**
   - Загрузить Docker образ в ECR
   - Создать Task Definition
   - Настроить Service с ALB
   - Использовать RDS для PostgreSQL
   - Использовать ElastiCache для Redis

2. **Lightsail**
   - Создать Container Service
   - Загрузить Docker образ
   - Настроить database instance

#### Google Cloud Platform

1. **Cloud Run**
   - Идеально для serverless deployment
   - Автоматический scaling
   - Использовать Cloud SQL для PostgreSQL

```bash
# Build и push
gcloud builds submit --tag gcr.io/PROJECT_ID/myapp

# Deploy
gcloud run deploy myapp \
  --image gcr.io/PROJECT_ID/myapp \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated
```

#### Digital Ocean

1. **App Platform**
   - GitHub integration
   - Автоматический deployment при push
   - Managed Database для PostgreSQL

2. **Droplets**
   - Полный контроль через VPS
   - Установить Docker и Docker Compose
   - Настроить SSL с Let's Encrypt

```bash
# На droplet
git clone your-repo
cd your-repo
docker-compose -f docker-compose.prod.yaml up -d
```

---

## Best Practices

### 1. Структура кода

✅ **DO:**
- Группируйте эндпоинты по функциональности
- Используйте транзакции для связанных операций
- Создавайте отдельные entity классы для сложных операций
- Держите бизнес-логику в эндпоинтах, не в моделях

❌ **DON'T:**
- Не делайте слишком большие эндпоинты
- Не дублируйте код между эндпоинтами
- Не забывайте про обработку ошибок

### 2. База данных

✅ **DO:**
- Используйте индексы для часто запрашиваемых полей
- Применяйте пагинацию для больших списков
- Используйте `include` только когда нужно
- Soft delete вместо hard delete для важных данных

❌ **DON'T:**
- Не делайте N+1 запросы
- Не загружайте все данные сразу
- Не забывайте про foreign key constraints

### 3. API дизайн

✅ **DO:**
- Используйте понятные имена методов
- Документируйте эндпоинты комментариями
- Возвращайте правильные типы ошибок
- Используйте optional параметры для гибкости

```dart
/// Получить серверы пользователя
///
/// [userId] - ID пользователя (optional)
/// [serverIds] - конкретные ID серверов (optional)
///
/// Возвращает все серверы если параметры не указаны
Future<List<DiscordServer>> getAllServers(
  Session session, {
  int? userId,
  List<int>? serverIds,
}) async { ... }
```

### 4. Безопасность

✅ **DO:**
- Всегда проверяйте `session.authenticated`
- Валидируйте входные данные
- Используйте HTTPS в production
- Храните пароли в переменных окружения
- Ограничивайте rate limiting

```dart
Future<Message> editMessage(
  Session session,
  int messageId,
  String content,
) async {
  // 1. Проверка аутентификации
  final user = await session.authenticated;
  if (user == null) {
    throw UnauthorizedException();
  }

  // 2. Валидация
  if (content.trim().isEmpty) {
    throw ValidationException('Content cannot be empty');
  }

  if (content.length > 2000) {
    throw ValidationException('Content too long');
  }

  // 3. Проверка владения
  final message = await Message.db.findById(session, messageId);
  if (message?.senderInfoId != user.userId) {
    throw ForbiddenException();
  }

  // 4. Выполнение операции
  return await Message.db.updateRow(
    session,
    message.copyWith(content: content),
  );
}
```

### 5. Производительность

✅ **DO:**
- Используйте Redis для кэширования
- Делайте batch операции где возможно
- Оптимизируйте сложные запросы
- Используйте connection pooling

```dart
// Batch операция вместо цикла
Future<List<DiscordServer>> _findServerFromIds(
  Session session,
  List<int> serverIds,
) async {
  // ❌ Плохо: N запросов
  final servers = <DiscordServer>[];
  for (var id in serverIds) {
    final server = await DiscordServer.db.findById(session, id);
    if (server != null) servers.add(server);
  }

  // ✅ Хорошо: один запрос
  return await DiscordServer.db.find(
    session,
    where: (s) => s.id.inSet(serverIds),
  );
}
```

### 6. Тестирование

✅ **DO:**
- Пишите unit тесты для эндпоинтов
- Используйте `serverpod_test` package
- Тестируйте edge cases

```dart
import 'package:serverpod_test/serverpod_test.dart';
import 'package:test/test.dart';

void main() {
  group('MessageEndpoint', () {
    late Session session;
    late MessageEndpoint endpoint;

    setUp(() async {
      session = await IntegrationTestServer().session();
      endpoint = MessageEndpoint();
    });

    test('sendMessage creates and broadcasts message', () async {
      final message = Message(
        content: 'Test message',
        channelId: 1,
        senderInfoId: 1,
        contentType: 'text',
        timeStamp: DateTime.now(),
        isDelivered: false,
        isDeleted: false,
      );

      final result = await endpoint.sendMessage(session, message);

      expect(result.isDelivered, true);
      expect(result.content, 'Test message');
    });
  });
}
```

### 7. Мониторинг

✅ **DO:**
- Используйте Serverpod Insights для мониторинга
- Логируйте важные операции
- Отслеживайте производительность запросов

```dart
Future<Message> sendMessage(Session session, Message message) async {
  final startTime = DateTime.now();

  try {
    final result = await _sendMessageInternal(session, message);

    final duration = DateTime.now().difference(startTime);
    session.log('Message sent in ${duration.inMilliseconds}ms',
                level: LogLevel.info);

    return result;
  } catch (e) {
    session.log('Failed to send message: $e', level: LogLevel.error);
    rethrow;
  }
}
```

---

## Полезные ссылки

### Официальная документация
- [Serverpod Documentation](https://docs.serverpod.dev/)
- [Serverpod GitHub](https://github.com/serverpod/serverpod)
- [Serverpod Discord Community](https://discord.gg/serverpod)

### Примеры проектов
- [Serverpod Examples](https://github.com/serverpod/serverpod/tree/main/examples)
- [Discord Clone (этот проект)](https://github.com/yourusername/discord_open)

### Дополнительные ресурсы
- [Flutter Documentation](https://flutter.dev/docs)
- [Dart Documentation](https://dart.dev/guides)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

---

## Заключение

Serverpod — это мощный фреймворк, который упрощает создание full-stack приложений на Dart. Ключевые преимущества:

- **Type-safe API** — меньше ошибок, больше уверенности
- **Автогенерация кода** — быстрая разработка
- **Real-time из коробки** — WebSocket без головной боли
- **Масштабируемость** — готовность к production
- **Единый язык** — Dart везде

Следуя этому руководству и best practices, вы сможете создавать качественные backend-приложения с Serverpod!

---

**Создано на основе Discord Clone проекта**

Автор документации: Claude
Дата: 2025
