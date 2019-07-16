# OAuth 2.0

OAuth 2.0 是现在关于授权的开发网络标准.

## 应对的需求

借用阮一峰上的例子: 有一个"云冲印"的网站, 可以将用户储存在Google里的照片, 冲印出来. 用户为了使用该服务, 必须让"云冲印"读取自己储存在Google上的照片. 这时候用户需要给"云冲印"授权, 这样子"云冲印"才可以拿到Goole里面的照片. 我们应该怎么给"云冲印"授权呢? 传统方法是 我们将Google的用户密码告诉"云冲印", 这样后者就可以读取用户照片了. 但是这种做法有很大的问题:

+ "云冲印"为了后续的服务, 会保存用户的密码, 这样很不安全.
+ Google不得不部署密码登录, 而我们知道, 单纯的密码登录并不安全.
+ "云冲印"拥有了获取用户储存在Google所有资料的权力, 用户没法限制"云冲印"获得授权的范围和有效期.
+ 用户只有修改密码, 才能收回赋予"云冲印"的权力. 但是这样做, 会使得其他所有获得用户授权的第三方应用程序全部失效.
+ 只要有一个第三方应用程序被破解, 就会导致用户密码泄漏, 以及所有被密码保护的数据泄漏.

OAuth就是为了解决授权的问题而产生的.

## OAuth2角色

OAuth2一般分为5个角色:

+ Client: 客户端, 又称第三方应用程序, 即请求获取用户存放在资源服务器上受保护的资源, 如上面例子的"云冲印", 需要请求存放在Goole里面的图片信息.
+ User Agent: 用户代理, 用户参与互联网的工具, 一般指浏览器.
+ Authorization server: 认证服务器, 即服务提供商专门提供用来进行验证的服务器, 用于依据资源所有者的许可证对第三方应用下发令牌.
+ Resource server: 资源服务器, 即服务提供商存放用户生成的资源的服务器. 能够接收和响应持有令牌的资源访问请求. 它与认证服务器, 可以是同一台服务器, 也可以是不同的服务器.

![show](https://image.cjyong.com/bg2014051203.png)

(A) 用户打开客户端("云冲印"), 客户端要求用户给予授权.

(B) 用户给予客户端授权.

(C) 客户端使用上一步获得的授权, 向认证服务器申请令牌.

(D) 认证服务器对客户端进行认证后, 确认无误, 同意发放令牌.

(E) 客户端使用令牌向资源服务器申请资源.

(F) 资源服务器确认令牌无误, 同意向客户端开发资源.

## 客户端的授权模式

上面例子中的客户端如何获得用户给予的授权(A,B)是非常关键的, OAuth2.0定义了四种方式用于用户给客户端授权:

### Authorization Code Grant: 授权码模式

这是OAuth2.0中最标准, 应用最广泛的授权模式, 非常适合具备服务端的应用使用.

![show](https://image.cjyong.com/oauth-v2-authorization-code.png)

1. 客户端携带client_id, scope, redirect_uri, state等信息引导用户请求授权服务器的授权端点下发code

2. 授权服务器验证客户端身份，验证通过则询问用户是否同意授权（此时会跳转到用户能够直观看到的授权页面，等待用户点击确认授权）

3. 假设用户同意授权，此时授权服务器会将code和state（如果客户端传递了该参数）拼接在redirect_uri后面，以302形式下发code

4. 客户端携带code, redirect_uri, 以及client_secret请求授权服务器的令牌端点下发access_token(这一步实际上中间经过了客户端的服务器, 除了code, 其它参数都是在应用服务器端添加.

5. 授权服务器验证客户端身份，同时验证code，以及redirect_uri是否与请求code时相同，验证通过后下发access_token，并选择性下发refresh_token.

#### 获取授权码

授权码是授权流程的一个中间临时凭证, 是对用户确认授权这一操作的一个暂时性的证书, 其生命周期一般较短, 协议建议最大不要超过10分钟, 在这一有效时间周期内, 客户端可以凭借该暂时性证书去授权服务器换取访问令牌.
请求参数说明:

| 名称            | 是否必须 | 描述                                          |
|:--------------|:-----|:--------------------------------------------|
| response_type | 必须   | 该模式中 response_type = code                   |
| client_id     | 必须   | 客户端ID, 用户标识一个客户端, 等同于appID, 在注册应用时生成的       |
| redirect_uri  | 可选   | 授权回调地址                                      |
| scope         | 可选   | 权限范围, 如果为空, 默认所有权限                          |
| state         | 推荐   | 用于维持请求和回调过程的状态, 防止CSRF攻击, 服务器不进行处理, 响应时原样返回 |

如:

> HTTP/1.1 302 Found
> Location: `https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz`

错误响应参数说明:

| 名称                | 是否必须 | 描述信息          |
|:------------------|:-----|:--------------|
| error             | 必须   | 错误代码          |
| error_description | 可选   | 具备可读性的错误描述信息  |
| error_uri         | 可选   | 错误信息描述地址页面uri |
| state             | 可选   | 如果客户端传递, 原样返回 |

如:

> HTTP/1.1 302 Found
> Location: `https://client.example.com/cb?error=access_denied&state=xyz`

#### 下发令牌

授权服务器的授权端点在以302形式下发code之后，用户User-Agent，比如浏览器，将携带对应的code回调请求用户指定的redirect_url，这个地址应该能够保证请求打到应用服务器的对应接口，该接口可以由此拿到code，并附加相应参数请求授权服务器的令牌端点，授权端点验证code和相关参数，验证通过则下发access_token.

| 名称           | 是否必须 | 描述信息                                  |
|:-------------|:-----|:--------------------------------------|
| garnt_type   | 必须   | 该模式中: garnt_type = authorization_code |
| code         | 必须   | 上一步获取的授权码                             |
| redirect_uri | 必须   | 如果上一步设置, 必须相同                         |
| client_id    | 必须   | 客户端ID, 用户标识一个客户端, 等同于appID, 注册应用时生成   |

如果在注册应用时有下发客户端凭证信息（client_secret），那么客户端必须携带该参数以让授权服务器验证客户端的有效性。针对客户端凭证需要多说的一点就是，不能将其传递到客户端，客户端无法保证凭证的安全，凭证应该始终留在应用的服务器端，当下发code回调请求到应用服务器时，在服务器端携带上凭证再次请求下发令牌。

如:

> OST /token HTTP/1.1
> Host: server.example.com
> Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
> Content-Type: application/x-www-form-urlencoded
> grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb

授权服务器需要验证客户端的有效性，以及是否与之前请求授权码的客户端是同一个（请求授权时的信息可以记录在code，或以code为key建立缓存），授权服务器还要保证code处于生命周期内（推荐10分钟内有效），且只能被使用一次。授权服务器验证通过之后，生成access_token，并选择性下发refresh_token，OAuth2.0协议明确了token的下发策略，对于生成策略没有做太多说明，我们将在本系列的下一篇详细介绍两种token类型，即BEARER类型和MAC类型。

成功响应参数说明：

|      名称       | 是否必须 | 描述信息                                                      |
|---------------|:-----|:----------------------------------------------------------|
| access_token  | 必须   | 访问令牌                                                      |
| token_type    | 必须   | 访问令牌的类型, 如bearer, mac等等                                   |
| expires_in    | 推荐   | 访问令牌的生命周期, 以秒为单位, 如果未指定时间, 使用默认值                          |
| refresh_token | 可选   | 刷新令牌, 选择性下发                                               |
| scope         | 可选   | 权限范围, 如果最终下发的访问令牌队形的权限范围与实际应用指定的不一致, 则必须在下发访问令牌时使用该参数指定说明 |

如:

> HTTP/1.1 200 OK
> Content-Type: application/json;charset=UTF-8
> Cache-Control: no-store
> Pragma: no-cache
> {
> "access_token":"2YotnFZFEjr1zCsicMWpAA",
> "token_type":"example",
> "expires_in":3600,
> "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
> "example_parameter":"example_value"
> }

错误响应参数说明:

|        名称         | 是否必须 | 描述信息       |
|-------------------|:-----|:-----------|
| error             | 必须   | 错误代码       |
| error_description | 可选   | 描述信息       |
| error_uri         | 可选   | 错误描述信息页面地址 |

如:

> HTTP/1.1 400 Bad Request
> Content-Type: application/json;charset=UTF-8
> Cache-Control: no-store
> Pragma: no-cache
> {
> "error":"invalid_request"
> }

### 简化模式（Implicit Grant）

对于一些纯客户端应用，往往无法妥善的保管客户端的凭证，但是因为没有服务器端，所以无法向授权服务器传递客户端凭证，并且纯客户端应用在请求交互上要弱于有服务器的应用，这时候减少交互可以让应用的稳定性和用户体验更好，简化模式是对这一应用场景的优化。
简化模式跳过了`获取授权码`的步骤, 在安全性上要弱于授权码模式, 并且因为无法对当前客户端的真实性进行验证, 同时对于下发的access_token存在被同设备上其它应用窃取的风险, 为了降低这类风险, 隐式授权模式强制要求不能下发refresh_token.

![show](https://image.cjyong.com/oauth-v2-implicit-grant.png)

1. 客户端携带client_id, scope, redirect_uri, state等信息引导用户请求授权服务器下发access_token

2. 授权服务器验证客户端身份，验证通过则询问用户是否同意授权（此时会跳转到用户能够直观看到的授权页面，等待用户点击确认授权）

3. 假设用户同意授权，此时授权服务器会将access_token和state（如果客户端传递了该参数）等信息以URI Fragment形式拼接在redirect_uri后面，并以302形式下发

4. 客户端利用脚本解析获取access_token

#### 请求获取访问令牌

简化模式一步即可拿到访问令牌, 请求参数说明:

|      名称       | 是否必须 | 描述                                          |
|---------------|:-----|:--------------------------------------------|
| response_type | 必须   | 该模式中 response_type = token                  |
| client_id     | 必须   | 客户端ID, 用户标识一个客户端, 等同于appID, 在注册应用时生成的       |
| redirect_uri  | 可选   | 授权回调地址                                      |
| scope         | 可选   | 权限范围, 如果为空, 默认所有权限                          |
| state         | 推荐   | 用于维持请求和回调过程的状态, 防止CSRF攻击, 服务器不进行处理, 响应时原样返回 |

如:

> GET /authorize?response_type=token&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
> Host: server.example.com

成功响应参数说明:

|      名称       | 是否必须 | 描述信息                                                      |
|---------------|:-----|:----------------------------------------------------------|
| access_token  | 必须   | 访问令牌                                                      |
| token_type    | 必须   | 访问令牌的类型, 如bearer, mac等等                                   |
| expires_in    | 推荐   | 访问令牌的生命周期, 以秒为单位, 如果未指定时间, 使用默认值                          |
| refresh_token | 可选   | 刷新令牌, 选择性下发                                               |
| scope         | 可选   | 权限范围, 如果最终下发的访问令牌队形的权限范围与实际应用指定的不一致, 则必须在下发访问令牌时使用该参数指定说明 |
| state         | 可选   | 如果客户端传递了该参数, 原样返回                                         |

如:

> HTTP/1.1 302 Found
> Location: `http://example.com/cb#access_token=2YotnFZFEjr1zCsicMWpAA&state=xyz&token_type=example&expires_in=3600`

错误响应参数说明:

|        名称         | 是否必须 | 描述信息              |
|-------------------|:-----|:------------------|
| error             | 必须   | 错误代码              |
| error_description | 可选   | 描述信息              |
| error_uri         | 可选   | 错误描述信息页面地址        |
| state             | 可选   | 如果客户端传递了该参数, 原样返回 |

如:

> HTTP/1.1 302 Found
> Location: `https://client.example.com/cb#error=access_denied&state=xyz`

### 密码模式(Resource Owner Password Credentials Grant)

资源所有者密码凭证授权模式建立在资源所有者充分信任客户端的前提下，因为该模式客户端可以拿到用的登录凭证，从而在用户无感知的情况下完成整个授权流程，毕竟都有用户的登录凭证了，再弹窗让用户确认授权也是多此一举。
这里可能有一个比较疑惑的地方是既然已经拿到了用户的登录凭证，为什么还需要绕一大圈子走OAuth授权，拿到令牌再去请求用户的受保护资源呢？实际中事情可能并不会这么简单，拿到用户登录凭证的不一定是用户本身，而且这里协议指的用户登录凭证是用户的用户名和密码，实际中还可以是走SSO登录下发的token，token在持有权限上要小于等于用户的用户名和密码，这是从客户端角度出发，对于资源服务器来说，有些敏感数据需要在用户级别做权限控制，对于服务级别的控制粒度太粗，所以这些服务往往需要服务携带access_token来请求某一个用户的敏感数据。

举个例子来说，比如有一个服务是获取某个用户的通讯录，这是一个十分敏感的数据，且一般只能授予内部应用，如果是在服务级别进行控制，那么只要拿到服务权限，该应用可以请求获取任何一个用户的通讯录数据，这是一件十分危险的事情。然而如果基于access_token来做鉴权，那么就可以将粒度控制在用户级别，前面讲的两种授权方式在这里应用时都有一个共同的缺点，需要弹出授权页让用户确认授权，要知道这样的场景往往是发生在内部应用里面，内部应用是可以持有用户登录态的，这里的确认授权对于一个用户体验好的APP来说就应该发生在用户登录时，通过用户协议等方式直接告诉用户，从而让用户在一次登录过程中可以让应用拿到用户的登录态和访问令牌。资源所有者密码凭证授权模式的交互时序如下：

![show](https://image.cjyong.com/oauth-v2-resource-owner-password-credentials.png)

1. 用于授予客户端登录凭证（比如用户名和密码信息）

2. 客户端携带用户的登录凭证和scope等信息请授权服务器的令牌端点下发refresh_token

3. 授权服务器验证用户的登录凭证和客户端信息的有效性，验证通过则下发access_token，并选择性下发refresh_token.

用户授予登录凭证:
用于登录凭证如何传递给客户端这一块协议未做说明，实际应用中该类授权一般应用在内部应用，这类应用的特点就是为用户提供登录功能，当用户登录之后，这类应用也就持有了用户的登录态，可以是用户登录的session标识，也可以是走SSO下发的token信息。

请求获取访问令牌:
请求参数说明:

|     名称     | 是否必须 | 描述信息                                   |
|------------|:-----|:---------------------------------------|
| grant_type | 必须   | 对于本模式: grant_type = password           |
| username   | 必须   | 用户名                                    |
| password   | 必须   | 用户密码                                   |
| scope      | 可选   | 权限返回, 如果最终下发访问令牌的权限不一致, 需要在下发令牌时用该参数指定 |

如果在注册应用时有下发客户端凭证信息（client_secret），那么客户端必须携带该参数以让授权服务器验证客户端的有效性。

如:

> POST /token HTTP/1.1
> Host: server.example.com
> Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
> Content-Type: application/x-www-form-urlencoded
> grant_type=password&username=johndoe&password=A3ddj3w

成功响应参数说明:

|      名称       | 是否必须 | 描述信息                                                      |
|---------------|:-----|:----------------------------------------------------------|
| access_token  | 必须   | 访问令牌                                                      |
| token_type    | 必须   | 访问令牌的类型, 如bearer, mac等等                                   |
| expires_in    | 推荐   | 访问令牌的生命周期, 以秒为单位, 如果未指定时间, 使用默认值                          |
| refresh_token | 可选   | 刷新令牌, 选择性下发                                               |
| scope         | 可选   | 权限范围, 如果最终下发的访问令牌队形的权限范围与实际应用指定的不一致, 则必须在下发访问令牌时使用该参数指定说明 |

如:

> HTTP/1.1 200 OK
> Content-Type: application/json;charset=UTF-8
> Cache-Control: no-store
> Pragma: no-cache
> {
> "access_token":"2YotnFZFEjr1zCsicMWpAA",
> "token_type":"example",
> "expires_in":3600,
> "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
> "example_parameter":"example_value"
> }

错误响应参数说明:

|        名称         | 是否必须 | 描述信息       |
|-------------------|:-----|:-----------|
| error             | 必须   | 错误代码       |
| error_description | 可选   | 描述信息       |
| error_uri         | 可选   | 错误描述信息页面地址 |

如:

> HTTP/1.1 400 Bad Request
> Content-Type: application/json;charset=UTF-8
> Cache-Control: no-store
> Pragma: no-cache
> {
> "error":"invalid_request"
> }

### 客户端模式(Client Credentials Grant)

客户端凭证授权模式基于客户端持有的证书去请求用户的受保护资源，如果把这里的受保护资源定义得更加宽泛一点，比如说是对一个内网接口权限的调用，那么这类授权方式可以被改造为内网权限验证服务。客户端凭证授权模式的交互时序如下：

![shows](https://image.cjyong.com/oauth-v2-client-credentials.png)

1. 客户端携带客户端凭证和scope等信息请求授权服务器的令牌端点

2. 授权服务器验证客户端凭证，验证通过下发access_token

|     名称     | 是否必须 | 描述信息                                   |
|------------|:-----|:---------------------------------------|
| grant_type | 必须   | 对于本模式: grant_type = client_credentials |
| scope      | 可选   | 权限返回, 如果最终下发访问令牌的权限不一致, 需要在下发令牌时用该参数指定 |

如:

> POST /token HTTP/1.1
> Host: server.example.com
> Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
> Content-Type: application/x-www-form-urlencoded
> grant_type=client_credentials

成功响应参数说明:

|      名称       | 是否必须 | 描述信息                                                      |
|---------------|:-----|:----------------------------------------------------------|
| access_token  | 必须   | 访问令牌                                                      |
| token_type    | 必须   | 访问令牌的类型, 如bearer, mac等等                                   |
| expires_in    | 推荐   | 访问令牌的生命周期, 以秒为单位, 如果未指定时间, 使用默认值                          |
| refresh_token | 可选   | 刷新令牌, 选择性下发                                               |
| scope         | 可选   | 权限范围, 如果最终下发的访问令牌队形的权限范围与实际应用指定的不一致, 则必须在下发访问令牌时使用该参数指定说明 |

最后访问令牌以JSON格式响应，并要求指定响应首部Cache-Control: no-store和Pragma: no-cache。

成功响应参数示例：

> HTTP/1.1 200 OK
> Content-Type: application/json;charset=UTF-8
> Cache-Control: no-store
> Pragma: no-cache
> {
> "access_token":"2YotnFZFEjr1zCsicMWpAA",
> "token_type":"example",
> "expires_in":3600,
> "example_parameter":"example_value"
> }

错误响应参数说明:

|        名称         | 是否必须 | 描述信息       |
|-------------------|:-----|:-----------|
| error             | 必须   | 错误代码       |
| error_description | 可选   | 描述信息       |
| error_uri         | 可选   | 错误描述信息页面地址 |

如:

> HTTP/1.1 400 Bad Request
> Content-Type: application/json;charset=UTF-8
> Cache-Control: no-store
> Pragma: no-cache
> {
> "error":"invalid_request"
> }

## 参考资料

[RFC6749](http://www.rfcreader.com/#rfc6749)

[阮一峰OAuth2详解](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)

[指间生活博客](http://www.zhenchao.org/2017/03/04/oauth-v2-principle/)
