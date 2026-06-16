# RuStore PaySDK in-app 支付接入文档

> 官方入口：[RuStore SDK 文档](https://www.rustore.ru/help/zh/sdk)、[应用内支付 PaySDK](https://www.rustore.ru/help/zh/sdk/pay)

[一、接入前准备](#一接入前准备)
* [1. 后台配置](#1-后台配置)
* [2. 推荐支付流程](#2-推荐支付流程)

[二、Android Kotlin/Java 接入](#二android-kotlinjava-接入)
* [1. 添加 Maven 仓库与 SDK 依赖](#1-添加-maven-仓库与-sdk-依赖)
* [2. 配置 AndroidManifest](#2-配置-androidmanifest)
* [3. 配置 Deeplink 回跳](#3-配置-deeplink-回跳)
* [4. 检查支付可用性](#4-检查支付可用性)
* [5. 获取商品信息](#5-获取商品信息)
* [6. 发起购买](#6-发起购买)
* [7. 查询购买记录](#7-查询购买记录)
* [8. 确认与取消两阶段购买](#8-确认与取消两阶段购买)
* [9. 异常处理](#9-异常处理)

[三、Unity 接入要点](#三unity-接入要点)

[四、其他平台文档入口](#四其他平台文档入口)

[五、测试与验收](#五测试与验收)

[六、常见问题](#六常见问题)


---

## 一、接入前准备


### 1. 后台配置

接入前需要在 [RuStore Console](https://console.rustore.ru/sign-in) 完成以下配置：

* 应用已创建，并且包名、签名与客户端构建一致。
* 应用通过审核；通常不要求已正式发布，但需要完成必要的应用审核流程。
* 已开通应用变现能力。
* 在「变现」中创建应用内商品，记录商品 ID。
* 记录 Console 应用 ID。官方示例中，`https://console.rustore.ru/apps/111111` 里的 `111111` 就是应用 ID。

### 3. 推荐支付流程

建议业务流程如下：

1. 客户端启动或进入商店页时调用 `getPurchaseAvailability()`。
2. 可用时调用 `getProducts()` 拉取 RuStore 后台配置的商品信息。
3. 用户点击购买后，客户端用自有订单号生成业务订单，并把订单号写入 `developerPayload`。
4. 调用 `purchase()` 或 `purchaseTwoStep()` 发起支付。
5. 支付返回成功后，客户端把 `purchaseId`、`invoiceId`、`subscriptionToken` 或相关结果提交到服务端。
6. 服务端调用 RuStore API 或处理服务端通知，校验订单状态。
7. 校验通过后发货。
8. 两阶段支付场景，发货成功后调用 `confirmTwoStepPurchase()` 完成确认；无法发货时谨慎调用 `cancelTwoStepPurchase()`。

---

## 二、Android Kotlin/Java 接入

官方当前 Kotlin/Java PaySDK 文档版本为 `10.4.0`。

### 1. 添加 Maven 仓库与 SDK 依赖

在项目 Gradle 中添加 RuStore Maven 仓库：

```gradle
repositories {
    maven {
        url = uri("https://artifactory-external.vkpartner.ru/artifactory/maven-rustore-exposed/")
    }
}
```

在 App 模块添加依赖：

```gradle
dependencies {
    implementation(platform("ru.rustore.sdk:bom:2026.05.01"))
    implementation("ru.rustore.sdk:pay")
}
```

PaySDK 依赖的部分 Android 库版本如下，冲突时优先检查这些版本：

```gradle
androidx.core:core-ktx:1.12.0
androidx.activity:activity-ktx:1.7.0
androidx.fragment:fragment-ktx:1.4.0
androidx.lifecycle:lifecycle-runtime-ktx:2.7.0
androidx.lifecycle:lifecycle-common-java8:2.7.0
androidx.recyclerview:recyclerview:1.2.1
com.google.android.material:material:1.5.0
io.coil-kt:coil:2.6.0
com.dpforge:primaree:1.2.0
```

### 2. 配置 AndroidManifest

在 `AndroidManifest.xml` 的 `<application>` 下配置 Console 应用 ID：

```xml
<manifest
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="your.app.package.name">

    <application
        android:theme="@style/Theme.App"
        tools:targetApi="n">

        <meta-data
            android:name="console_app_id_value"
            android:value="@string/CONSOLE_APPLICATION_ID" />

    </application>
</manifest>
```

注意事项：

* `CONSOLE_APPLICATION_ID` 填 RuStore Console 中的应用 ID，不是包名。
* `build.gradle` 中的 `applicationId` 必须与 RuStore Console 中的应用包名一致。
* 测试包签名要与 RuStore 后台登记的签名一致，否则可用性检查和支付流程可能失败。

### 3. 配置 Deeplink 回跳

PaySDK 需要通过 deeplink 回到应用。先在 `AndroidManifest.xml` 里配置 `sdk_pay_scheme_value`：

```xml
<activity
    android:name=".YourPayActivity"
    android:exported="true"
    android:launchMode="singleTop">

    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="yourappscheme" />
    </intent-filter>
</activity>

<meta-data
    android:name="sdk_pay_scheme_value"
    android:value="yourappscheme" />
```

`yourappscheme` 建议使用能唯一标识应用的 scheme，例如 `ru.package.name.rustore.scheme`。scheme 必须符合 URI scheme 规则，不要使用中文、空格或特殊符号。

然后在回跳 Activity 中处理 intent：

```kotlin
class YourPayActivity : AppCompatActivity() {

    private val intentInteractor: IntentInteractor by lazy {
        RuStorePayClient.instance.getIntentInteractor()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        if (savedInstanceState == null) {
            intentInteractor.proceedIntent(intent, sdkTheme = SdkTheme.LIGHT)
        }
    }

    override fun onNewIntent(intent: Intent?) {
        super.onNewIntent(intent)
        intentInteractor.proceedIntent(intent, sdkTheme = SdkTheme.LIGHT)
    }
}
```

### 4. 检查支付可用性

进入商店页或展示购买按钮前，先检查支付是否可用：

```java
PurchaseInteractor purchaseInteractor =
    RuStorePayClient.Companion.getInstance().getPurchaseInteractor();

purchaseInteractor.getPurchaseAvailability()
    .addOnSuccessListener(result -> {
        if (result instanceof PurchaseAvailabilityResult.Available) {
            // 展示商品和购买按钮
        } else if (result instanceof PurchaseAvailabilityResult.Unavailable) {
            // 展示不可购买原因或降级入口
        }
    })
    .addOnFailureListener(throwable -> {
        // 处理 SDK 或环境错误
    });
```

该接口会检查应用变现开通状态、RuStore 安装/版本、应用或用户是否被限制、当前环境是否可进行支付等条件。

### 5. 获取商品信息

通过商品 ID 列表获取 RuStore 后台配置的商品信息：

```java
List<ProductId> productsId = Arrays.asList(
    new ProductId("coins_100"),
    new ProductId("remove_ads")
);

ProductInteractor productInteractor =
    RuStorePayClient.Companion.getInstance().getProductInteractor();

productInteractor.getProducts(productsId)
    .addOnSuccessListener(products -> {
        // 使用 RuStore 返回的价格、标题、描述渲染商店页
    })
    .addOnFailureListener(throwable -> {
        // 处理商品不存在、网络错误或 SDK 错误
    });
```

建议不要把价格写死在客户端 UI 中，以 RuStore 返回结果为准。

### 6. 发起购买

单阶段支付示例：

```java
ProductPurchaseParams params = new ProductPurchaseParams(
    new ProductId("coins_100"),
    null,
    null,
    null,
    null,
    null
);

RuStorePayClient ruStorePayClient = RuStorePayClient.Companion.getInstance();
PurchaseInteractor purchaseInteractor = ruStorePayClient.getPurchaseInteractor();

purchaseInteractor.purchase(
    params,
    PreferredPurchaseType.ONE_STEP,
    SdkTheme.LIGHT,
    null
).addOnSuccessListener(result -> {
    // 将支付结果提交到服务端校验
}).addOnFailureListener(throwable -> {
    if (throwable instanceof ProductPurchaseCancelled) {
        // 用户取消支付
    } else if (throwable instanceof ProductPurchaseException) {
        // 支付失败
    } else {
        // 其他 SDK 错误
    }
});
```

两阶段支付可使用：

```java
purchaseInteractor.purchaseTwoStep(
    params,
    SdkTheme.LIGHT,
    null
).addOnSuccessListener(result -> {
    // 等待服务端校验与发货后，再确认购买
}).addOnFailureListener(throwable -> {
    // 处理取消或错误
});
```

说明：

* `purchase()` 默认可按 `PreferredPurchaseType.ONE_STEP` 走单阶段支付。
* 如果指定 `PreferredPurchaseType.TWO_STEP`，能否冻结资金还取决于用户选择的支付方式。
* `purchaseTwoStep()` 是明确发起两阶段购买的接口。
* 用户尚未选择支付方式前，两阶段状态可能是 `UNDEFINED`，回调处理中不要把它直接当成支付失败。

### 7. 查询购买记录

查询当前用户购买记录：

```java
PurchaseInteractor purchaseInteractor =
    RuStorePayClient.Companion.getInstance().getPurchaseInteractor();

purchaseInteractor.getPurchases(null, null, null)
    .addOnSuccessListener(purchases -> {
        // 恢复未发货订单、检查待确认订单
    })
    .addOnFailureListener(error -> {
        // 处理查询失败
    });
```

按类型和状态过滤：

```java
purchaseInteractor.getPurchases(
    ProductType.CONSUMABLE_PRODUCT,
    ProductPurchaseStatus.PAID,
    null
).addOnSuccessListener(purchases -> {
    // 处理已支付但未确认/未发货的消耗品
});
```

建议在以下时机查询购买记录：

* App 启动后。
* 用户进入商店页时。
* 支付中断或回跳失败后。
* 服务端发现订单已支付但客户端未发货时。

### 8. 确认与取消两阶段购买

两阶段支付在资金冻结后，开发者发货成功才应确认：

```java
PurchaseInteractor purchaseInteractor =
    RuStorePayClient.Companion.getInstance().getPurchaseInteractor();

purchaseInteractor.confirmTwoStepPurchase(
    new PurchaseId("purchaseId"),
    null
).addOnSuccessListener(success -> {
    // 标记业务订单完成
}).addOnFailureListener(throwable -> {
    // 记录并重试，避免订单长期停留在 PAID
});
```

无法发货时才考虑取消两阶段购买：

```java
purchaseInteractor.cancelTwoStepPurchase(
    new PurchaseId("purchaseId")
).addOnSuccessListener(success -> {
    // 标记业务订单取消
}).addOnFailureListener(throwable -> {
    // 处理取消失败
});
```

业务建议：

* 不要在客户端支付成功回调里直接发货，必须先做服务端校验。
* 发货接口需要幂等，防止回调、轮询、服务端通知重复触发。
* 订单状态达到官方最终成功态后再发货。
* 对 `PAID` 状态订单建立补偿任务，避免用户已付款但未到账。

### 9. 异常处理

常见 SDK 异常：

| 异常 | 含义 | 处理建议 |
| --- | --- | --- |
| `RuStorePaymentException` | PaySDK 异常基类 | 统一记录日志并上报 |
| `RuStorePayClientAlreadyExist` | SDK 重复初始化 | 检查初始化时机，避免多进程或重复初始化 |
| `RuStorePayClientNotCreated` | 未创建客户端就调用 SDK | 确认 manifest 与初始化流程 |
| `RuStorePayInvalidConsoleAppId` | Console App ID 缺失或非法 | 检查 `console_app_id_value` |
| `ProductPurchaseCancelled` | 用户取消购买 | 不弹错误，可提示“支付已取消” |
| `ProductPurchaseException` | 商品购买失败 | 展示失败提示，并记录错误码/上下文 |

---

## 三、Unity 接入要点

官方当前 Unity PaySDK 文档版本为 `10.5.0`。

Unity 项目建议按以下顺序接入：

1. 导入 RuStore Core 与 PaySDK Unity 插件。
2. 安装并启用 External Dependency Manager for Android。
3. 在 Unity Android Player Settings 中启用：
   * `Custom Main Manifest`
   * `Custom Main Gradle Template`
   * `Custom Gradle Properties Template`
4. 在 Gradle 模板中补充 Android 依赖。
5. 配置 `console_app_id_value`、`sdk_pay_scheme_value` 和 PayActivity。
6. 每次改动插件版本或 Gradle 模板后，执行：

```text
Assets -> External Dependency Manager -> Android Resolver -> Force Resolve
```

Unity 侧仍然需要遵守 Android 支付主流程：检查可用性、拉取商品、发起支付、服务端校验、发货、两阶段确认或取消。

---

## 四、其他平台文档入口

| 平台 | 当前官方入口 |
| --- | --- |
| Kotlin/Java | https://www.rustore.ru/help/zh/sdk/pay/kotlin-java/10-4-0 |
| Kotlin/Java 依赖列表 | https://www.rustore.ru/help/zh/sdk/pay/kotlin-java/dependencies |
| Unity | https://www.rustore.ru/help/zh/sdk/pay/unity/10-5-0 |
| Unreal Engine | https://www.rustore.ru/help/zh/sdk/pay/unreal/10-5-0 |
| Flutter | https://www.rustore.ru/help/zh/sdk/pay/flutter/10-3-1 |
| Godot | https://www.rustore.ru/help/zh/sdk/pay/godot/10-3-1 |
| Defold | https://www.rustore.ru/help/zh/sdk/pay/defold/10-3-1 |
| React Native | https://www.rustore.ru/help/zh/sdk/pay/react-native/10-3-1 |
| Construct | https://www.rustore.ru/help/zh/sdk/pay/construct/10-3-1 |

---

## 五、测试与验收

### 1. 本地检查

* App 包名与 RuStore Console 包名一致。
* 使用与 RuStore 后台一致的签名证书打包。
* `console_app_id_value` 已配置。
* `sdk_pay_scheme_value` 与 Activity intent-filter 中的 scheme 一致。
* 设备 Android 版本满足 SDK 要求；官方文档中 Android 平台要求为 Android 7.0 或更高。
* 设备安装当前版本 RuStore。

### 2. 支付链路检查

* `getPurchaseAvailability()` 返回可用。
* `getProducts()` 能返回后台配置的商品。
* 用户取消支付时不会发货。
* 支付成功但客户端崩溃后，重启 App 能通过 `getPurchases()` 恢复订单。
* 服务端能校验 RuStore 订单状态。
* 发货接口幂等。
* 两阶段支付在发货成功后能调用 `confirmTwoStepPurchase()`。
* 无法发货时能进入退款/取消补偿流程。

### 3. 上线前验收

* 使用正式包名、正式签名、正式 Console App ID 验证。
* 验证弱网、切后台、杀进程、支付后不回跳等异常场景。
* 验证服务端通知和客户端轮询不会重复发货。
* 客服和日志系统能根据业务订单号、`purchaseId` 或 `invoiceId` 定位问题。

---

## 六、常见问题

### 1. 是否还能使用旧版 billingClient SDK？

可以维护旧项目，但新接入建议使用 PaySDK。旧版 billingClient SDK 与 PaySDK 的接口、能力和迁移步骤不同，迁移时应参考官方迁移文档。

### 2. 为什么商品列表为空？

常见原因：

* 商品 ID 写错。
* 商品未在 RuStore Console 创建或未启用。
* App 包名、签名、Console App ID 不匹配。
* 变现能力未开通。
* 当前用户或应用状态不允许支付。

### 3. 支付成功后可以直接发货吗？

不建议。客户端回调只能作为用户体验流程的一部分，最终发货应以服务端校验或 RuStore 服务端通知为准。

### 4. 单阶段和两阶段怎么选？

虚拟币、道具等可以即时发货的场景通常使用单阶段。需要库存、人工审核、服务端复杂履约或可能无法立即发货的场景使用两阶段。

### 5. 两阶段支付为什么需要确认？

两阶段支付会先冻结资金。只有开发者确认购买后才完成扣款；如果无法发货，应按业务规则取消或退款，避免用户长期处于已付款但未到账状态。

### 6. PaySDK 当前是否支持订阅？

按当前官方 PaySDK 概览页，订阅能力仍属于临时限制，官方标注为后续支持。需要订阅能力时，应先确认最新官方状态。

---
