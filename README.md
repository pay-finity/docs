# API Документация

## Введение

Добро пожаловать в документацию по API **Pay-Finity**. Наш API позволяет вам программно взаимодействовать с нашими сервисами, что обеспечивает бесшовную интеграцию с вашими приложениями.

## Аутентификация и Подпись

Для обеспечения безопасности нашего API все запросы должны быть подписаны с использованием подписи (Signature), сгенерированной с помощью приватного ключа (PrivateKey), который мы предоставляем нашим клиентам. Подпись используется для проверки целостности и подлинности запросов.

### Генерация Подписи

Подпись генерируется с использованием алгоритма HMAC-SHA512. Ниже приведена пример функции генерирования подписи (Signature):
#### Golang
```go
func GenerateSignature(secret string, message string) string {
    mac := hmac.New(sha512.New, []byte(secret))
    mac.Write([]byte(message))
    return hex.EncodeToString(mac.Sum(nil))
}
```

### Формирование Сообщения

Сообщение (message), используемое для генерации подписи, формируется по-разному в зависимости от метода HTTP запроса:

- **GET Запрос**: Сообщение составляется из URL и закодированных параметров запроса.
- **POST Запрос**: Сообщение включает URL и JSON тело запроса.

#### Примеры кода на разных языках для формирования сообщения:
- **exp** - время жизни запроса в формате UNIX, которое вы вставляетя в заголовок"Expires" (о заголовках можете прочитать ниже)
- **secret** - приватный ключ (PrivateKey), переданный вам поддержкой Pay Finity
- **body** - тело запроса, в случае GET оно пустое (nil/null), обязательно чтобы JSON ключи и query параметры шли в алфавитном порядке!!!
#### <span style="color:red"> *Обязательно JSON ключи в теле (body) и query параметры запроса должны идти в алфавитном порядке!!!* </span>

#### Golang
```go
func generateMessage(r *resty.Request, body []byte, exp, secret string) (string, error) {
	message := r.URL
	if r.Method == http.MethodGet {
		message += r.QueryParam.Encode()
	} else {
		data := make(map[string]any)
		if body != nil {
			if err := json.Unmarshal(body, &data); err != nil {
				return "", err
			}

			jsonBody, err := json.Marshal(data)
			if err != nil {
				return "", err
			}

			message += string(jsonBody)
		}
	}
	return GenerateSignature(secret, message+exp), nil
}
```

### Примеры

Для GET запроса на `https://pay-finity.com/api/v1/account/transactions?limit=3&page=1&type=IN`:

- **URL**: `/api/v1/account/transactions`
- **Параметры запроса**: `limit=3&page=1&type=IN`
- **Expires**: `1721585422` - время жизни запроса в формате UNIX по UTC, рекомендуем добавлять к текущему времени 3-5 минут
- **Сообщение - message (для подписи)**: `/api/v1/account/transactionslimit=3&page=1&type=IN1721585422`

Для POST запроса на `https://pay-finity.com/api/v1/payment` с телом: `
{
  "clientID": "test",
  "currency": "RUB",
  "callbackURL": "http://test.com/test1",
  "description": "test",
  "amount": "1000"
}
`:

- **URL**: `/api/v1/payment`
- **Тело запроса**: `
{
  "clientID": "test",
  "currency": "RUB",
  "callbackURL": "https://test.com/test1",
  "description": "test",
  "amount": "1000"
}`
- **Сообщение - message (для подписи), JSON ключи идут в афавитном порядке**: `/api/v1/payment{"amount":"1000","callbackURL":"https://test.com/test1","clientID":"test","currency":"RUB","description":"test"}`

## Заголовки Запросов

Каждый запрос к API должен включать следующие заголовки:

- **Content-Type**: `application/json`
- **Public-Key**: Ваш публичный ключ, предоставленный Pay-Finity.
- **Expires**: Время истечения жизни запроса.
- **Signature**: Подпись HMAC-SHA512, сгенерированная с использованием вашего закрытого ключа и сообщения.

### Пример заголовков

```http
Content-Type: application/json
Expires: 1717025133
Public-Key: testPublicKey
Signature: 2816894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
```

## Выполнение Запроса

Ниже приведен пример того, как выполнить GET запрос к API Pay-Finity:

### GET Запрос

```http
GET /api/v1/account/transactions?limit=5&page=1 HTTP/1.1
Host: pay-finity.com
Content-Type: application/json
Expires: 1717025133
Public-Key: testPublicKey
Signature: 2816894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
```

### POST Запрос

Ниже приведен пример того, как выполнить POST запрос к API Pay-Finity:

```http
POST /api/v1/payment HTTP/1.1
Host: pay-finity.com
Expires: 1717025134
Public-Key: testPublicKey
Signature: 3336894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8678

{
  "clientID": "test",
  "currency": "RUB",
  "callbackURL": "https://test.com/test1",
  "description": "test",
  "amount": "1000"
}
```

## Ответ

Все ответы от API Pay-Finity будут в формате JSON. Ответ включает в себя:

- **success** - Статус запроса.
- **data** - Данные, возвращенные API (только в случая успешного ответа от API).
- **error** - Данные, по возникшей ошибки (только в случая ошибочного ответа от API).

### Пример ответа

```json
{
  "success": true,
  "data": {
    "balance": {
      "payment": {
        "currency": "RUB",
        "available": "7002.95",
        "frozen": "4500.47"
      },
      "payout": {
        "currency": "RUB",
        "available": "10000.23",
        "frozen": "7801.68"
      }
    }
  }
}
```

## Обработка Ошибок

В случае ошибки ответ будет включать:

- **message**: Описание ошибки.
- **code**: Код ошибки.

### Пример ответа с ошибкой

```json
{
  "success": false,
  "error": {
      "message": "Invalid Signature",
      "code": 60006
  }
}
```

## Коды ошибок

| Код ошибки | Сообщение                               | HTTP статус код |
|------------|----------------------------------------|------------------|
| 10000      | unauthorized                           | 401              |
| 20000      | wrong input                            | 400              |
| 20001      | can't bind body to request model       | 422              |
| 20002      | can't bind query parameters            | 422              |
| 20003      | failed to parse key                    | 422              |
| 20004      | signature header value missing or malformed | 400           |
| 20005      | public-Key header value missing or malformed | 400         |
| 20006      | timestamp header value missing or outdated | 400          |
| 20012      | invalid query params                   | 400              |
| 20013      | empty receipt                          | 400              |
| 20014      | large file size                        | 400              |
| 20015      | conflict                               | 409              |
| 20016      | empty trackerID                        | 400              |
| 20200      | token not found                        | 404              |
| 30000      | forbidden                              | 403              |
| 30001      | no access to requested session         | 403              |
| 30002      | requested sessions has expired        | 403              |
| 30003      | user doesn't exists                   | 403              |
| 30004      | zero balance                           | 403              |
| 30005      | not enough balance                     | 402              |
| 30006      | amount less than min                   | 400              |
| 30007      | amount greater than max                | 400              |
| 40000      | internal error                         | 500              |
| 60000      | ticker doesnt exists                   | 400              |
| 60001      | ticker type doesnt exists              | 400              |
| 60002      | invalid status                         | 400              |
| 60003      | empty Public-Key                       | 401              |
| 60004      | empty Expires                          | 401              |
| 60005      | empty Signature                        | 401              |
| 60006      | invalid Signature                      | 401              |
| 60007      | request timeout                        | 408              |
| 60008      | invalid Public-Key                     | 400              |
| 60009      | empty clientID                         | 400              |
| 60010      | clientID already exists                | 409              |
| 60011      | payment doesn't exists                 | 404              |
| 60012      | payment is finalized                   | 409              |
| 60013      | type doesnt exists                     | 400              |
| 60014      | bank doesnt exists                     | 400              |


## API запросы

### Получение актуального баланса (GET)
```http
GET /api/v1/account/balance HTTP/1.1
Host: pay-finity.com
Content-Type: application/json
Expires: 1717025135
Public-Key: testPublicKey
Signature: 2216894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
```
#### Пример успешного ответа
```json
{
  "success": true,
  "data": {
    "balance": {
      "payment": {
        "currency": "RUB",
        "available": "8355.95",
        "frozen": "570"
      },
      "payout": {
        "currency": "RUB",
        "available": "0",
        "frozen": "0"
      }
    }
  }
}
```
Струтура balance:
- **currency** - направление, всегда RUB
- **payment** - депозитный баланс
- **payout** - выплатной баланс
- **available** - доступный баланс
- **frozen** - замороженный баланс



### Получение актуальных ставок (GET)
```http
GET /api/v1/account/commissions HTTP/1.1
Host: pay-finity.com
Content-Type: application/json
Expires: 1717025144
Public-Key: testPublicKey
Signature: 4216894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
```
#### Пример успешного ответа
```json
{
  "success": true,
  "data": {
    "payment": [
      {
        "bank": "ANY_BANK",
        "currency": "RUB",
        "percent": "8",
        "minAmount": "1000",
        "maxAmount": "120000"
      }
    ],
    "payout": [
      {
        "bank": "ANY_BANK",
        "currency": "RUB",
        "percent": "7",
        "minAmount": "4000",
        "maxAmount": "100000"
      },
      {
        "bank": "SBER",
        "currency": "RUB",
        "percent": "6",
        "minAmount": "3000",
        "maxAmount": "100000"
      },
      {
        "bank": "SBP",
        "currency": "RUB",
        "percent": "5",
        "minAmount": "3000",
        "maxAmount": "100000"
      }
    ]
  }
}
```
Струтура rates:
- **payment** - комиссии по приему
- **payout** - комиссии по выплатам
- **bank** - банк
- **currency** - направление, всегда RUB
- **minAmount** - минимальная сумма для пополнения/выплаты
- **maxAmount** - максимальная сумма для пополнения/выплаты
- **percent** - процентная комиссия



### Создание транзакции по приему, используются все банки (POST)
```http
POST /api/v1/payment HTTP/1.1
Host: pay-finity.com
Content-Type: application/json
Expires: 1717025205
Public-Key: testPublicKey
Signature: nzxk21jl94fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e180e757lk4nm

{
  "clientID": "test",
  "currency": "RUB",
  "callbackURL": "https://test.com/test1",
  "description": "test payment",
  "amount": "1000",
  "type": "CARD"
}
```

Струтура тела запроса:
- **clientID** - уникальный идентификатор транзакции в вашей системе (обязательный)
- **currency** - направление, всегда RUB  (обязательный)
- **amount** - сумма транзакции  (обязательный)
- **callbackURL** - ваш URL, на который будет приходить оповещение об изменении статуса транзакции (опциональный)
- **description** - описание транзакции (опциональный)
- **type** - тип пополнений (опциональный, по дефолту CARD), возможные значения CARD - перевод по карте и SBP - перевод через СБП

#### Пример успешного ответа
```json
{
  "success": true,
  "data": {
    "trackerID": "23w324nn3754aa7aaa0ffe1fe154a64119bfc2f18804809b7c207d81a2aab27",
    "currency": "RUB",
    "amount": "1000",
    "cardNumber": "2200188977146220",
    "bank": "ALFA",
    "SBPPhoneNumber": "+79963614478",
    "holder": "Ivanov Ivan"
  }
}
```
Струтура ответа:
- **trackerID** - уникальный идентификатор транзакции в нашей стороне
- **currency** - направление, всегда RUB
- **cardNumber** - номер карты, на который нужно совершить перевод средств
- **bank** - банк на который нужно совершить перевод средств
- **updatedTime** - время обновления транзакции
- **SBPPhoneNumber** - номер телефона для перевода средств через СБП
- **holder** - ФИО держателя карты



### Создание выплаты, используются SBER, TINKOFF, ANY_BANK (POST)
```http
POST /api/v1/payout HTTP/1.1
Host: pay-finity.com
Content-Type: application/json
Expires: 1717025207
Public-Key: testPublicKey
Signature: nnd5721jl94fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbksadd234pqn144

{
  "clientID": "test2",
  "bank": "TINKOFF",
  "callbackURL": "http://test.com/test2",
  "description": "test payout",
  "amount": "5500",
  "receiver": "2200555555555555",
  "type": "CARD"
}
```

Струтура тела запроса:
- **clientID** - уникальный идентификатор транзакции в вашей системе (обязательный)
- **bank** - банк, на который вы хотите вывести средства, доступны только ANY_BANK (межбанк), SBER, TINKOFF (обязательный)
- **amount** - сумма транзакции  (обязательный)
- **receiver** - номер карты, на которую вы хотите вывести средства (обязательный)
- **type** - тип выплаты, CARD или SBP (обязательный)
- **callbackURL** - ваш URL, на который будет приходить оповещение об изменении статуса транзакции (опциональный)
- **description** - описание транзакции (опциональный)

#### Пример успешного ответа
```json
{
  "success": true,
  "data": {
    "trackerID": "llasdq2nn3754aa7aaa0ffe1fe154a64119bfc2f18804809b7c207d81a2aazkl",
    "currency": "RUB",
    "amount": "1100",
    "commission": "100",
    "status": "PENDING"
  }
}
```
Струтура ответа:
- **trackerID** - уникальный идентификатор транзакции в нашей стороне
- **currency** - направление, всегда RUB
- **amount** - сумма выплата с учетом комиссии
- **commission** - комиссия за выплату
- **status** - статус транзакции



### Получение транзакции по trackerID (GET)
```http
GET /api/v1/account/transaction?trackerID=8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1 HTTP/1.1
Host: pay-finity.com
Content-Type: application/json
Expires: 1717025201
Public-Key: testPublicKey
Signature: 234jjl94fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e180e7575c5dba2
```

#### Пример успешного ответа
```json
{
  "success": true,
  "data": {
	"clientID": "1wswssxfxxf22qqffs",
	"trackerID": "8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1",
	"type": "OUT",
	"currency": "RUB",
	"status": "SUCCESS",
	"createdTime": "2024-05-15T23:55:43.035172Z",
	"updatedTime": "2024-05-16T03:17:37.331142+03:00",
	"amount": "6802",
	"commission": "340.1"
  }
}
```
Струтура transactions:
- **clientID** - уникальный идентификатор транзакции в вашей системе
- **trackerID** - уникальный идентификатор транзакции в нашей стороне
- **currency** - направление, всегда RUB
- **status** - статус транзакции (SUCCESS - успешная, PENDING - в ожидании, ERROR - ошибочная)
- **type** - тип транзакции (выплата - OUT, пополнение - IN)
- **createdTime** - время создания транзакции
- **updatedTime** - время обновления транзакции
- **amount** - сумма транзакции
- **commission** - комиссия за транзакцию

  

### Получение массива транзакций по заданным параметрам (GET)
```http
GET /api/v1/account/transactions?limit=5&page=1&type=IN HTTP/1.1
Host: pay-finity.com
Content-Type: application/json
Expires: 1717025144
Public-Key: testPublicKey
Signature: 4216894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
```
#### Возможные параметры, по которым можно фильтровать транзакции
- **limit** - количество транзакций, которые вы хотите получить (обязательный)
- **page** - номер страницы, с которой вы хотите получить транзакции (обязательный)
- **currency** - направление, всегда RUB (опциональный)
- **status** - статус транзакции (SUCCESS - успешная, PENDING - в ожидании, ERROR - ошибочная) (опциональный)
- **type** - тип транзакции (выплата - OUT, пополнение - IN) (опциональный)
- **clientID** - уникальный id транзакции в вашей системе (опциональный)
- **amountMin** - минимальная сумма (опциональный)
- **amountMax** - максимальная сумма (опциональный)
- **dateFrom** - с какого времени (опциональный)
- **dateTo** - до какого времени (опциональный)

#### Пример успешного ответа
```json
{
  "success": true,
  "data": {
    "transactions": [
      {
        "clientID": "1wswssxfxxf22qqffs",
	"trackerID": "8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1",
        "type": "OUT",
        "currency": "RUB",
        "status": "SUCCESS",
        "createdTime": "2024-05-15T23:55:43.035172Z",
        "updatedTime": "2024-05-16T03:17:37.331142+03:00",
        "amount": "6802",
        "commission": "340.1"
      },
      {
        "clientID": "b122cckaaoz87",
	"trackerID": "813hbeo32279942e15075cc6eb0ad05d430c242a750b140db1446f774wj3",
        "type": "IN",
        "currency": "RUB",
        "status": "SUCCESS",
        "createdTime": "2024-05-14T23:44:57.286611Z",
        "updatedTime": "2024-05-15T02:47:54.904398+03:00",
        "amount": "600",
        "commission": "30"
      },
      {
        "clientID": "yeahbzoz87",
	"trackerID": "ojdsan435k9942e15075cc6eb0ad05d430c242a750b140db14123n934naq",
        "type": "OUT",
        "currency": "RUB",
        "status": "PENDING",
        "createdTime": "2024-05-14T23:09:37.694018Z",
        "updatedTime": "2024-05-15T02:09:37.694018+03:00",
        "amount": "600",
        "commission": "30"
      },
      {
        "clientID": "ortndc81",
	"trackerID": "bzx3k1qedak614279942e15075cc6eb0ad05d430c242a750b140db1412mmxcvl1",
        "type": "IN",
        "currency": "RUB",
        "status": "ERROR",
        "createdTime": "2024-05-09T01:31:50.858344Z",
        "updatedTime": "2024-05-09T04:32:12.894818+03:00",
        "amount": "1000",
        "commission": "35"
      },
      {
        "clientID": "ijm1sibsib221ad",
	"trackerID": "zlp23ommf45k9942e15075cc6eb0ad05d430c242a750b140db141qsmc03",
        "type": "IN",
        "currency": "RUB",
        "status": "SUCCESS",
        "createdTime": "2024-05-27T00:15:59.562019Z",
        "updatedTime": "2024-05-27T03:25:46.861819+03:00",
        "amount": "502",
        "commission": "25.1"
      }
    ],
    "pages": 1
  }
}
```
Струтура transactions:
- **clientID** - уникальный идентификатор транзакции в вашей системе
- **trackerID** - уникальный идентификатор транзакции в нашей стороне
- **currency** - направление, всегда RUB
- **status** - статус транзакции (SUCCESS - успешная, PENDING - в ожидании, ERROR - ошибочная)
- **type** - тип транзакции (выплата - OUT, пополнение - IN)
- **createdTime** - время создания транзакции
- **updatedTime** - время обновления транзакции
- **amount** - сумма транзакции
- **commission** - комиссия за транзакцию
- **pages** - количество страниц с транзакциями


## Отправка колбэка об изменении статуса транзакции (POST)
На указанный Вами callbcakURL, при изменении статуса транзакции будет отправлен POST запрос с trackerID, по которому вы можете получить информацию о своей транзакции, отправив запрос на получение информации по транзакции. Пример тела запроса колбэка:
```json
{
  "trackerID": "8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1",
}
```
  
*Мы надеемся, что эта документация поможет вам интегрироваться с API Pay-Finity. Если у вас есть вопросы, пожалуйста, свяжитесь с нашей службой поддержки.*
