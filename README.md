# amoCRM API Library

В данном пакете представлен API клиент с поддержкой основных сущностей и авторизацией по протоколу OAuth 2.0 в amoCRM.

## Установка

Установить можно с помощью composer:

```
composer require amocrm/amocrm-api-library
```

## Начало работы (авторизация)

Для начала использования вам необходимо создать объект бибилиотеки:
```php
$apiClient = new \AmoCRM\Client\AmoCRMApiClient($clientId, $clientSecret, $redirectUri);
```

Затем необходимо создать объект (`\League\OAuth2\Client\Token\AccessToken`) Access токена из вашего хранилища токенов и установить его в API клиент.

Также необходимо установить домен аккаунта amoCRM в виде СУБДОМЕН.amocrm.(ru/com).

Вы можете установить функцию-callback на событие обновления Access токена, если хотите дополнительно обрабатывать новый токен (например сохранять его в хранилище токенов):
```php
$apiClient->setAccessToken($accessToken)
        ->setAccountBaseDomain($accessToken->getValues()['baseDomain'])
        ->onAccessTokenRefresh(
            function (\League\OAuth2\Client\Token\AccessTokenInterface $accessToken, string $baseDomain) {
                saveToken(
                    [
                        'accessToken' => $accessToken->getToken(),
                        'refreshToken' => $accessToken->getRefreshToken(),
                        'expires' => $accessToken->getExpires(),
                        'baseDomain' => $baseDomain,
                    ]
                );
            });
```

Отправить пользователя на страницу авторизации можно 2мя способами:
1. Отрисовав кнопку на сайт:
```php
$apiClient->getOAuthClient()->getOAuthButton(
            [
                'title' => 'Установить интеграцию',
                'compact' => true,
                'class_name' => 'className',
                'color' => 'default',
                'error_callback' => 'handleOauthError',
                'state' => $state,
            ]
        );
```
2. Отправив пользователя на страницу авторизации
```php
$authorizationUrl = $apiClient->getOAuthClient()->getAuthorizeUrl([
            'state' => $state,
            'mode' => 'post_message', //post_message - редирект произойдет в открытом окне, popup - редирект произойдет в окне родителе
        ]);

header('Location: ' . $authorizationUrl);
```

Для получения Access Token можно использовать следующий код в обработчике, который будет находится по адресу, указаному в redirect_uri
```php
$accessToken = $apiClient->getOAuthClient()->getAccessTokenByCode($_GET['code']);
```

Пример авторизации можно посмотреть в файле examples/get_token.php

## Подход к работе с библиотекой
В библиотеке используется сервисный подход. Для каждой сущности имеется сервис.
Для каждого метода имеется свой объект коллекции и модели.
Работа с данными происходит через коллекции и методы библиотеки.

Модели и коллекции имеют методы ```toArray()``` и ```toApi()```, методы возвращают представление объекта в виде массива и в виде данных, отправляемых в API.

Также для работы с коллекциями имеют следующие методы:
1. add(BaseApiModel $model): self - добавляет модель в конец коллекции.
2. prepend(BaseApiModel $value): self - добавляет модель в начало коллекции.
3. all(): array - возвращает массив моделей в коллекции.
4. first(): ?BaseApiModel - получение первой модели в коллекции.
5. last(): ?BaseApiModel - получение последней модели в коллекции.
6. count(): int - получение кол-ва элементов в коллекции.
7. isEmpty(): bool - проверяет, что коллекция не пустая.
8. getBy($key, $value): ?BaseApiModel - получение модели по значению ключа.
9. replaceBy($key, $value, BaseApiModel $replacement): void - замена модели по значению ключа.

При работе с библиотекой необходимо не забывать о лимитах API amoCRM.
Для оптимальной работы с данными лучше всего создавать/изменять за раз не более 50 сущностей в методах, где есть пакетная обработка.

Нужно не забывать про обработку ошибок, а также не забывать о безопасности хранилища токенов. **Утечка токена грозит потерей досутпа к аккаунту.**

## Поддерживаемые методы и сервисы

Библиотека поддерживает большое количество методов API. Методы сгруппированы и объекты-сервисы. Получить объект сервиса можно вызвав необходимый метод у библиотеки, например:
```php
$leadsService = $apiClient->leads();
```

В данный момент доступны следующие сервисы:

| Сервис            | Цель сервиса                  |
|-------------------|-------------------------------|
| notes             | Примечание сущности           |
| tags              | Теги сущностей                |
| tasks             | Задачи                        |
| leads             | Сделки                        |
| contacts          | Контакты                      |
| companies         | Компании                      |
| catalogs          | Каталоги                      |
| catalogElements   | Элементы каталогов            |
| customFields      | Пользовательские поля         |
| customFieldGroups | Группы пользовательских полей |
| account           | Информация об аккаунте        |
| roles             | Роли пользователей            |
| users             | Роли юзеров                   |
| customersSegments | Сегменты покупателей          |
| events            | Список событий                |
| webhooks          | Вебхуки                       | 
| unsorted          | Неразобранное                 |
| pipelines         | Воронки сделок                |
| statuses          | Статусы сделок                |
| widgets           | Виджеты                       |
| lossReason        | Причины отказа                |
| transactions      | Покупки покупателей           |
| customers         | Покупатели                    |
| customersStatuses | Сегменты покупателя           |
| getOAuthClient    | oAuth сервис                  |
| getRequest        | Голый запросы                 |

#### Для большинства сервисов есть базовый набор методов:

1. getOne - Получить 1 сущность
    1. id (int|string) - id сущности
    2. with (array) - массив параметров with, которые поддерживает модель сервиса
    3. Результатом выполнения будет модель сущности
    ```php
    getOne($id, array $with => []);
    ```

2. get Получить несколько сущностей:
    1. filter (BaseEntityFilter) - фильтр для сущности
    2. with (array) - массив параметров with, которые поддерживает модель сервиса
    3. Результатом выполнения будет коллекция сущностей
    ```php
    get(BaseEntityFilter $filter = null, array $with = []);
    ```
   
3. addOne Создать одну сущность:
    1. model (BaseApiModel) - модель создаваемой сущности
    2. Результатом выполнения будет модель сущности
    ```php
    addOne(BaseApiModel $model);
    ```
   
4. add Создать сущности пакетно:
    1. collection (BaseApiCollection) - коллекция моделей создаваемой сущности
    2. Результатом выполнения будет коллекция моделей сущности
    ```php
    add(BaseApiCollection $collection);
    ```

5. updateOne Обновить одну сущность:
    1. model (BaseApiModel) - модель создаваемой сущности
    2. Результатом выполнения будет модель сущности
    ```php
    updateOne(BaseApiModel $model);
    ```
   
6. update Обновить сущности пакетно:
    1. collection (BaseApiCollection) - коллекция моделей создаваемой сущности
    2. Результатом выполнения будет коллекция моделей сущности
    ```php
    update(BaseApiCollection $collection);
    ```
   
7. syncOne Синхронизировать одну модель с сервером:
    1. model (BaseApiModel) - коллекция моделей создаваемой сущности
    2. with (array) - массив параметров with, которые поддерживает модель сервиса
    4. Результатом выполнения будет коллекция моделей сущности
    ```php
    syncOne(BaseApiModel $model, $with = []);
    ```
   
Не все методы доступны во всех сервисах. В случае их вызова будет выброшены Exception.

Некоторые сервисы имеют специфичные методы, ниже рассмотрим сервисы, которые имеют специфичные методы.

#### Методы доступные в сервисе ```getOAuthClient```:
1. getAuthorizeUrl получение ссылки на авторизация
    1. options (array)
        1. state (string) состояние приложения
    2. Результатом выполнения будет строка с ссылкой на авторизация приложения
    ```php
    getAuthorizeUrl(array $options = []);
    ```
   
2. getAccessTokenByCode получение аксес токена по коду авторизации
    1. code (string) код авторизации
    2. Результатом выполнения будет объект (AccessTokenInterface)
    ```php
    getAccessTokenByCode(string $code);
    ```
   
3. getAccessTokenByRefreshToken получение аксес токена по рефреш токену
    1. accessToken (AccessTokenInterface) объект аксес токена
    2. Результатом выполнения будет объект (AccessTokenInterface)
    ```php
    getAccessTokenByRefreshToken(AccessTokenInterface $accessToken);
    ```
   
4. setBaseDomain установка базового домена, куда будут отправляться запросы необходимые для работы с токенами
    1. domain (string)
    ```php
    setBaseDomain(string $domain);
    ```

5. setAccessTokenRefreshCallback установка callback, который будет вызван при обновлении аксес токена
    1. function (callable)
    ```php
    setAccessTokenRefreshCallback(callable $function);
    ```

5. getOAuthButton установка callback, который будет вызван при обновлении аксес токена
    1. options (array)
        1. state (string) состояние приложения
        2. color (string)
        3. title (string)
        4. compact (bool)
        5. class_name (string)
        6. error_callback (string)
        7. mode (string)
    2. Результатом выполнения будет строка с HTML кодом кнопки авторизации
    ```php
    getOAuthButton(array $options = []);
    ```
   
6. exchangeApiKey метод для обмена API ключа на код авторизации
    1. login - email пользователя, для которого обменивается API ключ
    2. apiKey - API ключ пользователя
    3. Код авторизации будет прислан на указанный в настройках приложения redirect_uri
    ```php
    exchangeApiKey(string $login, string $apiKey);
    ```

#### Методы связей доступны в сервисах ```leads```, ```contacts```, ```companies```, ```customers```:
1. link Привязать сущность
    1. model (BaseApiModel) - модель главной сущности
    2. links (LinksCollection|LinkModel) - коллекция или модель связи
    3. Результатом выполнения является коллекция связей (LinksCollection)
    ```php
    link(BaseApiModel $model, $linkedEntities);
    ```

2. getLinks Получить связи сущности
    1. model (BaseApiModel) - модель главной сущности
    2. Результатом выполнения является коллекция связей (LinksCollection)
    ```php
    getLinks(BaseApiModel $model);
    ```
       
3. unlink Отвязать сущность
    1. model (BaseApiModel) - модель главной сущности
    2. links (LinksCollection|LinkModel) - коллекция или модель связи
    3. Результатом выполнения является bool значение
    ```php
    unlink(BaseApiModel $model, $linkedEntities);
    ```

#### Методы удаления доступны в сервисах ```transactions```, ```lossReasons```, ```statuses```, ```pipelines```, ```customFields```, ```customFieldsGroups```, ```roles```, ```customersStatuses```:
1. delete
    1. model (BaseApiModel) - модель сущности
    2. Результатом выполнения является bool значение
    ```php
    deleteOne(BaseApiModel $model);
    ```
   
2. deleteOne
    1. collection (BaseApiCollection) - коллекция моделей сущностей
    2. Результатом выполнения является bool значение
    ```php
    deleteOne(BaseApiModel $model);
    ```

#### Методы доступные в сервисе ```account```
1. getCurrent
    1. with (array) - массив параметров with, которые поддерживает модель сервиса
    2. Результатом выполнения является модель AccountModel
    ```php
    getCurrent(array $with = []);
    ```
   

#### Методы доступные в сервисе ```unsorted```
1. addOne Создать одну сущность:
    1. model (BaseApiModel) - модель создаваемой сущности
    2. Результатом выполнения будет модель сущности
    ```php
    addOne(BaseApiModel $model);
    ```
   
2. add Создать сущности пакетно:
    1. collection (BaseApiCollection) - коллекция моделей создаваемой сущности
    2. Результатом выполнения будет коллекция моделей сущности
    ```php
    add(BaseApiCollection $collection);
    ```

3. link
    1. model (BaseApiModel) - модель неразобранного
    2. body (array) - массив дополнительной информации для привязки 
    2. Результатом выполнения будет модель LinkUnsortedModel
    ```php
    link(BaseApiModel $unsortedModel, $body = []);
    ```
   
4. accept
    1. model (BaseApiModel) - модель неразобранного
    2. body (array) - массив дополнительной информации для принятия 
    2. Результатом выполнения будет модель AcceptUnsortedModel
    ```php
    accept(BaseApiModel $unsortedModel, $body = []);
    ```

5. decline
    1. model (BaseApiModel) - модель неразобранного
    2. body (array) - массив дополнительной информации для отклонения 
    2. Результатом выполнения будет модель DeclineUnsortedModel
    ```php
    decline(BaseApiModel $unsortedModel, $body = []);
    ```

6. summary
    1. filter (BaseEntityFilter) - фильтр для сущности
    2. Результатом выполнения будет модель UnsortedSummaryModel
    ```php
    summary(BaseEntityFilter $filter);
    ```
   
#### Методы доступные в сервисе ```webhooks```
1. subscribe
    1. model (WebhookModel) - модель вебхука
    2. Результатом выполнения является модель WebhookModel
    ```php
    subscribe(WebhookModel $webhookModel);
    ```
   
2. unsubscribe
    1. model (WebhookModel) - модель вебхука
    2. Результатом выполнения является bool значение
    ```php
    unsubscribe(WebhookModel $webhookModel);
    ```

#### Методы доступные в сервисе ```widgets```
1. install
    1. model (WidgetModel) - модель виджета
    2. Результатом выполнения является модель WidgetModel
    ```php
    install(WidgetModel $widgetModel);
    ```
   
2. uninstall
    1. model (WidgetModel) - модель виджета
    2. Результатом выполнения является модель WidgetModel
    ```php
    uninstall(WidgetModel $widgetModel);
    ```

## Обработка ошибок

Вызов методов библиотеки может выбрасывать ошибки типа ```AmoCRMApiException```.
В данные момент доступны следующие типы ошибок, они все наследуют AmoCRMApiException:

|Тип                                                 |Условия                                                                                              |
|----------------------------------------------------|-----------------------------------------------------------------------------------------------------|
|AmoCRM\Exceptions\AmoCRMApiConnectExceptionException|Подключение к серверу не было выполнено                                                              |
|AmoCRM\Exceptions\AmoCRMApiErrorResponseException   |Сервер вернул ошибку на выполняемый запрос                                                           |
|AmoCRM\Exceptions\AmoCRMApiHttpClientException      |Произошла ошибка http клиента                                                                        |
|AmoCRM\Exceptions\AmoCRMApiNoContentException       |Сервер вернул код 204 без результата, ничего страшного не произошло, просто нет данных на ваш запрос |
|AmoCRM\Exceptions\AmoCRMApiTooManyRedirectsException|Слишком много редиректов (в нормальном режиме не выкидывается)                                       |
|AmoCRM\Exceptions\AmoCRMoAuthApiException           |Ошибка в oAuth клиенте                                                                               |
|AmoCRM\Exceptions\BadTypeException                  |Передан не верный тип данных                                                                         |
|AmoCRM\Exceptions\InvalidArgumentException          |Передан не верный аргумент                                                                           |
|AmoCRM\Exceptions\NotAvailableForActionException    |Метод не доступен для вызова                                                                         |

У выброшенных Exception есть следующие методы:
1. ```getErrorCode()```
2. ```getTitle()```
3. ```getLastRequestInfo()```
4. ```getDescription()```

У ошибки типа AmoCRMApiErrorResponseException есть метод ```getValidationErrors()```, который вернет ошибки валидации входящих данных.

## Фильтры

```todo```

## Примеры
В рамках данного репозитория имеется папка examples с различными примерами.

Для их работы необходимо добавить в неё файл .env со следующим содержимым, указав ваши значения:
```dotenv
CLIENT_ID=
CLIENT_SECRET=
CLIENT_REDIRECT_URI=
```

Затем вы можете поднять локальный сервер командой ```composer serve```. После конфигурацию необходимо перейти в браузере на страницу
```http://localhost:8181/examples/get_token.php``` для получения Access Token.
Для получения доступа к вашему локальному серверу из вне можно использовать сервис ngrok.io. 

После авторизации вы можете проверить работу примеров, обращаясь к ним из браузера. Стоит отметить, что для корректной работы примеров
необходимо проверить ID сущностей в них.

## License

MIT
