# 华为应用内支付 IAP Android 集成技术文档

华为开发者联盟 HMS Core 应用内支付服务文档  
入口文档地址：[订单、订阅服务](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/phone-pad-0000001050035011)  
Android 接入文档地址：[Android](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/android-0000001265624218)  

## 1. 适用范围

商品类型与 `priceType` 对应关系：

| 商品类型 | `priceType` | 说明 |
| --- | ---: | --- |
| 消耗型商品 | `0` | 可重复购买。发货成功后必须消耗，否则用户可能无法再次购买同一商品。 |
| 非消耗型商品 | `1` | 买断型权益。购买后通过已购查询恢复权益。 |
| 订阅型商品 | `2` | 周期性续费权益。需要处理订阅有效性、续期、取消、恢复和切换。 |

## 2. 开发环境要求

| 项目 | 要求 |
| --- | --- |
| JDK | 1.8 及以上 |
| Android Studio | 3.6.1 及以上 |
| `minSdkVersion` | 19 及以上 |
| `targetSdkVersion` | 推荐 34 |
| `compileSdkVersion` | 推荐 34 |
| Gradle | 5.4.1 及以上 |
| 测试设备 | EMUI 3.0 及以上华为手机，或 Android 4.4 及以上非华为手机 |
| HMS Core APK | IAP SDK 最新版本要求设备安装 HMS Core APK 4.0.4.300 及以上 |

## 3. AppGallery Connect 准备

1. 注册华为开发者并完成实名认证。
2. 开通商户服务，用于接收应用内商品收益。
3. 在 AppGallery Connect 创建项目和 Android 应用。
4. 使用签名证书生成 SHA256 证书指纹，并在项目设置中配置。
5. 在“API 管理”中打开“应用内支付服务”开关。
6. 进入“盈利 > 应用内支付服务 > 设置”，完成支付服务参数配置。
7. 保存 IAP 公钥。客户端或服务端校验 `InAppPurchaseData` 签名时需要使用该公钥。
8. 按需开启“订单/订阅关键事件通知”，服务端通过通知及时处理订单、订阅和延迟付款状态变化。

生成 SHA256 指纹示例：

```powershell
keytool -list -v -keystore C:\TestApp.jks
```

下载 `agconnect-services.json` 后，将文件放到 Android 应用模块根目录，例如 `app/agconnect-services.json`。

## 4. 集成 HMS Core IAP SDK

### 4.1 配置 Maven 仓库

Gradle 插件 7.1 及以上推荐在项目级 `settings.gradle` 中配置：

```gradle
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
        maven { url 'https://developer.huawei.com/repo/' }
    }
}

dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://developer.huawei.com/repo/' }
    }
}
```

项目级 `build.gradle` 添加 AGC 插件依赖。版本请按 AGC 插件依赖关系选择，华为文档示例为：

```gradle
buildscript {
    dependencies {
        classpath 'com.huawei.agconnect:agcp:1.6.0.300'
    }
}
```

### 4.2 添加应用模块依赖

应用级 `build.gradle`：

```gradle
plugins {
    id 'com.android.application'
    id 'com.huawei.agconnect'
}

dependencies {
    implementation 'com.huawei.hms:iap:6.13.0.300'
}
```

如果应用集成 IAP SDK 6.13.0.300 及以上版本，且应用上架华为应用市场，可选接入 HMS Core Installer SDK，引导用户安装或更新 HMS Core APK。若应用上架其他应用市场，例如 Google Play，华为文档建议跳过该配置。

```gradle
dependencies {
    implementation 'com.huawei.hms:hmscoreinstaller:6.11.0.302'
}
```

### 4.3 配置签名

应用签名必须与 AppGallery Connect 中配置的 SHA256 指纹一致：

```gradle
android {
    signingConfigs {
        releaseConfig {
            storeFile file('your-release-key.jks')
            keyAlias 'your-key-alias'
            keyPassword 'your-key-password'
            storePassword 'your-store-password'
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.releaseConfig
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

### 4.4 混淆配置

`proguard-rules.pro` 添加 HMS Core 保留规则：

```proguard
-ignorewarnings
-keepattributes *Annotation*
-keepattributes Exceptions
-keepattributes InnerClasses
-keepattributes Signature
-keepattributes SourceFile,LineNumberTable
-keep class com.huawei.hianalytics.**{*;}
-keep class com.huawei.updatesdk.**{*;}
-keep class com.huawei.hms.**{*;}
```

## 5. 商品配置

如果使用 PMS 商品，需要先在 AppGallery Connect 配置商品信息，包括商品 ID、商品类型、销售国家/地区、价格、商品名称和描述。

注意事项：

- 客户端调用购买接口时传入的商品 ID 和商品类型必须与 AppGallery Connect 配置一致。
- `obtainProductInfo` 每次只能查询一种商品类型。
- 单次商品查询不要超过 200 条。
- 商品价格不会跟随汇率自动实时刷新，需要在 AppGallery Connect 中定期手动刷新并保存。
- 开发阶段可配置沙盒测试账号验证购买流程。

## 6. Android 端支付流程

### 6.1 检查支付环境

调用 `Iap.getIapClient(activity).isEnvReady()` 检查用户账号和服务地是否支持 IAP。若返回未登录，可通过 `Status.startResolutionForResult()` 拉起华为账号登录。

```java
Task<IsEnvReadyResult> task = Iap.getIapClient(activity).isEnvReady();
task.addOnSuccessListener(result -> {
    String carrierId = result.getCarrierId();
}).addOnFailureListener(e -> {
    if (e instanceof IapApiException) {
        IapApiException apiException = (IapApiException) e;
        Status status = apiException.getStatus();
        if (status.getStatusCode() == OrderStatusCode.ORDER_HWID_NOT_LOGIN && status.hasResolution()) {
            status.startResolutionForResult(activity, REQ_CODE_IAP);
        }
    }
});
```

### 6.2 查询商品信息

```java
ProductInfoReq req = new ProductInfoReq();
req.setPriceType(0);
req.setProductIds(Collections.singletonList("ConsumeProduct1001"));

Iap.getIapClient(activity).obtainProductInfo(req)
    .addOnSuccessListener(result -> {
        List<ProductInfo> productList = result.getProductInfoList();
    })
    .addOnFailureListener(e -> {
        int code = e instanceof IapApiException
            ? ((IapApiException) e).getStatusCode()
            : -1;
    });
```

### 6.3 发起购买

PMS 商品使用 `createPurchaseIntent`：

```java
PurchaseIntentReq req = new PurchaseIntentReq();
req.setProductId("CProduct1");
req.setPriceType(0);
req.setDeveloperPayload("hashed-user-or-account-id");

Iap.getIapClient(activity).createPurchaseIntent(req)
    .addOnSuccessListener(result -> {
        Status status = result.getStatus();
        if (status.hasResolution()) {
            status.startResolutionForResult(activity, REQ_CODE_IAP);
        }
    });
```

`developerPayload` 建议传入与应用账号稳定关联的哈希或加密标识，不建议直接传业务订单号，也不要传明文隐私数据。

### 6.4 处理支付回调

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if (requestCode != REQ_CODE_IAP || data == null) {
        return;
    }

    PurchaseResultInfo resultInfo = Iap.getIapClient(this).parsePurchaseResultInfoFromIntent(data);
    switch (resultInfo.getReturnCode()) {
        case OrderStatusCode.ORDER_STATE_SUCCESS:
            String inAppPurchaseData = resultInfo.getInAppPurchaseData();
            String signature = resultInfo.getInAppDataSignature();
            // 先验签和校验订单字段，再发放权益。
            // 消耗型商品发货成功后，再调用 consumeOwnedPurchase。
            break;
        case OrderStatusCode.ORDER_STATE_CANCEL:
            // 用户取消。
            break;
        case OrderStatusCode.ORDER_STATE_FAILED:
        case OrderStatusCode.ORDER_PRODUCT_OWNED:
        case OrderStatusCode.ORDER_STATE_DEFAULT_CODE:
            // 触发补单查询，检查是否存在已购未发货商品。
            break;
        default:
            break;
    }
}
```

## 7. 验单、发货和消耗

支付成功后不要直接发货，应先完成以下校验：

1. 使用 IAP 公钥验证 `InAppPurchaseData` 和签名字符串。
2. 校验 `productId`、`price`、`currency` 等字段与业务订单一致。
3. 检查购买状态，确认订单已支付。
4. 服务端做幂等控制，确保同一笔购买只发货一次。
5. 发货成功后，消耗型商品再调用 `consumeOwnedPurchase` 消耗，或由服务端调用确认购买接口。

消耗型商品客户端消耗示例：

```java
InAppPurchaseData data = new InAppPurchaseData(inAppPurchaseData);

ConsumeOwnedPurchaseReq req = new ConsumeOwnedPurchaseReq();
req.setPurchaseToken(data.getPurchaseToken());

Iap.getIapClient(activity).consumeOwnedPurchase(req)
    .addOnSuccessListener(result -> {
        // 消耗成功。
    });
```

如果具备服务端能力，华为文档建议服务端进一步验证购买 Token。购买 Token 验证适用于消耗型商品和非消耗型商品。

## 8. 补单与恢复权益

消耗型商品必须实现补单。以下场景应调用 `IapClient.obtainOwnedPurchases` 查询已购但未发货商品：

- 应用启动时。
- 购买请求返回 `ORDER_STATE_FAILED`。
- 购买请求返回 `ORDER_PRODUCT_OWNED`。
- 购买请求返回 `ORDER_STATE_DEFAULT_CODE`。

```java
OwnedPurchasesReq req = new OwnedPurchasesReq();
req.setPriceType(0);

Iap.getIapClient(activity).obtainOwnedPurchases(req)
    .addOnSuccessListener(result -> {
        List<String> purchaseDataList = result.getInAppPurchaseDataList();
        List<String> signatureList = result.getInAppSignature();
        // 逐条验签、校验订单字段、确认购买状态、幂等发货，发货成功后消耗。
    });
```

非消耗型商品和订阅型商品也可通过 `obtainOwnedPurchases` 查询当前有效权益。历史购买记录可通过 `obtainOwnedPurchaseRecord` 查询。

## 9. 订阅型商品

订阅型商品需要先创建订阅组，再在订阅组下创建订阅商品。同一订阅组承载同类权益，不同订阅商品可对应不同续费周期。

订阅处理要点：

- 用户对每个订阅组至多享受一次促销。
- 续期前约 10 天确定下一个续期价格。
- 订阅周期结束前约 24 小时，IAP 会尝试续费扣款。
- 订阅可能处于续期、到期、已到期、待生效等状态。
- 查询订阅权益时应验签，并检查 `purchaseState` 和 `isSubValid()`。
- 可通过 `startIapActivity` 跳转到 HMS Core APK 的订阅管理页或编辑订阅页。

订阅查询示例：

```java
OwnedPurchasesReq req = new OwnedPurchasesReq();
req.setPriceType(2);

Iap.getIapClient(activity).obtainOwnedPurchases(req)
    .addOnSuccessListener(result -> {
        for (String item : result.getInAppPurchaseDataList()) {
            InAppPurchaseData data = new InAppPurchaseData(item);
            int purchaseState = data.getPurchaseState();
            boolean valid = data.isSubValid();
        }
    });
```

## 10. 延迟付款和好友代付

延迟付款型支付表示用户下单后不会立即完成付款，订单会先进入 Pending 状态。订单从 Pending 变为 Purchased 后，才可以发货。

接入要点：

- 应用初始化时调用 `enablePendingPurchase` 开启延迟付款能力。
- 仅手机和平板支持延迟付款。
- 仅 `createPurchaseIntent` 购买一次性商品场景支持延迟付款，包括消耗型商品和非消耗型商品。
- Pending 返回码为 `60057`，即 `OrderStatusCode.ORDER_STATE_PENDING`。
- 服务端应接入延迟付款关键事件通知，及时刷新订单状态并发货。

好友代付属于延迟付款型支付，需要 IAP SDK 6.10.0.300 及以上版本，不支持沙盒测试。发起好友代付时在 `PurchaseIntentReq` 中传入：

```java
req.setReservedInfor("{\"PayTypePolicy\":{\"DefaultPayment\":\"entrusted\"}}");
```

## 11. 上线前检查清单

- AppGallery Connect 已开通应用内支付服务。
- 商户服务、支付服务参数、公钥和签名证书指纹已配置。
- `agconnect-services.json` 已放入应用模块根目录。
- Gradle 已配置华为 Maven 仓、AGC 插件和 IAP SDK 依赖。
- Release 签名与 AppGallery Connect SHA256 指纹一致。
- 混淆规则已配置。
- PMS 商品已创建并生效，商品 ID、类型和价格配置正确。
- 客户端已接入 `isEnvReady`、`obtainProductInfo`、`createPurchaseIntent`、支付回调、补单、验签和消耗。
- 服务端已实现订单幂等、验签、购买 Token 验证和关键事件通知处理。
- 消耗型商品已实现应用启动和异常返回场景的补单。
- 发货成功后再调用消耗或确认购买接口。
- 沙盒测试通过。
- 提交上架前完成华为“支付服务开发自检 Checklist”。

## 12. 参考文档地址

- [订单、订阅服务](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/phone-pad-0000001050035011)
- [Android](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/android-0000001265624218)
- [使用入门](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/dev-process-0000001050033070)
- [配置 AppGallery Connect](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/config-agc-0000001050033072)
- [集成 HMS Core SDK](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/integrating-sdk-0000001050035023)
- [配置商品信息](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/config-product-0000001050033076)
- [开发商品购买](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/dev-guide-0000001050130254)
- [消耗型商品的补单流程](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/redelivering-consumables-0000001051356573)
- [验证 InAppPurchaseData](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/verifying-inapppurchasedata-0000001494212281)
- [Order 服务验证购买 Token](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/order-verify-purchase-token-0000001050033078)
- [订阅专用功能说明](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/subscription-functions-0000001050130264)
- [处理延迟付款型支付](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/process-pending-payment-0000001180993022)
- [处理好友代付](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/process-entrusted-payment-0000001521998653)
- [开发后自检](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/pre-release-check-0000001076257572)
- [版本更新说明](https://developer.huawei.com/consumer/cn/doc/HMSCore-Guides/version-change-history-0000001050065947)
