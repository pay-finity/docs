# API Документация

## Введение

Добро пожаловать в документацию по API **Pay-Finity**. Наш API позволяет вам программно взаимодействовать с нашими сервисами, что обеспечивает бесшовную интеграцию с вашими приложениями.

## Аутентификация и Подпись

Для обеспечения безопасности нашего API все запросы должны быть подписаны с использованием подписи (Signature), сгенерированной с помощью приватного ключа (PrivateKey), который мы предоставляем нашим клиентам. Подпись используется для проверки целостности и подлинности запросов.

### Генерация Подписи

Подпись генерируется с использованием алгоритма HMAC-SHA512. Ниже приведена пример функции генерирования подписи (Signature) на Go:

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

#### Пример кода на Go для формирования сообщения:

```go
func generateSign(r *resty.Request, body []byte, exp, secret string) (string, error) {
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
- **exp**: время жизни запроса в формате UNIX, которое вы вставляетя в заголовок"Expires" (о заголовках можете прочитать ниже)
- **secret**: приватный ключ (PrivateKey), переданный вам поддержкой Pay Finity
- **body**: тело запроса, в случае GET оно пустое (nil/null), обязательно чтобы JSON ключи шли в алфавитном порядке!!!
#### <span style="color:red"> *Обязательно JSON ключи в теле (body) запроса должны идти в алфавитном порядке!!!* </span>

### Примеры

Для GET запроса на `https://pay-finity.com/api/v1/account/transactions?limit=5&page=1`:

- **URL**: `/api/v1/account/transactions`
- **Параметры запроса**: `currency=RUB`
- **Сообщение - message (для подписи)**: `/api/v1/accountuser_id=123`

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

- **success**: Статус запроса.
- **data**: Данные, возвращенные API (только в случая успешного ответа от API).
- **error**: Данные, по возникшей ошибки (только в случая ошибочного ответа от API).

### Пример ответа

```json
{
  "success": true,
  "data": {
    "balance": [
      {
        "currency": "RUB",
        "available": "985.95",
        "frozen": "100.21"
      }
    ]
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

| Код ошибки | Сообщение                          | HTTP статус код |
|------------|------------------------------------|-----------------|
| 10000      | unauthorized                       | 401             |
| 20000      | wrong input                        | 400             |
| 20001      | can't bind body to request model   | 422             |



Мы надеемся, что эта документация поможет вам интегрироваться с API Pay-Finity. Если у вас есть вопросы, пожалуйста, свяжитесь с нашей службой поддержки.
