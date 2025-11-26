# API Документация

## Введение

Добро пожаловать в документацию по API **Pay-Finity**. Наш API позволяет вам программно взаимодействовать с нашими сервисами, что обеспечивает бесшовную интеграцию с вашими приложениями.

## Аутентификация и Подпись

Для обеспечения безопасности нашего API все запросы должны быть подписаны с использованием подписи (Signature), сгенерированной с помощью приватного ключа (PrivateKey), который мы предоставляем нашим клиентам. Подпись используется для проверки целостности и подлинности запросов.

### Генерация Подписи

Подпись генерируется с использованием алгоритма HMAC-SHA512. Ниже приведен пример функции генерирования подписи (Signature):
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
- **Expires**: `1721585422` - время жизни запроса в формате UNIX по UTC, рекомендуем добавлять к текущему времени 3-5 минут
- **Тело запроса**: `
{
  "clientID": "test",
  "currency": "RUB",
  "callbackURL": "https://test.com/test1",
  "description": "test",
  "amount": "1000"
}`
- **Сообщение - message (для подписи), JSON ключи идут в афавитном порядке**: `/api/v1/payment{"amount":"1000","callbackURL":"https://test.com/test1","clientID":"test","currency":"RUB","description":"test"}1721585422`

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
      "payment": [
        {
          "currency": "RUB",
          "available": "844",
          "frozen": "12000"
        },
        {
          "currency": "UZS",
          "available": "198999.78",
          "frozen": "65000"
        }
      ],
      "payout": [
        {
          "currency": "RUB",
          "available": "97930.461",
          "frozen": "923.05"
        },
        {
          "currency": "UZS",
          "available": "700000",
          "frozen": "2000.22"
        }
      ]
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
GET /api/v1/account/balances HTTP/1.1
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
      "payment": [
        {
          "currency": "RUB",
          "available": "844",
          "frozen": "12000"
        },
        {
          "currency": "UZS",
          "available": "198999.78",
          "frozen": "65000"
        }
      ]
    }
  }
}
```
Струтура balance:
- **currency** - валюта
- **payment** - баланс
- **available** - доступный баланс
- **frozen** - замороженный баланс


### Получение активных банков для передачи на payIn/payOut (GET)
```http
GET /api/v1/account/banks HTTP/1.1
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
    "banks": [
      {
        "ticker": "SBER",
        "currency": "RUB"
      },
      {
        "ticker": "TINKOF",
        "currency": "RUB"
      },
      {
        "ticker": "ALFA",
        "currency": "RUB"
      },
      {
        "ticker": "UZ_CARD",
        "currency": "UZS"
      },
    ]
  }
}
```

### Получение активных валют для передачи на прием/выплаты (GET)
```http
GET /api/v1/account/currencies HTTP/1.1
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
    "currencies": [
      {
        "name": "RUB"
      },
      {
        "name": "UZS"
      },
      {
        "name": "TRY"
      },
      {
        "name": "AZN"
      }
    ]
  }
}
```


### Создание транзакции payIn (POST)
```http
POST /api/v1/payment HTTP/1.1
Host: pay-finity.com
Content-Type: application/json
Expires: 1717025205
Public-Key: testPublicKey
Signature: nzxk21jl94fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e180e757lk4nm

{
  "bank": "ANY_BANK",
  "clientID": "test",
  "currency": "RUB",
  "callbackURL": "https://test.com/test1",
  "description": "test payment",
  "amount": "1000",
  "type": "CARD",
  "merchantUserID": "test_user",
  "merchantUserIP": "192.168.1.1"
}
```

Струтура тела запроса:
- **clientID** - уникальный идентификатор транзакции в вашей системе (обязательный)
- **currency** - валюта: RUB, UZS, AZN, TRY (обязательный)
- **amount** - сумма транзакции в нативной валюте, например если создаете на 30000.53 UZS, то отправляете 30000.53 (обязательный)
- **callbackURL** - ваш URL, на который будет приходить оповещение об изменении статуса транзакции (опциональный)
- **description** - описание транзакции (опциональный)
- **type** - тип пополнений (опциональный, по дефолту CARD), возможные значения CARD - перевод по карте и SBP - перевод через СБП, ACCOUNT - перевод через банковский счет, CROSSBORDER_CARD - трансграничный перевод по карте, CROSSBORDER_SBP - трансграничный перевод по СБП, NSPK - НСПК
- **bank** - название банка, на который хотите совершить перевод средств, по умолчанию если не передавать поле, то используется ANY_BANK (опциональный)
- **merchantUserID** - id конечного пользователя (обязательный)
- **merchantUserIP** - IP конечного пользователя (оцпиональный)

#### Пример успешного ответа
```json
{
  "success": true,
  "data": {
    "trackerID": "23w324nn3754aa7aaa0ffe1fe154a64119bfc2f18804809b7c207d81a2aab27",
    "currency": "RUB",
    "amount": "1000",
    "commission": "0",
    "cardNumber": "2200188977146220",
    "accountNumber": "40817810099910004312",
    "bank": "ALFA",
    "SBPPhoneNumber": "+79963614478",
    "holder": "Ivanov Ivan",
    "nspkURL": "https://qr.nspk.ru/AS2A003H1TLHVARJ8TDOM6SQ7MNNTKQB?type=01&bank=100000000052&crc=8144",
    "country": "Таджикистан"
  }
}
```
Струтура ответа:
- **trackerID** - уникальный идентификатор транзакции в нашей стороне
- **currency** - валюта: RUB, UZS, AZN, TRY
- **cardNumber** - номер карты, на который нужно совершить перевод средств
- **bank** - банк на который нужно совершить перевод средств
- **updatedTime** - время обновления транзакции
- **SBPPhoneNumber** - номер телефона для перевода средств через СБП
- **accountNumber** - номер счета для перевода средств через номер счета
- **holder** - ФИО держателя карты
- **nspkURL** - ссылка для оплаты по методу НСПК
- **country** - страна для перевода в случае трансграничных методов оплаты
- **commission** - комиссия в запрошенной валюте 0, после перевода в успех в запросе на получении информации по заявке будет актуальная информация



### Создание транзакции payOut (POST)
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
  "currency": "RUB",
  "amount": "5500",
  "receiver": "2200555555555555",
  "type": "CARD",
  "merchantUserID": "test_user",
  "merchantUserIP": "192.168.1.1"
}
```

Струтура тела запроса:
- **clientID** - уникальный идентификатор транзакции в вашей системе (обязательный)
- **bank** - банк, на который вы хотите вывести средства, доступны только ANY_BANK (межбанк), SBER, TINKOFF, ... (обязательный)
- **amount** - сумма транзакции в нативной валюте, например если создаете на 30000.53 UZS, то отправляете 30000.53 (обязательный)
- **currency** - валюта: RUB, UZS, AZN, TRY (обязательный)
- **receiver** - номер карты, на которую вы хотите вывести средства (обязательный)
- **type** - тип выплаты, CARD, SBP, ACCOUNT (обязательный)
- **callbackURL** - ваш URL, на который будет приходить оповещение об изменении статуса транзакции (опциональный)
- **description** - описание транзакции (опциональный)
- **merchantUserID** - id конечного пользователя (обязательный)
- **merchantUserIP** - IP конечного пользователя (оцпиональный)

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
- **currency** - валюта: RUB, UZS, AZN, TRY
- **amount** - сумма выплата с учетом комиссии в нативной валюте
- **commission** - комиссия за выплату в нативной валюте
- **status** - статус транзакции



### Получение информации payIn по trackerID (GET)
<span style="color:red; font-weight:bold">
Если вы получили ошибку при запросе информации по ордеру, то не переводите ордер в статус отмены,
а попробуйте снова запросить, чтобы получить актуальный статус и сумму.
</span>

```http
GET /api/v1/account/transaction?trackerID=8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1 HTTP/1.1
Host: pay-finity.com
Content-Type: application/json
Expires: 1717025201
Public-Key: testPublicKey
Signature: 234jjl94fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e180e7575c5dba2
```

Возможные параметры запроса:
- **trackerID** - уникальный идентификатор транзакции в нашей стороне (обязательный)

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
	"commission": "340.1",
	"holder": "Ivanov Ivan",
	"receiver": "2200555555555555"
  }
}
```
Струтура transactions:
- **clientID** - уникальный идентификатор транзакции в вашей системе
- **trackerID** - уникальный идентификатор транзакции в нашей стороне
- **currency** - валюта: RUB, UZS, AZN, TRY
- **status** - статус транзакции (SUCCESS - успешная, PENDING - в ожидании, ERROR - ошибочная)
- **type** - тип транзакции (выплата - OUT, пополнение - IN)
- **createdTime** - время создания транзакции
- **updatedTime** - время обновления транзакции
- **amount** - сумма транзакции в нативной валюте
- **commission** - комиссия за транзакцию в нативной валюте
- **holder** - ФИО держателя карты
- **receiver** - реквизит, на который выводились средства в случае payOut (выплаты)


### Получение информации payOut по trackerID (GET)
<span style="color:red; font-weight:bold">
Если вы получили ошибку при запросе информации по ордеру, то не переводите ордер в статус отмены,
а попробуйте снова запросить, чтобы получить актуальный статус и сумму.
</span>

```http
GET /api/v1/account/payout?trackerID=8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1 HTTP/1.1
Host: pay-finity.com
Content-Type: application/json
Expires: 1717025201
Public-Key: testPublicKey
Signature: 234jjl94fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e180e7575c5dba2
```

Возможные параметры запроса:
- **trackerID** - уникальный идентификатор транзакции в нашей стороне (обязательный)

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
	"commission": "340.1",
	"holder": "Ivanov Ivan",
	"receiver": "2200555555555555"
  }
}
```
Струтура transactions:
- **clientID** - уникальный идентификатор транзакции в вашей системе
- **trackerID** - уникальный идентификатор транзакции в нашей стороне
- **currency** - валюта: RUB, UZS, AZN, TRY
- **status** - статус транзакции (SUCCESS - успешная, PENDING - в ожидании, ERROR - ошибочная)
- **type** - тип транзакции (выплата - OUT, пополнение - IN)
- **createdTime** - время создания транзакции
- **updatedTime** - время обновления транзакции
- **amount** - сумма транзакции в нативной валюте
- **commission** - комиссия за транзакцию в нативной валюте
- **holder** - ФИО держателя карты
- **receiver** - реквизит, на который выводились средства в случае payOut (выплаты)


## Отправка колбэка об изменении статуса транзакции (POST)
На указанный Вами callbcakURL, при изменении статуса транзакции будет отправлен POST запрос с trackerID, по которому вы можете получить информацию о своей транзакции, отправив запрос на получение информации по транзакции. Пример тела запроса колбэка:
```json
{
  "trackerID": "8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1",
}
```
  
*Мы надеемся, что эта документация поможет вам интегрироваться с API Pay-Finity. Если у вас есть вопросы, пожалуйста, свяжитесь с нашей службой поддержки.*
