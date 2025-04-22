# EasyCrm.Hub

EasyCrm.Hub 是一个用于集成会员管理系统的 的 C# 类库（SDK），该类库协助第三方系统调用会员管理系统，实现渠道绑定、会员信息同步、订单信息同步等操作。

## 安装

使用 NuGet 包管理器：

```bash
Install-Package EasyCrm.Hub -Version 0.1.3-alpha.6
```

## 基本概念

### 一、EasyCrmConfig、EasyCrmClient

EasyCrm.Hub 将会员管理系统提供给第三方系统的接口封装成对应Service的方法。这样，当第三方系统需要对接会员管理系统时，只需要通过安装该sdk，按照一定的配置好EasyConfig、EasyCrmClient以及对应Service，
就访问到会员管理系统，无需自己重新再写一套对接代码，重新配置请求Header以及接口所需参数等。


- EasyConfig: 配置会员管理系统的Url、访问请求时需要用到的AppId、AppSecret。


- EasyClient：EasyClient会在初始化时，通过配置EasyConfig，从而实现给每个访问会员管理系统的请求配置请求，根据EasyConfig配置的AppId、AppSecret，配置好请求Header，如根据AppId+AppSecret+TimeStamp通过SHA256加密获取签名等，用于会员管理系统Api的身份验证。


- Service：根据不同的功能，区分出不同的Service，每个Service中的每个方法，会对应会员管理系统的一个接口。每个Service在初始化时，都需要传入EasyClient，才能实现在每个请求添加会员管理系统用于身份验证的Header。


对于如何使用EasyConfig、EasyClient、Service，请参考下面的示例代码。

### 示例代码

```csharp
using EasyCrm.Hub.HttpClient;
using EasyCrm.Hub.Service.Location;

// 配置EasyCrmConfig
var easyCrmConfig = new EasyCrmConfig("easycrm_base_url", "your_app_id", "your_secret");
        
// 配置EasyCrm客户端
var easyCrmClient = new EasyCrmClient(easyCrmConfig);

// 创建ChannelService
var channelService = new ChannelService(easyCrmClient);

// 调用ChannelService.BindChannelnAsync 绑定渠道
var bindResponse = await channelService.BindChannelAsync(webhookUrl, merchId, CancellationToken.None);

```


配置参数说明：

- **easycrm_base_url**: 会员管理系统的Url。
- **your_app_id**: 会员管理系统渠道的AppId，即会员管理系统对应渠道的AppId。
- **your_secret**: 会员管理系统渠道的AppSecret，即会员管理系统对应渠道的AppId，用于身份验证。

通过上面的代码，配置好EasyConfig、EasyClient，我们就可以轻松跟会员管理系统对接，实现渠道绑定、同步会员、同步等订单功能。那会员管理系统究竟是怎么进行身份验证的？

### 二、会员管理系统的身份验证


#### 1. 请求Header

会员管理系统的身份验证逻辑旨在确保数据传输的安全性和完整性。通过在请求的 Header 中传递必要的参数来实现身份验证。这些参数包括：

- **app-id**: 唯一标识客户端应用程序的 ID，即会员系统对应渠道的AppId。
- **signature**: 由 渠道的`AppId`、`AppSecret` 和 `Timestamp` 组合并通过SHA256算法加密生成的签名，用于验证请求的完整性。
- **timestamp**: 当前时间戳，用于防止重放攻击，10s内有效。


EasyCrm.Hub在SignatureValidator类中定义了生成时间戳、校验时间戳有效性、生成签名、验证签名等方法，方便第三方系统生成对应的Header。


#### 1.1. 请求头名称



为了统一Header名称，在EasyCrm.Hub的ApiConstants中，已定义了以下请求头名称，方便开发者通过ApiConstants类中获取到对应的header名，请求会员管理系统的接口或者作为后续会员管理系统回调event时身份验证的依据：

- `app-id`: 用于传递 `appid`,可从ApiConstants.AppId获取到header名


- `signature`: 用于传递 `signature`,可从ApiConstants.Signature获取到header名


- `timestamp`: 用于传递 `timestamp`,可从ApiConstants.TimeStamp获取到header名


#### 1.2. 获取timestamp

为了方便第三方获取当前时间戳，EasyCrm.Hub在SignatureValidator类中也定义了获取当前时间戳的方法（GetTimestamp），默认获取秒级时间戳。此外该方法也支持调用方通过传入对应的参数，获取秒级、毫秒级、微妙级、纳秒级的时间戳。


- `GetTimestamp` 的调用方法法如下：

    ```
      
    // 获取秒级的当前时间戳
    long timestamp1 = SignatureValidator.GetTimestamp();
      
    // 获取毫秒级的当前时间戳
    long timestamp2 = SignatureValidator.GetTimestamp(TimestampPrecision.Milliseconds);
      
    // 获取微秒级的当前时间戳
    long timestamp3 = SignatureValidator.GetTimestamp(TimestampPrecision.Microseconds);
      
    // 获取纳秒级的当前时间戳
    long timestamp4 = SignatureValidator.GetTimestamp(TimestampPrecision.Nanoseconds);
      
    ```

#### 1.3. 验证timestamp有效性

- 为了防止重放攻击，会员管理系统对于提供给第三方的接口，都会将请求Header传入的timestamp跟当前时间戳做对比，校验是否在10s内。如果在10s内，则认为当前请求的timestamp有效，否则会返回401的错误。

- EasyCrm.Hub的SignatureValidator类定义了验证时间戳有效性的方法（ValidateTimestamp），无论是第三方系统还是会员管理系统，都可使用方法验证时间戳是否在一定时间内的


- `ValidateTimestamp` 的调用方法法如下：

    ```
    // 验证时间戳是否在10s内有效
    bool result = SignatureValidator.ValidateTimestamp(currentTimestamp, 10);
      
    ```

- 参数说明：

    | 参数名          | 类型   | 是否必填 | 描述             |
    | --------------- | ------ | -------- | ---------------- |
    | `timestamp`     | `long` | 是       | 需要校验的时间戳 |
    | `validDuration` | `int`  | 否       | 秒级，默认是10s  |


#### 1.4. 生成签名

为了方便第三方请求会员管理系统时生成签名，以及会员管理系统身份验证时生成签名，EasyCrm.Hub在SignatureValidator类的GenerateSignature方法中，
定义了如何根据AppId，AppSecret、Timestamp通过SHA256算法生成加密后的signature。


- `signature` 的生成方法如下：

    ```
      
    string signature = SignatureValidator.GenerateSignature(appId, appSecret, timestamp)
      
    ```

- 其中，`SignatureValidator.GenerateSignature` 是sdk中的一个加密函数，使用安全算法（如 HMAC-SHA256）进行加密。

#### 1.5. 验证签名有效性

EasyCrm.Hub也在SignatureValidator类定义了校验signature的方法（VerifySignature），通过传入signature、appId、appSecret、timestamp，验证signature是否有效

- 验证`signature` 的有效性方法如下：

    ```
      
    bool isValid = SignatureValidator.VerifySignature(signature, appId, appSecret, timestamp)
      
    ```


#### 2. 会员管理系统身份验证流程

(1). **参数准备**:

  - 第三方在发送请求时，需要将 `AppId`、`Timestamp` 和 `AppSecret`生成的 `app-id`、`signature` 和 `timestamp` 添加到请求 Header 中。

    - `Header` 的生成方法如下：

        ```
        var appId = "App12313";
        var appSecret = "Secret12313";
        var currentTimeStamp = SignatureValidator.GetTimestamp();
        var signature = SignatureValidator.GenerateSignature(config.AppId, config.AppSecret, currentTimeStamp);
        
        httpRequestMessage.Headers.Add(ApiConstants.AppId, config.AppId);
        httpRequestMessage.Headers.Add(ApiConstants.Signature, signature);
        httpRequestMessage.Headers.Add(ApiConstants.RequestId, $"{Guid.NewGuid()}");
        httpRequestMessage.Headers.Add(ApiConstants.TimeStamp, currentTimeStamp.ToString());
        ```

(2) **请求发送**:

  - EasyCrmClient将上述参数放到Header中，并发送到服务器。

(3) **服务器验证**:

  - 服务器接收到请求后，会提取 Header 中的 `app-id`、`signature` 和 `timestamp`。
  - 服务器会根据接收到的 `app-id` 查询对应的 `appsecret`，并重新计算 `signature`。
  - 如果计算出的 `signature` 与接收到的值一致，并且当前时间与传入的 `timestamp` 相差**不超过 10 秒**，则认为该请求有效。

### 二、Webhook 回调

当会员管理系统做了一些处理，需要通过Webhook事件通知给第三方系统时。会员管理系统会向第三方系统发送 HTTP  POST请求，在Header添加相应的参数，并在body中传入相关的 Webhook事件，请求第三方的Webhook接口（该接口地址，即在绑定渠道时第三方传入的webhookUrl）。

#### 1. Webhook 回调事件

会员管理系统会向第三方系统绑定渠道时，通过 POST 请求方式向指定的回调 URL (即在绑定渠道时，第三方传入的webhookUrl) 发送数据。请求体将包含如下结构的数据：

```csharp
public class IEventData
{
    public EventType EventType { get; set; } // event类型
    public string Data { get; set; }         // event数据（序列化后的数据）
}
```

在这个结构中，事件的数据会被序列化后存储到属性 `Data` 中。各自系统需要根据事件类型 (`EventType`) 将序列化的数据反序列化为对应的事件数据 (`Data`) 来执行相应逻辑处理。


- EventType：列举会员管理系统的Webhook事件类型，在EasyCrm.Hub.Enums命名空间下


- Data：Webhook的Event数据序列化后的数据，其中每个Event数据的定义都会放在EasyCrm.Hub.Events命名空间下

#### 回调事件

| 事件名称                     | 描述                         | Data对应的数据类型                    |
| ---------------------------- | ---------------------------- | ------------------------------------- |
| EventType.UnBindChannelEvent | 用户解除绑定某个渠道时触发。 | EasyCrm.Hub.Events.UnBindChannelEvent |




#### 2. Webhook身份验证

像第三方系统调用会员管理系统接口传入的Header一样，会员管理系统发起Webhook回调时，同样会在Header中传入`app-id`、`signature`、`timestamp`。第三方系统就可以利用这些 Header 数据，通过 SDK 封装的身份验证通用方法，对会员管理系统发起的请求进行身份验证。

- **`app-id`**:  唯一标识客户端应用程序的 ID，即会员系统对应渠道的AppId。


- **`signature`**: 用由 渠道的`AppId`、`AppSecret` 和 `Timestamp` 组合并通过SHA256算法加密生成的签名，用于验证请求的完整性。


- **`timestamp`**: 请求发送时的时间戳，用于防止重放攻击，第三方系统可以自己设置时间戳的有效时间。


##### 示例代码

```csharp
// 从Header 中获取到 app-id
string appid = "App24234234";

// 第三方系统根据appId，查询对应的appSecret
string appsecret = "Secret24324";

// 获取当前时间戳
long currentTimeStamp = SignatureValidator.GetTimestamp();

// 根据 appid、appsecret 和 currentTimeStamp 生成签名
string signature = SignatureValidator.GenerateSignature(appid, appsecret, currentTimeStamp);

// 验证时间戳是否在 30 秒内有效
bool isTimestampValid = SignatureValidator.ValidateTimestamp(currentTimeStamp, 30);

// 验证签名
bool isSignatureValid = SignatureValidator.VerifySignature(signature, appid, appsecret, timestamp);

// 只有当时间戳以及签名有效，才认为验证通过
bool isValid = isTimestampValid && isSignatureValid;
```


## 会员管理系统API

### 一、配置 EasyCRM 客户端

在开始之前，需要配置 `EasyCrmClient` 实例。以下是所需的配置信息：

- **easycrm_base_url**: 会员管理系统的Url。
- **your_app_id**: 会员管理系统渠道的AppId，即会员管理系统对应渠道的AppId。
- **your_secret**: 会员管理系统渠道的AppSecret，即会员管理系统对应渠道的AppId，用于身份验证。

### 示例代码

```csharp
using EasyCrm.Hub.HttpClient;
using EasyCrm.Hub.Service.Location;

// 配置EasyCrmConfig
var easyCrmConfig = new EasyCrmConfig("easycrm_base_url", "your_app_id", "your_secret");
        
// 配置EasyCrm客户端
var easyCrmClient = new EasyCrmClient(easyCrmConfig);

// 创建ChannelService
var channelService = new ChannelService(easyCrmClient);

// 调用ChannelService.BindChannelnAsync 绑定渠道
var bindResponse = await channelService.BindChannelAsync(webhookUrl, merchId, CancellationToken.None);

```

### 二、异常处理

在EasyCrm Service底层，会对于如果请求响应状态码表示成功（2xx），则返回响应文本；如果因为参数校验、会员管理系统业务逻辑校验异常等返回了其他状态码时，则抛出异常HttpClientException，第三方系统则需要捕获到该异常，对于不同异常做相应处理：

- 常规异常捕获，当请求会员管理系统接口时，如果业务逻辑上校验不通过，则会返回非2xx的状态以及对应的异常类型，sdk这边就会将返回的结果，按照ExceptionResultDto组装对应的数据，并序列化到HttpClientException.ResponseBody中。

```csharp
  try
  {
      var response = await channelService.BindChannelAsync(webhookUrl, merchId, CancellationToken.None);
  }
  catch (HttpClientException exception)
  {
       var exceptionDto = JsonConvert.DeserializeObject<ExceptionResultDto>(exception.ResponseBody);
      
      // 第三方系统选择对于不同的异常，做特殊处理
      if (exceptionDto.Type == "UnBindWithoutBindingException")
      {
          
      }
  }
```


- 对于不同异常的处理，第三方需要通过使用try-catch块来捕获到异常，并通过ExceptionResultDto.Type来获取到具体的异常类型，做各自的逻辑。常见的异常类型如下：


| ExceptionType                     | 描述                                        |
| --------------------------------- | ------------------------------------------- |
| InvalidAppIdAndAppSecretException | 无效的AppId或AppSecret                      |
| InvalidWebhookUrlException        | 绑定渠道时，提供的Event回调Url不符合规范    |
| MissExternalMerchIdException      | 绑定渠道时，没有提供MerchId                 |
| RepetitiveBindChannelException    | 该渠道已被绑定了，不可以重复绑定            |
| UnBindWithoutBindingException     | 该渠道没有被绑定，无法解绑                  |
| CustomerNotFoundException         | 更新会员信息时，该会员不存在                |
| QuantityExceededException         | 同步会员信息/订单信息时，超过单次同步的条数 |
| ValidationException               | 请求参数无效，如缺少必填参数、参数值不对等  |


### 三、EasyCrm Services


EasyCrm.Hub将会员管理系统提供给第三方的接口，按照功能，分成不同Service的不同方法，组装成每个接口需要的参数，传递给会员管理系统。
第三方系统则可以通过直接调用这些方法，请求到会员管理系统中，实现渠道绑定、同步会员信息、同步订单信息等操作。


### 1、渠道（ChannelService）


在会员管理系统中，一个店铺下会有多个Location，每个Location中都会有不同类型的Channel。ChannelService提供了绑定渠道、解绑渠道以及查看渠道详情等相关方法。


### 1.1. 绑定渠道 (BindChannelAsync)

- 功能：第三方系统对接会员管理系统时，需要先绑定到对应的渠道，后面的会员、订单信息才能同步到会员系统中。


- 请求参数 :

    | 字段名              | 类型                | 必填 | 描述                        |
    | ------------------- | ------------------- | ---- | --------------------------- |
    | `webhookUrl`        | `string`            | 是   | 接收会员系统Event的回调地址 |
    | `merchId`           | `string`            | 是   | 第三方系统的商家Id          |
    | `cancellationToken` | `CancellationToken` | 是   | 用于取消操作的令牌          |

- 返回值 :

    - BindChannelResponse :包含绑定结果的响应对象

        | 字段名     | 类型          | 描述         |
        | ---------- | ------------- | ------------ |
        | `Merch`    | `MerchDto`    | 店铺信息     |
        | `Location` | `LocationDto` | Location信息 |
        | `Channel`  | `ChannelDto`  | Channel信息  |

    - MerchDto字段说明

        | 字段名         | 类型     | 描述                 |
        | -------------- | -------- | -------------------- |
        | `Id`           | `long`   | 会员管理系统的店铺Id |
        | `EnterpriseId` | `long`   | 店铺所属的 企业Id    |
        | `Name`         | `string` | 店铺名称             |
        | `Logo`         | `string` | 店铺Logo             |

    - LocationDto字段说明

        | 字段名       | 类型     | 描述         |
        | ------------ | -------- | ------------ |
        | `Id`         | `long`   | Location Id  |
        | `MerchId`    | `long`   | 店铺Id       |
        | `Name`       | `string` | Location名   |
        | `Address`    | `string` | Location地址 |
        | `Lat`        | `long`   | 纬度         |
        | `Lng`        | `string` | 维度         |
        | `TimezoneId` | `string` | 时区ID       |

    - ChannelDto字段说明
               

        | 字段名           | 类型          | 描述                 |
        | ---------------- | ------------- | -------------------- |
        | `Id`             | `long`        | 渠道Id               |
        | `LocationId`     | `long`        | 渠道所属的LocationId |
        | `Name`           | `string`      | 渠道名称             |
        | `Logo`           | `string`      | 渠道Logo             |
        | `IsActive`       | `bool`        | 渠道是否被绑定       |
        | `AppId`          | `string`      | 渠道的AppId          |
        | `AppSecret`      | `string`      | 渠道的AppSecret      |
        | `ChannelType`    | `ChannelType` | 渠道类型             |
        | `EasyCrmMerchId` | `string`      | 第三方系统的MerchId  |


- 示例 :

    ```csharp
    var easyCrmConfig = new EasyCrmConfig("easycrm_base_url", "your_app_id", "your_secret");
            
    // 配置EasyCrm客户端
    var easyCrmClient = new EasyCrmClient(easyCrmConfig);
    
    // 创建ChannelService
    var channelService = new ChannelService(easyCrmClient);
    
    // 调用ChannelService.BindChannelnAsync 绑定渠道
    var bindResponse = await channelService.BindChannelAsync(webhookUrl, merchId, CancellationToken.None);
    
    ```

### 2. 解绑渠道 (UnBindChannelAsync)


- 功能：当第三方系统不再需要绑定该渠道时，则需要调用该方法去解绑。解绑后，之前同步的订单和会员信息都依旧会保留在当时的渠道中。


- 请求参数 :

    | 字段名              | 类型                | 必填 | 描述               |
    | ------------------- | ------------------- | ---- | ------------------ |
    | `cancellationToken` | `CancellationToken` | 是   | 用于取消操作的令牌 |

- 返回值 :

    - UnBindChannelResponse : 包含解绑结果的响应对象

        | 字段名    | 类型         | 描述        |
        | --------- | ------------ | ----------- |
        | `Channel` | `ChannelDto` | Channel信息 |


- 示例 :

    ```csharp
    var easyCrmConfig = new EasyCrmConfig("easycrm_base_url", "your_app_id", "your_secret");
            
    // 配置EasyCrm客户端
    var easyCrmClient = new EasyCrmClient(easyCrmConfig);
    
    // 创建ChannelService
    var channelService = new ChannelService(easyCrmClient);
    
    // 调用ChannelService.UnBindChannelnAsync 解绑渠道
    var unBindResponse = await channelService.UnBindChannelAsync(CancellationToken.None);
    
    ```

### 3. 获取渠道详情 (GetChannelDetailAsync)


- 功能：根据渠道ID获取渠道的详细信息。


- 请求参数 :

    | 字段名              | 类型                | 必填 | 描述               |
    | ------------------- | ------------------- | ---- | ------------------ |
    | `channelId`         | `long`              | 是   | 渠道ID             |
    | `cancellationToken` | `CancellationToken` | 是   | 用于取消操作的令牌 |



- 返回值 :

    - GetChannelDetailResponse : 包含Channel的详情数据

        | 字段名     | 类型          | 描述         |
        | ---------- | ------------- | ------------ |
        | `Merch`    | `MerchDto`    | 店铺信息     |
        | `Location` | `LocationDto` | Location信息 |
        | `Channel`  | `ChannelDto`  | Channel信息  |


- 示例 :

    ```csharp
    var easyCrmConfig = new EasyCrmConfig("easycrm_base_url", "your_app_id", "your_secret");
              
      // 配置EasyCrm客户端
      var easyCrmClient = new EasyCrmClient(easyCrmConfig);
      
      // 创建ChannelService
      var channelService = new ChannelService(easyCrmClient);
      
      long channelId = 1;
    
      // 调用ChannelService.GetChannelDetailAsync 获取渠道详情
      var Response = await channelService.GetChannelDetailAsync(channelId, CancellationToken.None);
      
    ```

### 2、会员（CustomerService）


- 在会员管理系统中，在一个店铺下，以手机号区分不同的会员。


- 会员同步的方式有很多种方式，一种是在会员小后台通过表格的方式去同步会员数据，另一种则是通过sdk绑定到对应渠道后，同步会员数据。


- 在一个店铺下，不同location的channel下，同个手机号的会员是同一个会员


- 更新会员时，会员的昵称、语言，只有是第一次同步会员对应的渠道/会员管理系统后小后台通过表格去更新，才能去更新，其他渠道的都不会更新。

### 2.1. AddOrUpdateCustomerAsync（新增/更新会员）


- 功能 : 新增或更新会员信息


- 请求参数 :
          

    | 字段名                    | 类型                      | 必填 | 描述                 |
    | ------------------------- | ------------------------- | ---- | -------------------- |
    | `addOrUpdateCustomerBody` | `AddOrUpdateCustomerBody` | 是   | 包含会员信息的请求体 |
    | `cancellationToken`       | `CancellationToken`       | 是   | 用于取消操作的令牌   |

     - addOrUpdateCustomerBody : AddOrUpdateCustomerBody - 包含会员信息的请求体

        | 字段名            | 类型     | 必填 | 描述                                                         |
        | ----------------- | -------- | ---- | ------------------------------------------------------------ |
        | `FirstName`       | `string` | 是   | 会员的名字                                                   |
        | `LastName`        | `string` | 是   | 会员的姓氏                                                   |
        | `DisplayName`     | `string` | 是   | 会员的昵称                                                   |
        | `Phone`           | `string` | 是   | 会员的手机号                                                 |
        | `Email`           | `string` | 是   | 会员的邮箱                                                   |
        | `Language`        | `string` | 是   | 会员的语言， 目前会员只有两种语言，英文（LanguageCode.EnUs）、繁体中文（LanguageCode.ZhTw） |
        | `IsSubSms`        | `bool`   | 是   | 会员短信订阅状态                                             |
        | `IsSubEmail`      | `bool`   | 否   | 会员Email订阅状态                                            |
        | `IsSubOrderSms`   | `bool`   | 否   | 会员订单短信订阅状态                                         |
        | `IsSubOrderEmail` | `bool`   | 否   | 会员订单Email订阅状态                                        |


- 返回参数 :

    - AddOrUpdateCustomerResponse : 包含操作结果的响应对象

        | 字段名     | 类型          | 描述     |
        | ---------- | ------------- | -------- |
        | `Customer` | `CustomerDto` | 会员信息 |

    - CustomerDto字段说明

        | 字段名             | 类型             | 描述                       |
        | ------------------ | ---------------- | -------------------------- |
        | `Id`               | `long`           | 会员管理系统的会员Id       |
        | `MerchId`          | `long`           | 会员管理系统中的 MerchId   |
        | `DisplayName`      | `string`         | 会员的昵称                 |
        | `FirstName`        | `string`         | 会员的FirstName            |
        | `LastName`         | `string`         | 会员的LastName             |
        | `Phone`            | `string`         | 会员的手机号               |
        | `Sex`              | `Gender`         | 会员的性别                 |
        | `Birthday`         | `DateTime`       | 会员的生日                 |
        | `IsSubSms`         | `bool`           | 短信订阅状态               |
        | `IsSubEmail`       | `Gender`         | 邮箱订阅状态               |
        | `IsSubOrderSms`    | `DateTime`       | 短信订单推送通知订阅状态   |
        | `IsSubOrderEmail`  | `bool`           | 邮箱订单推动通知订阅状态   |
        | `Language`         | `string`         | 会员语言                   |
        | `Address`          | `string`         | 地址                       |
        | `NumberOfDiners`   | `DateTime`       | 会员的用餐人数             |
        | `Emails`           | `List<string>`   | 该会员在该店铺下的邮箱列表 |
        | `CreatedDate`      | `DateTimeOffset` | 会员创建时间               |
        | `LastModifiedDate` | `DateTimeOffset` | 会员最后一次修改时间       |

### 2.2. UpdateCustomerEmailAsync（更新会员邮箱）

- 功能 : 更新会员的邮箱


- 请求参数 :

    | 字段名              | 类型                | 必填 | 描述               |
    | ------------------- | ------------------- | ---- | ------------------ |
    | `phone`             | `string`            | 是   | 会员的手机号       |
    | `email`             | `string`            | 是   | 新的邮箱地址       |
    | `cancellationToken` | `CancellationToken` | 是   | 用于取消操作的令牌 |


- 返回参数 :

    - UpdateCustomerEmailResponse : 包含操作结果的响应对象

        | 字段名     | 类型          | 描述     |
        | ---------- | ------------- | -------- |
        | `Customer` | `CustomerDto` | 会员信息 |



### 2.3. UpdateCustomerLanguageAsync（更新会员语言设置）


- 功能 : 更新会员的语言设置


- 请求参数 :

    | 字段名              | 类型                | 必填 | 描述               |
    | ------------------- | ------------------- | ---- | ------------------ |
    | `phone`             | `string`            | 是   | 会员的手机号       |
    | `languageCode`      | `string`            | 是   | 会员语言           |
    | `cancellationToken` | `CancellationToken` | 是   | 用于取消操作的令牌 |


- 返回参数 :

    - UpdateCustomerLanguageResponse : 包含操作结果的响应对象

        | 字段名     | 类型          | 描述     |
        | ---------- | ------------- | -------- |
        | `Customer` | `CustomerDto` | 会员信息 |


### 2.4. UpdateCustomerOrderEmailSubStatusAsync（更新会员邮件订阅状态）


- 功能 : 更新会员的邮件订阅状态



- 请求参数 :

    | 字段名              | 类型                | 必填 | 描述               |
    | ------------------- | ------------------- | ---- | ------------------ |
    | `phone`             | `string`            | 是   | 会员的手机号       |
    | `isSubOrderEmail`   | `bool`              | 是   | 是否订阅邮件       |
    | `cancellationToken` | `CancellationToken` | 是   | 用于取消操作的令牌 |


- 返回参数 :

    - UpdateCustomerIsSubOrderEmailResponse : 包含操作结果的响应对象

        | 字段名     | 类型          | 描述     |
        | ---------- | ------------- | -------- |
        | `Customer` | `CustomerDto` | 会员信息 |


### 2.5. GetCustomerDetailAsync（获取会员详情）


- 功能 : 获取会员详情


- 请求参数 :

    | 字段名              | 类型                | 必填 | 描述               |
    | ------------------- | ------------------- | ---- | ------------------ |
    | `phones`            | `List<string>`      | 是   | 会员的手机号       |
    | `cancellationToken` | `CancellationToken` | 是   | 用于取消操作的令牌 |


- 返回参数 :

    - GetCustomerDetailResponse : 包含会员详情的响应对象

        | 字段名      | 类型                | 描述     |
        | ----------- | ------------------- | -------- |
        | `Customers` | `List<CustomerDto>` | 会员信息 |



### 3、订单（OrderService）

- 在会员管理系统中，在同个店铺下，同个手机号的会员有且只有一个。与此同时，在同个店铺下，同个手机号，同个ExternalOrderId，也有且只有1个订单。


- 第三方系统在同步订单数据时，需要将自己系统订单数据，组装成Order（订单基本信息）、OrderItem（订单相关的Item，包括商品item、折扣item、小费Item等）、OrderPayment（订单支付信息），传到会员管理系统。
    会员系统将会根据传入的订单数据进行统计，以及创建营销活动等。


- Order数据：存储订单的基本信息，如第三方系统的订单号、下单时间、ready时间等。在一个会员，同个ExternalOrderId，有且只有一个订单。


- OrderItem数据：包括了下单时、退款时、取消时生成的订单相关的Item。OrderItem商品item、折扣item、小费Item等。

    - ExternalOrderItemId： 第三方系统的OrderItem
    - Subtype： 用于标记OrderItem是商品/Tip/折扣等
    - ItemType： 用于标记下单时生成的Item，还是退款/取消时生成的Item
    - ReversalItemType： 用于标记是退款产生的ReversalItem，还是取消时产生的ReversalItem

- OrderPayment数据：
    - PaymentMethod: 支付方式
    - SettlementAmount: 支付金额
    - PayTime： 支付时间

### 3.1 AddOrUpdateOrderAsync（新增/更新订单）

- 功能 : 新增或更新订单信息，会员管理系统会根据请求参数中的CustomerPhone+ExternalOrderId，查看这个手机号对应的会员，是否存在订单的ExternalOrderId等于请求参数的ExternalOrderId。
    如果存在，则更新订单相关的数据，否则就创建。


- 请求参数 （AddOrUpdateOrderBody）:

    | 字段名                                    | 类型                     | 必填 | 描述                                                         |
    | ----------------------------------------- | ------------------------ | ---- | ------------------------------------------------------------ |
    | `CustomerPhone`                           | `string`                 | 是   | 会员手机号， 会员系统将会根据这个手机号查找对应店铺下的会员  |
    | `ExternalOrderId`                         | `string`                 | 是   | 第三方系统标记order的唯一标识                                |
    | `ExternalMerchId`                         | `string`                 | 是   | 第三方系统的merchId                                          |
    | `OrderStatus`                             | `OrderStatus`            | 是   | 会员手机号， 会员管理系统的订单状态                          |
    | `DeliveryType`                            | `DeliveryType`           | 是   | 配送方式                                                     |
    | `ExternalMerchId`                         | `string`                 | 是   | 第三方系统的merchId                                          |
    | `DeliveryType`                            | `string`                 | 是   | 配送方式                                                     |
    | `DeliveryTypeDescription`                 | `string`                 | 是   | 配送类型的备注                                               |
    | `SerialNumber`                            | `string`                 | 是   | 订单序列号                                                   |
    | `PaymentType`                             | `OrderPaymentMethod`     | 是   | 支付方式                                                     |
    | `PaymentStatus`                           | `PaymentStatus`          | 是   | 支付状态                                                     |
    | `OrderDate`                               | `DateTimeOffset`         | 是   | 下单时间                                                     |
    | `SystemCalculatedReadyForPickupTime`      | `DateTimeOffset`         | 否   | 第三方系统计算的ready时间                                    |
    | `MerchantConfirmedDeliveryTimeRangeStart` | `DateTimeOffset`         | 否   | 配送的开始时间                                               |
    | `MerchantConfirmedDeliveryTimeRangeEnd`   | `DateTimeOffset`         | 否   | 配送的结束时间                                               |
    | `ReadyAt`                                 | `DateTimeOffset`         | 否   | 实际订单ready时间，pos没有ready状态，所以pos这边可以不用传这个字段 |
    | `CompletedAt`                             | `DateTimeOffset`         | 否   | 完成时间，pos那边的completed_at、 system_completed_date是一样的 |
    | `RejectedAt`                              | `DateTimeOffset`         | 否   | 拒单时间，pos那边没有拒单场景，pos也不用传                   |
    | `ReturnedAt`                              | `DateTimeOffset`         | 否   | 完成后退订单时间                                             |
    | `TurnedProcessingAt`                      | `DateTimeOffset`         | 否   | 接单时间                                                     |
    | `PaymentSubmittedDate`                    | `DateTimeOffset`         | 否   | 支付时间                                                     |
    | `DeliveryAt`                              | `DateTimeOffset`         | 否   | 配送时间                                                     |
    | `SystemCompletedDate`                     | `DateTimeOffset`         | 否   | 系统自动完成的时间                                           |
    | `OrderType`                               | `OrderType`              | 是   | 订单类型                                                     |
    | `Phone`                                   | `string`                 | 否   | 订单对应的手机号                                             |
    | `MealCode`                                | `string`                 | 否   | 取餐码                                                       |
    | `Email`                                   | `string`                 | 否   | 下单时设置的email                                            |
    | `OrderItems`                              | `List<OrderItemBody>`    | 是   | 该订单关联的order item，会员管理系统会根据该orderitem数据，计算订单的金额等 |
    | `OrderPayments`                           | `List<OrderPaymentBody>` | 是   | 该订单关联的order payment数据，这里不会包括退款的payment数据 |


  - OrderItemBody字段说明

    | 字段名                   | 类型                      | 必填 | 描述                                                         |
    | ------------------------ | ------------------------- | ---- | ------------------------------------------------------------ |
    | `ExternalOrderItemId`    | `string`                  | 是   | 第三方系统的OrderItemId，如果是Food类型的OrderItem，该字段必填 |
    | `Name`                   | `string`                  | 是   | OrderItem名，如果是Food类型的OrdeItem，该字段必填，需要填写为FoodName |
    | `Params`                 | `List<OrderItemParamDto>` | 是   | 规格选项                                                     |
    | `FoodItemAmount`         | `decimal`                 | 否   | 当orderItem. = FoodItem时，则该字段需要传值，表示商品的item金额（数量*单价），不含税、不含规格 |
    | `ParamsAmount`           | `decimal`                 | 否   | 当orderItem.Subtype = FoodItem时，则该字段需要传值，表示该商品的规格总金额 |
    | `TotalAmount`            | `decimal`                 | 是   | 该OrerItem的总金额，如果是FoodItem，Total Amount  = FoodItemAmount  + ParamsTotalAmount |
    | `Subtype`                | `OrderItemSubType`        | 是   | 标记该OrderItem是商品的、Tax、折扣、其他杂费                 |
    | `ItemType`               | `OrderItemType`           | 是   | OrderItem类型，标记该OrderItem是正常的还是退款的             |
    | `Quantity`               | `int`                     | 是   | 数量                                                         |
    | `Price`                  | `decimal`                 | 是   | OrderItem的单价                                              |
    | `Banner`                 | `string`                  | 是   | 商品图片                                                     |
    | `ExternalReversalItemId` | `string`                  | 否   | 如果这个reversal item是跟商品有关，则需要传该商品的对应的ExternalOrderItemId，标记该OrderItem是关联哪个商品item |
    | `PayRecipient`           | `PaymentParty`            | 是   | 支付的接收方                                                 |
    | `PayBy`                  | `PaymentParty`            | 是   | 支付的支付方                                                 |
    | `ReversalItemType`       | `OrderReversalItemType`   | 否   | 当这个Item是退款/去取消时产生时，该字段需要传值，标记是取消还是退款时产生的 |


- OrderPaymentBody字段说明

    | 字段名                     | 类型             | 必填 | 描述           |
    | -------------------------- | ---------------- | ---- | -------------- |
    | `PaymentMethod`            | `string`         | 是   | 支付方式       |
    | `PaymentMethodDescription` | `string`         | 是   | 支付方式的描述 |
    | `SettlementAmount`         | `decimal`        | 是   | 支付金额       |
    | `PayTime`                  | `DateTimeOffset` | 否   | 支付时间       |



- 返回参数 （AddOrUpdateOrderResponse）:

    - AddOrUpdateOrderResponse类型，是OrderDto的继承类

        - OrderDto字段说明 

            | 字段名                                    | 类型                     | 描述                                                         |
            | ----------------------------------------- | ------------------------ | ------------------------------------------------------------ |
            | `Id`                                      | `long`                   | 会员系统的订单Id                                             |
            | `CustomerId`                              | `long`                   | 会员系统的会员Id                                             |
            | `MerchId`                                 | `long`                   | 会员系统的商家Id                                             |
            | `ExternalOrderId`                         | `string`                 | 第三方系统的订单Id                                           |
            | `ExternalMerchId`                         | `string`                 | 第三方系统的商家Id                                           |
            | `OrderStatus`                             | `OrderStatus`            | 订单状态                                                     |
            | `DeliveryType`                            | `DeliveryType`           | 配送方式                                                     |
            | `DeliveryTypeDescription`                 | `string`                 | 配送类型的备注                                               |
            | `SerialNumber`                            | `string`                 | 订单序列号                                                   |
            | `PaymentType`                             | `OrderPaymentMethod`     | 支付方式                                                     |
            | `PaymentStatus`                           | `PaymentStatus`          | 支付状态                                                     |
            | `OrderDate`                               | `DateTimeOffset`         | 下单时间                                                     |
            | `SystemCalculatedReadyForPickupTime`      | `DateTimeOffset`         | 第三方系统计算的ready时间                                    |
            | `MerchantConfirmedDeliveryTimeRangeStart` | `DateTimeOffset`         | 配送的开始时间                                               |
            | `MerchantConfirmedDeliveryTimeRangeEnd`   | `DateTimeOffset`         | 配送的结束时间                                               |
            | `ReadyAt`                                 | `DateTimeOffset`         | 实际订单ready时间，pos没有ready状态，所以pos这边可以不用传这个字段 |
            | `CompletedAt`                             | `DateTimeOffset`         | 完成时间，pos那边的completed_at、 system_completed_date是一样的 |
            | `RejectedAt`                              | `DateTimeOffset`         | 拒单时间，pos那边没有拒单场景，pos也不用传                   |
            | `ReturnedAt`                              | `DateTimeOffset`         | 完成后退订单时间                                             |
            | `TurnedProcessingAt`                      | `DateTimeOffset`         | 接单时间                                                     |
            | `PaymentSubmittedDate`                    | `DateTimeOffset`         | 支付时间                                                     |
            | `DeliveryAt`                              | `DateTimeOffset`         | 配送时间                                                     |
            | `SystemCompletedDate`                     | `DateTimeOffset`         | 系统自动完成的时间                                           |
            | `OrderType`                               | `OrderType`              | 订单类型                                                     |
            | `Phone`                                   | `string`                 | 订单对应的手机号                                             |
            | `MealCode`                                | `string`                 | 取餐码                                                       |
            | `Email`                                   | `string`                 | 下单时设置的email                                            |
            | `CreatedDate`                             | `DateTimeOffset`         | 订单创建的时间                                               |
            | `LastModifiedDate`                        | `DateTimeOffset`         | 订单最后一次修改的时间                                       |
            | `OrderItems`                              | `List<OrderItemDto>`     | 该订单关联的order item，会员管理系统会根据该orderitem数据，计算订单的金额等 |
            | `OrderPayments`                           | `List<OrderPaymentBody>` | 该订单关联的order payment数据，这里不会包括退款的payment数据 |

        - OrderItemDto字段说明 

            | 字段名                | 类型                      | 描述                                                         |
            | --------------------- | ------------------------- | ------------------------------------------------------------ |
            | `Id`                  | `long`                    | 会员系统的OrderItemId                                        |
            | `OrderId`             | `long`                    | 会员系统的订单Id                                             |
            | `ExternalOrderItemId` | `string`                  | 第三方系统的OrderItemId                                      |
            | `Name`                | `string`                  | OrderItem名                                                  |
            | `Params`              | `List<OrderItemParamDto>` | 规格选项                                                     |
            | `FoodItemAmount`      | `decimal`                 | 商品总额（不含税、不含规格的）                               |
            | `ParamsAmount`        | `decimal`                 | 商品的规格总金额                                             |
            | `TotalAmount`         | `decimal`                 | 该OrerItem的总金额，如果是FoodItem，Total Amount  = FoodItemAmount  + ParamsTotalAmount |
            | `Subtype`             | `OrderItemSubType`        | 标记该OrderItem是商品的、Tax、折扣、其他杂费                 |
            | `ItemType`            | `OrderItemType`           | OrderItem类型，标记该OrderItem是正常的还是退款的             |
            | `Quantity`            | `int`                     | 数量                                                         |
            | `Price`               | `decimal`                 | OrderItem的单价                                              |
            | `Banner`              | `string`                  | 商品图片                                                     |
            | `ReversalItemId`      | `long`                    | ReversalItem类型的Item，对应的NormalItem的Id                 |
            | `PayRecipient`        | `PaymentParty`            | 支付的接收方                                                 |
            | `PayBy`               | `PaymentParty`            | 支付的支付方                                                 |
            | `ReversalItemType`    | `OrderReversalItemType`   | 当这个Item是退款/去取消时产生时，该字段需要传值，标记是取消还是退款时产生的 |

        - OrderItemParamDto字段说明

            | 字段名       | 类型                      | 描述                     |
            | ------------ | ------------------------- | ------------------------ |
            | `Id`         | `string`                  | 第三方系统的规格选项id   |
            | `Name`       | `string`                  | 第三方系统的规格选项名称 |
            | `Quantity`   | `int`                     | 规格选项的数量           |
            | `price`      | `decimal`                 | 规格选项的单价           |
            | `ParamItems` | `List<OrderItemParamDto>` | 子规格选项               |

        - OrderPaymentDto字段说明

            | 字段名                     | 类型             | 描述                     |
            | -------------------------- | ---------------- | ------------------------ |
            | `Id`                       | `long`           | 会员系统的OrderPaymentId |
            | `OrderId`                  | `long`           | 会员系统的订单Id         |
            | `CreatedDate`              | `DateTimeOffset` | 创建时间                 |
            | `PaymentMethod`            | `PaymentMethod`  | 支付方式                 |
            | `PaymentMethodDescription` | `string`         | 支付方式备注             |
            | `SettlementAmount`         | `decimal`        | 支付金额                 |
            | `PayTime`                  | `DateTimeOffset` | 支付时间                 |
            | `LastModifiedDate`         | `DateTimeOffset` | 最后一次修改时间         |


- 示例 :

```csharp
var response = await orderService.AddOrUpdateOrderAsync(new AddOrUpdateOrderBody
{
    OrderDate = DateTime.Now,
    OrderStatus = OrderStatus.Pending,
    PaymentStatus = PaymentStatus.Paid,
    OrderItems = new List<OrderItemDto>
    {
        new OrderItemDto
        {
            ItemName = "Product A",
            Quantity = 1,
            Price = 100.0m
        }
    }
}, CancellationToken.None);

```

### 3.2 BatchAddOrUpdateOrderAsync（批量新增/更新订单）


- 功能 : 批量新增/更新订单数据，单次最多只能同步300个订单


- 请求参数 （AddOrUpdateOrderBody）:

    | 字段名              | 类型                         | 必填 | 描述                |
    | ------------------- | ---------------------------- | ---- | ------------------- |
    | `Orders`            | `List<AddOrUpdateOrderBody>` | 是   | 新增/更新的订单列表 |
    | `cancellationToken` | `CancellationToken`          | 是   | 用于取消操作的令牌  |

    

- 返回参数 （BatchAddOrUpdateOrderResponse）：
          

    | 字段名   | 类型                             | 描述                      |
    | -------- | -------------------------------- | ------------------------- |
    | `Orders` | `List<AddOrUpdateOrderResponse>` | 新增/更新后的订单列表数据 |



- 示例 :

```csharp
var response = await orderService.BatchAddOrUpdateOrderAsync(new BatchAddOrUpdateOrderBody
{
    Orders = new List<AddOrUpdateOrderBody>
    {
        new AddOrUpdateOrderBody
        {
            OrderDate = DateTime.Now,
            OrderStatus = OrderStatus.Pending,
            PaymentStatus = PaymentStatus.Paid,
            OrderItems = new List<OrderItemDto>
            {
                new OrderItemDto
                {
                    ItemName = "Product A",
                    Quantity = 1,
                    Price = 100.0m
                }
            }
        }
    }
}, CancellationToken.None);
```


### 3.3 GetOrderListAsync（获取订单列表）


- 功能 : 分页获取当前渠道同步的订单列表


- 请求参数 :

    | 字段名                | 类型                  | 必填 | 描述                 |
    | --------------------- | --------------------- | ---- | -------------------- |
    | `GetOrderListRequest` | `GetOrderListRequest` | 是   | 分页获取请求订单列表 |
    | `cancellationToken`   | `CancellationToken`   | 是   | 用于取消操作的令牌   |



- 返回参数 :

    - GetOrderListResponse返回参数说明

        | 字段名      | 类型             | 描述     |
        | ----------- | ---------------- | -------- |
        | `PageIndex` | `int`            | 当前页数 |
        | `PageSize`  | `int`            | 页大小   |
        | `Total`     | `int`            | 总订单数 |
        | `Orders`    | `List<OrderDto>` | 订单列表 |

- 示例 :

```csharp
var response = await orderService.GetOrderListAsync(new GetOrderListRequest
{
    StartDate = DateTime.Now.AddDays(-7),
    EndDate = DateTime.Now
}, CancellationToken.None);
```
