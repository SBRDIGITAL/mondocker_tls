# Методичка по развёртыванию MongoDB в Docker Compose и подключению к ней с помощью TLS/SSL сертификатов.

### Шаг 1: Установка зависимости `Docker`, `Docker Compose` и `OpenSSL`

1. **Установите Docker**:
   - Перейдите на [официальный сайт Docker](https://www.docker.com/products/docker-desktop) и скачайте установщик для вашей операционной системы.
   - Установите Docker, следуя инструкциям на экране.

2. **Установите OpenSSL**:
   - Используйте команду для установки:
   ```bash
   apt install openssl
   ```

3. **Проверьте установку**:
   - Для проверки установки `Docker` и `Docker Compose` в командной строке выполните команды:
   ```bash
   docker --version
   docker-compose --version
   ```
   - Для проверки установки `OpenSSL` в командной строке выполните команду:
   ```bash
   openssl help
   ```

---
### Шаг 2: Настройка проекта

1. **Клонируйте репозиторий**:
   - Используйте следующую команду для клонирования репозитория:
   ```bash
   git clone https://github.com/SBRDIGITAL/sbr_mongo_db.git ./
   ```

### Шаг 3: Создание сертификатов

1. **Создание директории для хранения сертификатов**:
   - Используйте следующую команду для создания в корне проекта директории:
   ```bash
   mkdir certs
   ```
   - Используйте следующую команду для перехода в директорию `certs`:
   ```bash
   cd certs
   ```

2. **Создание CA сертификата**:
   - Используйте следующую команду для создания самоподписанного CA сертификата без пароля:
   ```bash
   openssl req -new -x509 -days 365 -keyout ca.key -out ca.pem -nodes -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=MyCA"
   ```

3. **Создание ключа сервера**:
   - Создайте ключ для сервера без пароля:
   ```bash
   openssl genpkey -algorithm RSA -out server.key -pkeyopt rsa_keygen_bits:2048
   ```

4. **Создание запроса на подпись сертификата (CSR) для сервера**:
   - Создайте CSR для вашего сервера:
   ```bash
   openssl req -new -key server.key -out server.csr -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=localhost"
   ```

5. **Подпись CSR с помощью CA сертификата**:
   - Подпишите CSR с помощью вашего CA сертификата:
   ```bash
   openssl x509 -req -in server.csr -CA ca.pem -CAkey ca.key -CAcreateserial -out server.pem -days 365
   ```

6. **Создание файла, содержащего сертификат и приватный ключ**:
   - Используйте следующую команду для создания файла, содержащего сертификат и приватный ключ:
   ```bash
   cat server.pem server.key > ./combined.pem
   ```

### Шаг 4: Проверка сертификатов

1. После создания сертификатов проверьте их, чтобы убедиться, что они содержат правильную информацию:
   - Для CA сертификата:
   ```bash
   openssl x509 -in ca.pem -text -noout
   ```
   - Для серверного сертификата:
   ```bash
   openssl x509 -in server.pem -text -noout
   ```

---
### Шаг 5: Запуск MongoDB

1. **Возврат в корень проекта**:
   - Используйте следующую команду для перехода в корень проекта (где лежит `docker-compose.yml`) из директории `certs`:
   ```bash
   cd ..
   ```

2. **Запустите контейнеры (выберите что-то одно)**:
   - Для запуска контейнера в фоновом режиме в командной строке выполните:
   ```bash
   docker-compose up -d
   ```
   - Для запуска контейнера, чтобы видеть логи в реальном времени в командной строке выполните:
   ```bash
   docker-compose up
   ```
   - Для полной пересборки контейнера, в командной строке выполните:
   ```bash
   docker-compose up --build --force-recreate
   ```

3. **Проверьте статус контейнеров**:
   ```bash
   docker ps
   ```
   или
   ```bash
   docker-compose ps
   ```

---
### Шаг 6: Подключение к MongoDB с помощью MongoDB Compass

1. [Скачайте](https://www.mongodb.com/try/download/compass) и установите GUI для нужной ОС с официального сайта.
2. В разделе `Connections` нажмите на `+` (Add new connection).
3. В строке подключения формата `mongodb://localhost:27017` введите IP-адрес сервера, на котором развернут MongoDB, а порт используйте тот, который указан слева в этой части `docker-compose.yml`, то есть в данном случае `35135`, но по желанию можете поменять его на другой:
   ```yml
   ports:
       "35135:27017"
   ```
4. Скачайте из директории `certs` файлы `ca.pem` и `combined.pem`.
5. Поместите их в надежное место.
6. В контекстном меню подключения нажмите на пункт `Advanced Connection Options`.
7. Перейдите в раздел `TLS/SSL`.
8. Установите режим `SSL/TLS Connection` в `On`.
9. Для `Certificate Authority (.pem)` нажмите на `Select a file...` и выберите там, где сохранили `ca.pem`.
10. Для `Client Certificate and Key (.pem)` нажмите на `Select a file...` и выберите там, где сохранили `combined.pem`.
11. Установите чекбокс `tlsAllowInvalidCertificates Disable the validation of the server certificates.` в активное состояние.
12. Нажмите на кнопку `Save & Connect` и проверяйте подключение. Если нет никаких ошибок в левом нижнем углу и у установленного соединения появилась зеленая точка, значит всё прошло хорошо. Пробуйте создать вручную тестовую базу данных и коллекцию.

---

### Шаг 7: Очистка файлов
1. Иногда нужно удалять файлы диагностики и метрики если они вам не нужны, находясь в `/mondocker_tls/data/db/mongo_data`, используйте команды: 
```bash
rm -rf diagnostic.data/
```
и
```bash
rm -rf journal/
```



## Разбор `docker-compose.yml`
---
```yaml
services:
  mongo:
    image: mongo:latest
```
- **services**: Определяет набор сервисов, которые будут развернуты. В данном случае у нас есть один сервис — `mongo`.
- **image**: Указывает, какой образ Docker будет использоваться для создания контейнера. Здесь используется последний образ MongoDB.

```yaml
    container_name: mondocker_tls_1
```
- **container_name**: Задает имя контейнера, что позволяет легко идентифицировать его в списке запущенных контейнеров.

```yaml
    ports:
      - "59135:27017"
```
- **ports**: Определяет, какие порты будут открыты на хосте и перенаправлены на контейнер. В данном случае порт `59135` на хосте перенаправляется на порт `27017` внутри контейнера (стандартный порт MongoDB).

```yaml
    volumes:
      - ./data/db/mongo_data:/data/db
      - ./certs:/etc/ssl/mongo
```
- **volumes**: Позволяет монтировать директории из хоста в контейнер. 
  - `./data/db/mongo_data:/data/db`: Монтирует локальную директорию `./data/db/mongo_data` в контейнер по пути `/data/db`, что позволяет сохранять данные MongoDB на хосте.
  - `./certs:/etc/ssl/mongo`: Монтирует локальную директорию `./certs` в контейнер по пути `/etc/ssl/mongo`, что позволяет использовать сертификаты для TLS.

```yaml
    networks:
      - mongo-cluster
```
- **networks**: Определяет, к каким сетям будет подключен контейнер. В данном случае контейнер подключается к сети `mongo-cluster`.

```yaml
    command: [
        "mongod",
        "--tlsMode",
        "requireTLS",
        "--tlsCertificateKeyFile",
        "/etc/ssl/mongo/combined.pem",
        "--tlsCAFile",
        "/etc/ssl/mongo/ca.pem",
        "--verbose"
      ]
```
- **command**: Переопределяет команду, которая будет выполнена при запуске контейнера. Здесь запускается сервер MongoDB с параметрами:
  - `--tlsMode requireTLS`: Включает режим TLS и требует его для всех подключений.
  - `--tlsCertificateKeyFile /etc/ssl/mongo/combined.pem`: Указывает путь к файлу, содержащему сертификат и ключ сервера.
  - `--tlsCAFile /etc/ssl/mongo/ca.pem`: Указывает путь к файлу CA сертификата.
  - `--verbose`: Включает подробный режим логирования.

```yaml
volumes:
  mongo_data:
```
- **volumes**: Определяет именованные тома, которые могут быть использованы для хранения данных. В данном случае `mongo_data` может быть использован для хранения данных MongoDB.

```yaml
networks:
  mongo-cluster:
```
- **networks**: Определяет сети, которые могут быть использованы сервисами. В данном случае создается сеть `mongo-cluster`, к которой может подключаться контейнер MongoDB.

Эти настройки позволяют развернуть MongoDB с поддержкой TLS, обеспечивая безопасное соединение между клиентами и сервером.

---

## Пример подключения с помощью Python
```python
from asyncio import to_thread

from motor.motor_asyncio import AsyncIOMotorClient, AsyncIOMotorCursor, AsyncIOMotorCollection
from pymongo.errors import ConnectionFailure

from core.database.db_settings import (
    MongoLiterals as MLit,
    Settings, mongo_settings,
    MongoHelper    
)

from core.utils.aiomonlogger.monlogger import monlogger



class AsyncMongoCRUD:
    """
    ## Асинхронный класс для выполнения базовых операций `CRUD` с коллекцией `MongoDB`.

    Класс обеспечивает безопасное подключение к MongoDB с использованием `TLS/SSL`, включая самоподписанные сертификаты, 
    и предоставляет методы для вставки, чтения, обновления и удаления документов.

    Atributes:
        uri (str): URI для подключения к MongoDB.
        db_name (str): Имя базы данных, с которой работает класс.
        ca_cert_path (str): Путь к файлу CA-сертификата.
        combined_cert_path (str): Путь к файлу сертификата клиента и закрытого ключа.
        client (AsyncIOMotorClient): Асинхронный клиент MongoDB.
        db (AsyncIOMotorDatabase): Экземпляр базы данных MongoDB.
    """
    def __init__(self,
        uri: str = mongo_settings.URI,
        db_name: str = mongo_settings.DB_NAME,
        ca_cert_path: str = mongo_settings.CA_CERT_PATH,
        combined_cert_path: str = mongo_settings.COMBINED_CERT_PATH
    ) -> None:
        """
        ## Инициализирует экземпляр класса `AsyncMongoCRUD`.

        Настраивает параметры для подключения к `MongoDB` с использованием `TLS/SSL`. 
        Устанавливает пути к сертификатам для безопасного соединения.

        Args:
            uri (str): URI для подключения к `MongoDB`.
            db_name (str): Имя базы данных, с которой будет работать класс.
            ca_cert_path (str): Путь к файлу CA-сертификата.
            combined_cert_path (str): Путь к файлу сертификата клиента и ключа.
        """
        self.db_name = db_name
        self.uri = uri
        self.ca_cert_path = ca_cert_path
        self.combined_cert_path = combined_cert_path
        self.mh = MongoHelper()
        self.client: AsyncIOMotorClient = None
        self.db = None

    def __return_collection(self, cn: MLit.CollectionsName) -> AsyncIOMotorCollection:
        """
        ## Возвращает объект коллекции по её названию.

        Этот метод используется для получения экземпляра коллекции `MongoDB` на основе имени коллекции.

        Args:
            cn (MLit.CollectionsName): Имя коллекции, заданное в виде перечисления `CollectionsName`.

        Returns:
            AsyncIOMotorCollection: Экземпляр коллекции `MongoDB` для выполнения операций.
        """
        return self.db[cn]

    async def connect(self):
        """
        ## Устанавливает соединение с базой данных при помощи `TLS\SSL` сертификатов.
        
        Пример использования:
        ```python
            from asyncio import run
            db = AsyncMongoCRUD()

            if __name__ == '__main__':
                async def start():
                    await db.connect()
                    await db.create('users', {"name": "Jonh"})
                    await db.close()  # После работы закрываем соединение
                run(start())
        ```
        """
        # Создание клиента Motor с использованием SSL
        try:
            self.client = AsyncIOMotorClient(
                self.uri, tls=True,
                tlsAllowInvalidCertificates=True,  # Игнорировать недействительные сертификаты
                # tlsInsecure=True,  # Игнорировать ошибки проверки сертификатов
                tlsCAFile=self.ca_cert_path,
                tlsCertificateKeyFile=self.combined_cert_path,
            )
            # Проверяем подключение
            res = await self.client.admin.command("ping")
            print(f'Проверка соединения:\n{res=}')
            self.db = self.client[self.db_name]
            
        except (ConnectionFailure, Exception,) as ex:
            print(f'Ошибка подключения к MongoDB: {ex}')
            raise
    
    async def close(self):
        """ ## Закрывает соединение с базой данных. """
        if self.client:
            self.client.close()
    
    async def create(self, cn: MLit.CollectionsName, document: dict) -> None:
        """
        ## Создает новый документ в коллекции с автоинкрементным _id.

        Args:
            cn (MLit.CollectionsName): Название коллекции в которой будет создана запись.
            document (dict): Документ, который будет добавлен в коллекцию.
        """
        document = await to_thread(self.mh.add_timestamps, data=document, time_field='createdAt')
        collection = await to_thread(self.__return_collection, cn)
        result = await collection.insert_one(document)
        await monlogger.info(f'Вставлен документ с id: {result.inserted_id}')

    async def read(self,
        cn: MLit.CollectionsName,
        query: dict,
        mode: MLit.QueryModes = 'one',
        max_len: int = None
    ) -> dict | None | list[dict]:
        """
        ## Читает документы из коллекции по заданному запросу.

        Args:
            cn (MLit.CollectionsName): Название коллекции в которой будет прочитана запись.
            query (dict): Запрос для поиска документов.
            mode (MLit.QueryModes, optional): Режим чтения. 
                'one' - вернуть только один документ, 
                'many' - вернуть все документы, соответствующие запросу. 
                По умолчанию 'one'.
            max_len (int, optional): Максимальное кол-во документов для возврата в режиме 'many'. 
                По умолчанию None, что означает возврат всех документов.

        Returns:
            dict | None | list[dict]:
                Cловарь, если запись найдена(при mode == 'one').
                None если ничего не найдено(при mode == 'one') или если произошла ошибка.
                list[dict]: список записей((при mode == 'many')) если записи есть, иначе пустой список.
        """
        try:
            collection = await to_thread(self.__return_collection, cn)
            if mode == 'one':
                return await collection.find_one(query)
            elif mode == 'many':
                result: AsyncIOMotorCursor = await to_thread(collection.find, query)
                return await result.to_list(length=max_len)
                    
        except Exception as ex:
            await monlogger.ex(ex, f'Ошибка при чтении из коллекции {cn}')
            raise ex

    async def update(self, cn: MLit.CollectionsName, query: dict, update_values: dict) -> None:
        """
        ## Обновляет документ в коллекции по заданному запросу.

        Args:
            cn (MLit.CollectionsName): Название коллекции в которой будет обновлена запись.
            query (dict): Запрос для поиска документа, который нужно удалить.
            update_values (dict): Значения, которые нужно обновить 
        """
        collection = await to_thread(self.__return_collection, cn)
        document = await to_thread(self.mh.add_timestamps, data=update_values, time_field='updatedAt')
        result = await collection.update_one(query, {'$set': document})
        await monlogger.info(f'Изменено {result.modified_count} документ(ов).')

    async def delete(self, cn: MLit.CollectionsName, query: dict, mode: MLit.QueryModes = 'one') -> None:
        """
        ## Удаляет документы из коллекции по заданному запросу.

        Args:
            cn (MLit.CollectionsName): Название коллекции в которой будет удалена запись.
            query (dict): Запрос для поиска документов, которые нужно удалить.
            mode (MLit.QueryModes, optional): Режим удаления. 
                'one' - удалить только один документ, 
                'many' - удалить все документы, соответствующие запросу. 
                По умолчанию 'one'.

        Вывод:
            None: Метод выводит количество удаленных документов в консоль.
        """
        collection = await to_thread(self.__return_collection, cn)
        if mode == 'one':
            result = await collection.delete_one(query)
        else:
            result = await collection.delete_many(query)
        await monlogger.info(f'Удалено {result.deleted_count} документ(ов).')
        
        

db = AsyncMongoCRUD()
```
