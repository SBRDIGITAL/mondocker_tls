# Методичка по развёртыванию MongoDB в Docker Compose

### Шаг 1: Установка Зависимости `Docker`, `Docker Compose` и `OpenSSL`

1. **Установите Docker**:
   - Перейдите на [официальный сайт Docker](https://www.docker.com/products/docker-desktop) и скачайте установщик для вашей операционной системы.
   - Установите Docker, следуя инструкциям на экране.

2. **Установите openssl**
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

1. **Склонируйте репозиторий**:
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
   - Используйте следующую команду для перехода в директорию `certs`
    ```bash
    cd certs
    ```

2. **Создание CA сертификата**:
   - Используйте следующую команду для создания самоподписанного CA сертификата без пароля:
   ```bash
   openssl req -new -x509 -days 365 -keyout ca.key -out ca.pem -nodes -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=MyCA"
   ```

3. **Создание ключа сервера**:
   Создайте ключ для сервера без пароля:
   ```bash
   openssl genpkey -algorithm RSA -out server.key -pkeyopt rsa_keygen_bits:2048
   ```

4. **Создание запроса на подпись сертификата (CSR) для сервера**:
   Создайте CSR для вашего сервера:
   ```bash
   openssl req -new -key server.key -out server.csr -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=localhost"
   ```

5. **Подпись CSR с помощью CA сертификата**:
   Подпишите CSR с помощью вашего CA сертификата:
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
    - Используйте следующую команду для перехода в корень проекта(где лежит `docker-compose.yml`) из дирекории `certs`
    ```bash
    cd ..
    ```

2. **Запустите контейнеры**:
   - Для запуска контейнера в фоновом режиме в командной строке выполните:
   ```bash
   docker-compose up -d
   ```
   - Для запуска контейнера, чтоб видеть логи в реальном времени в командной строке выполните:
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

1. [Скачайте](https://www.mongodb.com/try/download/compass) и установите GUI для нужной OS с официального сайта:

2. В разделе `Connections` нажмите на `+`(Add new connection)
3. В строке подключения формата `mongodb://localhost:27017` введите ip адрес сервера, на котором развернут MongoDB, а порт используйте тот который указан слева в этой части `docker-compose.yml`:
    ```yml
        ports:
        - "35135:27017"
    ```