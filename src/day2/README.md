# Day2: Anatomy of an HTTP message

HTTP 簡單來說就是一個文本(字串)傳輸的協議

就像小學的時候，學校會教書信格式

```
小明：
    您好，近來身體可好
    內容......
    祝
身體健康、工作順利
　　　　　　　　　　　　　　　　　　　　學生：王大明
　　　　　　　　　　　　　　　　　　　　　　１月１日
```

HTTP(Hyper Text Transfe Protocol)其實也是各種大神們訂的一個格式

分為 Request 跟 Response

HTTP Request:

https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages#http_requests

```
POST /users HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 50

name=FirstName%20LastName&email=bsmth%40example.com
```

HTTP Response:

https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages#http_responses

```
HTTP/1.1 201 Created
Content-Type: application/json
Location: http://example.com/users/123

{
  "message": "New user created",
  "user": {
    "id": 123,
    "firstName": "Example",
    "lastName": "Person",
    "email": "bsmth@example.com"
  }
}
```

可以看到 Request 跟 Response 都分成三個區塊

```
start-line
headers

body
```

區塊之間是透過 `\r\n`(也就是所謂的 CRLF) 來切開的

有些聰明的小夥伴可能就會想到

那如果我在 header value 插入 `\r\n` 會發生什麼事情?

還真的有人發現這種漏洞，漏洞名為 CRLF Injection

簡單來說就是 server 端沒有驗證使用者傳入的參數，就直接塞進 header value

```js
const queryStringRedirect = "https://google.com\r\nSet-Cookie: xxx";
res.setHeader("Location", queryStringRedirect);
```

透過這個 CRLF Injection，就可以設定很多惡意的 header

但如果你實際上用 nodejs 跑上面的程式碼，會發現噴錯

```
TypeError: Invalid character in header content ["Location"]
    ......
  code: 'ERR_INVALID_CHAR'
```

因為 nodejs 有實作過濾，避免 CRLF Injection 的情況發生

https://github.com/nodejs/node/blob/main/lib/_http_outgoing.js#L697
```js
OutgoingMessage.prototype.setHeader = function setHeader(name, value) {
  if (this._header) {
    throw new ERR_HTTP_HEADERS_SENT('set');
  }
  validateHeaderName(name);
  validateHeaderValue(name, value);

  let headers = this[kOutHeaders];
  if (headers === null)
    this[kOutHeaders] = headers = { __proto__: null };

  headers[name.toLowerCase()] = [name, value];
  return this;
};
```

其中過濾的邏輯就是這個 `validateHeaderValue`

https://github.com/nodejs/node/blob/main/lib/_http_outgoing.js#L673

```js
const validateHeaderValue = hideStackFrames((name, value) => {
  if (value === undefined) {
    throw new ERR_HTTP_INVALID_HEADER_VALUE.HideStackFramesError(value, name);
  }
  if (checkInvalidHeaderChar(value)) {
    debug('Header "%s" contains invalid characters', name);
    throw new ERR_INVALID_CHAR.HideStackFramesError('header content', name);
  }
});
```

我們再繼續往下追 `checkInvalidHeaderChar` 的實作:

https://github.com/nodejs/node/blob/main/lib/_http_common.js#L214

```js
const headerCharRegex = /[^\t\x20-\x7e\x80-\xff]/;
/**
 * True if val contains an invalid field-vchar
 *  field-value    = *( field-content / obs-fold )
 *  field-content  = field-vchar [ 1*( SP / HTAB ) field-vchar ]
 *  field-vchar    = VCHAR / obs-text
 */
function checkInvalidHeaderChar(val) {
  return headerCharRegex.test(val);
}
```

這就是為什麼了解 HTTP 的規範，還有其他的底層邏輯很重要

我相信能發些這些資安漏洞的大神們

一定對於這些規範也都很了解，才能想得出有這種資安漏洞可以鑽

如果大家對 CRLF Injection 有興趣，可參考以下 CVE 資料庫紀載的資安漏洞

- https://nvd.nist.gov/vuln/detail/CVE-2023-0040
- https://nvd.nist.gov/vuln/detail/CVE-2024-53693

今天的內容到這裡，開頭是介紹 Anatomy of an HTTP message

但沒想到卻延伸到了 CRLF Injection XDD

## 參考資料
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages#anatomy_of_an_http_message
- https://ithelp.ithome.com.tw/m/articles/10242682
- https://ithelp.ithome.com.tw/m/articles/10353840