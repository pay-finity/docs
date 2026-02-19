# API Документация

## Введение

Добро пожаловать в документацию по API **Pay-Finity**. Наш API позволяет вам программно взаимодействовать с нашими сервисами, что обеспечивает бесшовную интеграцию с вашими приложениями.
Базовый URL для отправки запросов = https://api.payfinity.pro

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

Для GET запроса на `https://api.payfinity.pro/v1/account/transactions?limit=3&page=1&type=IN`:

- **URL**: `/v1/account/transactions`
- **Параметры запроса**: `limit=3&page=1&type=IN`
- **Expires**: `1721585422` - время жизни запроса в формате UNIX по UTC, рекомендуем добавлять к текущему времени 3-5 минут
- **Сообщение - message (для подписи)**: `/v1/account/transactionslimit=3&page=1&type=IN1721585422`

Для POST запроса на `https://api.payfinity.pro/v1/payment` с телом: `
{
  "clientID": "test",
  "currency": "RUB",
  "callbackURL": "http://test.com/test1",
  "description": "test",
  "amount": "1000"
}
`:

- **URL**: `/v1/payment`
- **Expires**: `1721585422` - время жизни запроса в формате UNIX по UTC, рекомендуем добавлять к текущему времени 3-5 минут
- **Тело запроса**: `
{
  "clientID": "test",
  "currency": "RUB",
  "callbackURL": "https://test.com/test1",
  "description": "test",
  "amount": "1000"
}`
- **Сообщение - message (для подписи), JSON ключи идут в афавитном порядке**: `/v1/payment{"amount":"1000","callbackURL":"https://test.com/test1","clientID":"test","currency":"RUB","description":"test"}1721585422`

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
GET /v1/account/transactions?limit=5&page=1 HTTP/1.1
Host: api.payfinity.pro
Content-Type: application/json
Expires: 1717025133
Public-Key: testPublicKey
Signature: 2816894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
```

### POST Запрос

Ниже приведен пример того, как выполнить POST запрос к API Pay-Finity:

```http
POST /v1/payment HTTP/1.1
Host: api.payfinity.pro
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
GET /v1/account/balances HTTP/1.1
Host: api.payfinity.pro
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
GET /v1/account/banks HTTP/1.1
Host: api.payfinity.pro
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
      }
    ]
  }
}
```

### Получение активных валют для передачи на прием/выплаты (GET)
```http
GET /v1/account/currencies HTTP/1.1
Host: api.payfinity.pro
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
      },
      {
        "name": "KZS"
      },
      {
        "name": "TJS"
      }
    ]
  }
}
```


### Создание транзакции payIn (POST)
```http
POST /v1/payment HTTP/1.1
Host: api.payfinity.pro
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
  "merchantUserIP": "192.168.1.1",
  "firstName": "John",
  "lastName": "Milton",
  "tcid": "12345678901",
  "card": {
	  "pan": "4004241010524322",
	  "year": "2045",
	  "month": "11",
	  "cvv": "912",
	  "holder": "Dustin Poirier"
  },
  "registeredAt": 1719937021,
  "userAgent": "Mozilla/5.0...",
  "fingerprint": "fbb77b9f4265b18538e66cac5a37c6410dc2cdd7f0cddfde6eda25aa10df669b",
  "merchantURL": "https://merchant-url.com",
}
```

Струтура тела запроса:
- **clientID** - уникальный идентификатор транзакции в вашей системе (обязательный)
- **currency** - валюта: RUB, UZS, AZN, TRY, INR (обязательный)
- **amount** - сумма транзакции в нативной валюте, например если создаете на 30000.53 UZS, то отправляете 30000.53 (обязательный)
- **callbackURL** - ваш URL, на который будет приходить оповещение об изменении статуса транзакции (опциональный)
- **description** - описание транзакции (опциональный)
- **type** - тип пополнений (опциональный, по дефолту CARD), возможные значения CARD - перевод по карте и SBP - перевод через СБП, ACCOUNT - перевод через банковский счет, CROSSBORDER_CARD - трансграничный перевод по карте, CROSSBORDER_SBP - трансграничный перевод по СБП, NSPK - НСПК, UPI - переводы через UPI (только INR), IMPS - переводы по IMPS (только INR), ECOM - перевод по ECOM
- **bank** - наименование банка, на который хотите совершить перевод средств, по умолчанию если не передавать поле, то используется ANY_BANK (опциональный)
- **merchantUserID** - id конечного пользователя (обязательный)
- **merchantUserIP** - IP конечного пользователя (оцпиональный)
- **firstName** - имя (обязательный только для TRY)
- **lastName** - фамилия (обязательный только для TRY)
- **tcid** - Turkish Identification Number (обязательный только для TRY)
- **card** - данные банковской карты (обязательно для ECOM)
	- **pan** — номер банковской карты (PAN)
 	- **year** — год окончания срока действия карты в формате `YYYY`
  	- **month** — месяц окончания срока действия карты в формате `MM`
	- **cvv** — CVV / CVC код карты
	- **holder** — держатель карты
- **registeredAt** — время регистрации пользователя в вашей системе в формате Unix timestamp (опциональный)
- **fingerprint** — уникальный цифровой отпечаток устройства пользователя (browser/device fingerprint), используется для antifraud-проверок (опциональный)
- **merchantURL** — URL сайта или сервиса мерчанта, с которого был инициирован платёж (опциональный)
- **userAgent** — строка User-Agent браузера или клиента пользователя, с которого был выполнен запрос (опциональный)

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
    "country": "Таджикистан",
	"paymentURL": "https://acs5.sbrf.ru:443/ncs_03/acs/pareq/05_0321d90c9450d44214a3a6d35e2854a0b4?MD=6c6cb5e6-35df-7137-9f57-f1d90238457c&PaReq=eJxVUttuwjA2%2FRXUSXsrubRNCzOZ2GCXhwIaoEl7C60Z3Ub40hbovn4Jl10iRfKxnWP7OHC9X3%2B0tqjLrMh7DmtTp4V5UqRZ%2Ftpz5rM7N3KuJcxWGnEwxaTWKCHGslSv2MrSnhN2hJci466fKnR9ynw3wgRdEaXLJQp%2FoShzJEz6T%2Fgp4VRImjptDuQMDaNOViqvJKjk8%2BZxJH0eCkqBnCCsUT8OJD0eZq4IfAHk6IZcrVFOF6gvLxgNwquVanCbNQjkEIGkqPNKN1L4HpAzgFp%2FyFVVbbqE7Ha7dmneL1T%2B3tY1EBsD8tvWpLZWabj2WSrjWZ%2FHX0Mvfhvy0dc8GM3mLB4kPB6894DYDEhVhZJT7lFOOy0WdBnr%2BiGQgx%2FU2jYhAx4chjwi2Ngi%2FX%2Bhvy4w6muznPMcZwS43xQ5mgwj6Y8N5Lfn2wcrbFIZrTqhYCKKPEabl%2BJ5eTd%2FakbNsD8eT%2B%2BFlfuQZBkzIxEP2ZHSAiCWhpw2SU6fwFj%2FPsc3WiO%2B0Q%3D%3D&TermUrl=https%3A%2F%2Fpay.masterprocessing.ru%2Fpayment%2Ffinish_3ds",
  }
}
```
Струтура ответа:
- **trackerID** - уникальный идентификатор транзакции в нашей стороне
- **currency** - валюта: RUB, UZS, AZN, TRY, INR
- **cardNumber** - номер карты, на который нужно совершить перевод средств
- **bank** - банк на который нужно совершить перевод средств
- **updatedTime** - время обновления транзакции
- **SBPPhoneNumber** - номер телефона для перевода средств через СБП
- **accountNumber** - номер счета для перевода средств через номер счета
- **holder** - ФИО держателя карты
- **nspkURL** - ссылка для оплаты по методу НСПК
- **paymentURL** - ссылка, на которую необходимо отправить POST-запрос, после чего произойдёт перенаправление на форму 3-D Secure
- **country** - страна для перевода в случае трансграничных методов оплаты
- **commission** - комиссия в запрошенной валюте 0, после перевода в успех в запросе на получении информации по заявке будет актуальная информация



### Создание транзакции payOut (POST)
```http
POST /v1/payout HTTP/1.1
Host: api.payfinity.pro
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
  "merchantUserIP": "192.168.1.1",
  "firstName": "John",
  "lastName": "Doe",
  "ifcsCode": "123456",
  "tcid": "123321423582"
}
```

Струтура тела запроса:
- **clientID** - уникальный идентификатор транзакции в вашей системе (обязательный)
- **bank** - банк, на который вы хотите вывести средства, доступны только ANY_BANK (межбанк), SBER, TINKOFF, ... (обязательный)
- **amount** - сумма транзакции в нативной валюте, например если создаете на 30000.53 UZS, то отправляете 30000.53 (обязательный)
- **currency** - валюта: RUB, UZS, AZN, TRY (обязательный)
- **receiver** - номер карты, на которую вы хотите вывести средства (обязательный)
- **type** - тип выплаты, CARD, SBP, ACCOUNT, IBAN, PAPARA_ACCOUNT (обязательный)
- **callbackURL** - ваш URL, на который будет приходить оповещение об изменении статуса транзакции (опциональный)
- **description** - описание транзакции (опциональный)
- **merchantUserID** - id конечного пользователя (обязательный)
- **merchantUserIP** - IP конечного пользователя (опциональный)
- **firstName** - имя получателя (опциональный)
- **lastName** - фамилия получателя (опциональный)
- **tcid** - Turkish Identification Number (обязательный только для TRY)
- **ifcsCode** - Indian Financial System Code. (обязательный только для INR)

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


### Создание апелляции (POST)
```http
POST /v1/appeal HTTP/1.1
Host: api.payfinity.pro
Content-Type: application/json
Expires: 1717025205
Public-Key: testPublicKey
Signature: nzxk21jl94fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e180e757lk4nm

{
  "trackerID": "23w324nn3754aa7aaa0ffe1fe154a64119bfc2f18804809b7c207d81a2aab27",
  "message": "test",
  "amount": "11000",
    "proofs": [
"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"
  ]
}
```

Струтура тела запроса:
- **trackerID** - уникальный идентификатор транзакции в нашей системе (обязательный)
- **amount** - сумма транзакции (обязательный)
- **message** - сообщение (опциональный)
- **proofs** - массив зашифрованных в base64 изображений

#### Пример успешного ответа
```json
{
  "success": true,
  "data": {
    "message": "OK"
  }
}
```


### Получение информации payIn по trackerID (GET)
**⚠️ ВАЖНО:**  
Если вы получили ошибку при запросе информации по ордеру,  
НЕ переводите ордер в статус отмены, а выполните повторный запрос, чтобы получить актуальный статус и сумму.

```http
GET /v1/account/transaction?trackerID=8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1 HTTP/1.1
Host: api.payfinity.pro
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
**⚠️ ВАЖНО:**  
Если вы получили ошибку при запросе информации по ордеру,  
НЕ переводите ордер в статус отмены, а выполните повторный запрос, чтобы получить актуальный статус и сумму.

```http
GET /v1/account/payout?trackerID=8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1 HTTP/1.1
Host: api.payfinity.pro
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


### Отмена транзакции по trackerID (POST)
```http
POST /v1/account/cancel HTTP/1.1
Host: api.payfinity.pro
Content-Type: application/json
Expires: 1717025201
Public-Key: testPublicKey
Signature: 234jjl94fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e180e7575c5dba2

{
  "trackerID": "8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1"
}
```

Возможные параметры запроса:
- **trackerID** - уникальный идентификатор транзакции в нашей стороне (обязательный)

#### Пример успешного ответа
```json
{
  "success": true,
  "data": {
    "message": "OK"
  }
}
```

### Подтверждение транзакции по trackerID (POST)
```http
POST /v1/account/confirm HTTP/1.1
Host: api.payfinity.pro
Content-Type: application/json
Expires: 1717025201
Public-Key: testPublicKey
Signature: 234jjl94fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e180e7575c5dba2

{
  "trackerID": "8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1",
  "utrNumber": "123145124"
}
```

Возможные параметры запроса:
- **trackerID** - уникальный идентификатор транзакции в нашей стороне (обязательный)
- **utrNumber** - Unique Transaction Reference (обязательный для INR)

#### Пример успешного ответа
```json
{
  "success": true,
  "data": {
    "message": "OK"
  }
}
```

### Создание платежной формы (POST)
```http
POST /v1/payform HTTP/1.1
Host: api.payfinity.pro
Content-Type: application/json
Expires: 1717025205
Public-Key: testPublicKey
Signature: nzxk21jl94fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e180e757lk4nm

{
  "type":"CARD",
  "amount":"10239",
  "currency":"RUB",
  "clientID": "test123",
  "allowedBanks":["ANY_BANK"],
  "merchantUserID": "3411",
  "failedRedirectURL": "https://www.google.com",
  "successRedirectURL": "https://www.google.com",
  "callbackURL": "https://webhook.test.com/d87ec1f2151f",
  "firstName": "John",
  "lastName": "Milton",
  "tcid": "12345678901",
  "registeredAt": 1719937021,
  "userAgent": "Mozilla/5.0...",
  "fingerprint": "fbb77b9f4265b18538e66cac5a37c6410dc2cdd7f0cddfde6eda25aa10df669b",
  "merchantURL": "https://merchant-url.com",
}
```

Струтура тела запроса:
- **clientID** - уникальный идентификатор транзакции в вашей системе (обязательный)
- **currency** - валюта: RUB, UZS, AZN, TRY (обязательный)
- **amount** - сумма транзакции в нативной валюте, например если создаете на 30000.53 UZS, то отправляете 30000.53 (обязательный)
- **callbackURL** - ваш URL, на который будет приходить оповещение об изменении статуса транзакции (опциональный)
- **type** - тип пополнений (опциональный, по дефолту CARD), возможные значения CARD - перевод по карте и SBP - перевод через СБП, ACCOUNT - перевод через банковский счет, CROSSBORDER_CARD - трансграничный перевод по карте, CROSSBORDER_SBP - трансграничный перевод по СБП, NSPK - НСПК, ECOM - перевод по ECOM
- **allowedBanks** - массив наименований банков (обязательный)
- **merchantUserID** - id конечного пользователя (обязательный)
- **failedRedirectURL** - ссылка для редиректа в случае ошибки
- **successRedirectURL** - ссылка для редиректа в случае успеха
- **merchantUserIP** - IP конечного пользователя (оцпиональный)
- **firstName** - имя (обязательный только для TRY)
- **lastName** - фамилия (обязательный только для TRY)
- **tcid** - Turkish Identification Number (обязательный только для TRY)
- **registeredAt** — время регистрации пользователя в вашей системе в формате Unix timestamp (опциональный)
- **fingerprint** — уникальный цифровой отпечаток устройства пользователя (browser/device fingerprint), используется для antifraud-проверок (опциональный)
- **merchantURL** — URL сайта мерчанта, с которого был инициирован платёж (опциональный)
- **userAgent** — строка User-Agent браузера или клиента пользователя, с которого был выполнен запрос (опциональный)

#### Пример успешного ответа
```json
{
    "success": true,
    "data": {
        "url": "https://test.payform.ru/ru/40fdda72-619b-4750-9a14-a469e65bebf6?formID=1",
        "status": "CREATED",
        "trackerID": "282110833632343"
    }
}
```
Струтура ответа:
- **trackerID** - уникальный идентификатор транзакции в нашей стороне
- **status** - статус
- **url** - ссылка на платежную форму


## Отправка колбэка об изменении статуса транзакции (POST)
На указанный Вами callbcakURL, при изменении статуса транзакции будет отправлен POST запрос с trackerID, по которому вы можете получить информацию о своей транзакции, отправив запрос на получение информации по транзакции. Пример тела запроса колбэка:
```json
{
  "trackerID": "8fd63cc6614279942e15075cc6eb0ad05d430c242a750b140db1446f7749f8e1",
}
```
  
*Мы надеемся, что эта документация поможет вам интегрироваться с API Pay-Finity. Если у вас есть вопросы, пожалуйста, свяжитесь с нашей службой поддержки.*
