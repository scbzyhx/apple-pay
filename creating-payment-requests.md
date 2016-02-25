# 创建支付请求

支付请求是 [PKPayementRequest](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentRequest_Ref/index.html#//apple_ref/occ/cl/PKPaymentRequest) 类的一个实列。一个支持请求包含用户支付的物品概要清单、可选配送方式列表、用户需提供的配送信息、商家的信息以及支付处理机构。

## 判断用户是否能够支付

创建支付请求前，可以先通过调用 [PKPaymentAuthorizationViewController](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentAuthorizationViewController_Ref/index.html#//apple_ref/occ/cl/PKPaymentAuthorizationViewController) 类的方法 [canMakePaymentsUsingNetworks](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentAuthorizationViewController_Ref/index.html#//apple_ref/occ/clm/PKPaymentAuthorizationViewController/canMakePaymentsUsingNetworks:) 判断用户是否能使用你支持的支付网络完成付款。[canMakePayments](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentAuthorizationViewController_Ref/index.html#//apple_ref/occ/clm/PKPaymentAuthorizationViewController/canMakePayments) 方法可以判断当前设备的硬件是否支持 Apple Pay 以及家长控制是否允许使用 Apple Pay。

如果 canMakePayments 返回 NO，则设备不支持 Apple Pay。不要显示 Apple Pay 按扭，你可以选择使用其它的支付方式。

如果 canMakePayments 返回 YES，但 canMakePayementsUsingNetworks: 返回 NO，则表示设备支持 Apple Pay，但是用户并没有为任何请求的支付网络添加银行卡。你可以选择显示一个支付设置按扭，引导用户添加银行卡。如果用户点击该按扭，则开始设置新的银行卡流程 (例如，通过调用 [openPaymentSetup](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPassLibrary_Ref/index.html#//apple_ref/occ/instm/PKPassLibrary/openPaymentSetup) 方法)。

一旦按下 Apple Pay 按扭，你就开始支付授权过程。在显示支付请求之前不要让用户进行任何其它操作。例如，如果用户需要输入优惠码，你应该在用户按下 Apple Pay 按扭之前要求用户输入该优惠码。

> 注意：
>  > 在 iOS 8.3 以及以后的系统中，你可以选择使用 [PKPayementButton](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentButton_Class/index.html#//apple_ref/occ/cl/PKPaymentButton) 方法在初始化支付请求时创建带商标的 Apple Pay 按扭。对于 iOS 8.2 以及更早的系统，你可以使用[《Apple Pay 标志指南》](https://developer.apple.com/apple-pay/Apple-Pay-Identity-Guidelines.pdf) 中提示的方法。
> > 其它关于使用 Apple Pay 按扭以及支付标志的指南请参考[《iOS 人机界面准则》](https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/MobileHIG/index.html#//apple_ref/doc/uid/TP40006556) 中的 Apple Pay 相关部分。

## 桥接基于 web 的支付接口

如果应用使用的是基于 web 的接口进行商品或服务的支付，那么你在处理 Apple Pay 事务之前你需要将 web 接口的请求发送至 iOS 本地代码。列表 3-1 展示了处理来自 web 视图的付款请求步骤：

列表 3-1 在 web 视图中购买商品或服务。

```
// Called when the web view tries to load "myShoppingApp:buyItem"
-(void)webView:(nonnull WKWebView *)webView
decidePolicyForNavigationAction:(nonnull WKNavigationAction *)navigationAction
decisionHandler:(nonnull void (^)(WKNavigationActionPolicy))decisionHandler {
 
    // Get the URL for the selected link.
    NSURL *URL = navigationAction.request.URL;
 
    // If the scheme and resource specifier match those defined by your app,
    // handle the payment in native iOS code.
    if ([URL.scheme isEqualToString:@"myShoppingApp"] &&
        [URL.resourceSpecifier isEqualToString:@"buyItem"]) {
 
        // Create and present the payment request here.
 
        // The web view ignores the link.
        decisionHandler(WKNavigationActionPolicyCancel);
    }
 
    // Otherwise the web view loads the link.
    decisionHandler(WKNavigationActionPolicyAllow);
}
```

## 包含货币以及地区信息的支付请求

在同一个支付请求中的所有汇总金额使用相同的货币。所使用的币种可以通过  PKPaymentRequest 的 [currencyCode](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentRequest_Ref/index.html#//apple_ref/occ/instp/PKPaymentRequest/currencyCode) 属性指定。币种由三个字符的 ISO 货币代码指定，例如 USD 表示美元。

支付请求中的国家代码表明支付发生的国家或者支付将在哪个国家处理。由三个字符的 ISO 国家代码指定该属性，例如 US。

在请求中指定的商户 ID 必须是应用程序有授权的商户 ID 中的某一个。

```
request.currencyCode = @"USD";
request.countryCode = @"US";
request.merchantIdentifier = @"merchant.com.example";
```

## 支付请求包括一系列的支付汇总项

由 [PKPaymentSummaryItem](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentSummaryItem_Ref/index.html#//apple_ref/occ/cl/PKPaymentSummaryItem) 类表示支付请求中的不同部分。一个支付请求包括多个支付汇总项，一般包括：小计、折扣、配送费用、税以及总计。如果你没有其它任何额外的费用 (例如，配送或税)，那么支付的总额直接是所有购买商品费用 的总和。关于每一项商品的费用的详细信息你需要在应用程序的其它合适位置显示。

如列表 3-2 所示，每一个汇总项都有标签和金额两个部分。标签是对该项的可读描述。金额对应于所需支付的金额。一个支付请求中的所有金额都使用该请求中指定的支付货币类型。对于折扣和优惠券，其金额被设置为负值。

某些场景下，如果在支付授权的时候还不能获取应当支付的费用(例如，出租车收费)，则使用 [PKPaymentSummaryItemTypePending](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentSummaryItem_Ref/index.html#//apple_ref/c/econst/PKPaymentSummaryItemTypePending) 类型做小计项，并将其金额值设置为 0.0。系统随后会设置该项的金额值。

列表 3-2 创建支付汇总项

```
// 12.75 subtotal
NSDecimalNumber *subtotalAmount = [NSDecimalNumber decimalNumberWithMantissa:1275 exponent:-2 isNegative:NO];
self.subtotal = [PKPaymentSummaryItem summaryItemWithLabel:@"Subtotal" amount:subtotalAmount];
 
// 2.00 discount
NSDecimalNumber *discountAmount = [NSDecimalNumber decimalNumberWithMantissa:200 exponent:-2 isNegative:YES];
self.discount = [PKPaymentSummaryItem summaryItemWithLabel:@"Discount" amount:discountAmount];
```

> 注意
> > 汇总项使用 [NSDecimalNumber](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSDecimalNumber_Class/index.html#//apple_ref/occ/cl/NSDecimalNumber) 类存储金额，并且金额使用 10 进制数表示。如示例代码演示的一样，可以通过显示地指定小数部分与指数部分创建该类的实例，也可以直接使用字符串的方式指定金额。在财务计算中绝大部分情况下都是使用的 10 进制数进行计算的，例如，计算 5% 的折扣。

> > 尽管 IEEE 浮点数 float 或 double 计算更加方便一些，但是它们并不适用于财务计算中，因为这些数字使用 2 进制表示。这意味着有些符点数不能被准确的表示，例如 0.42 只能被近似的表示为 0.41999...。这样的近似可能导致财务计算返回错误的结果。

汇总项列表中最后一项是总计项。总计项的金额是其它所有汇总项的金额的和。总计项的显示不同用于其它项。在该项中，你应该使用你的公司名称作为其标签，使用所有其它项的金额之和作为其金额值。最后，使用 [paymentSummaryItems](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentRequest_Ref/index.html#//apple_ref/occ/instp/PKPaymentRequest/paymentSummaryItems) 属性将所有汇总项都添加到支付请求中。

```
// 10.75 grand total
NSDecimalNumber *totalAmount = [NSDecimalNumber zero];
totalAmount = [totalAmount decimalNumberByAdding:subtotalAmount];
totalAmount = [totalAmount decimalNumberByAdding:discountAmount];
self.total = [PKPaymentSummaryItem summaryItemWithLabel:@"My Company Name" amount:totalAmount];
 
self.summaryItems = @[self.subtotal, self.discount, self.total];
request.paymentSummaryItems = self.summaryItems;
```
## 配送方式是一个特殊的支付汇总项

为每一个可选的配送方式创建一个 [PKShippingMethod](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKShippingMethod_Ref/index.html#//apple_ref/occ/cl/PKShippingMethod) 实例。与其它支付汇总项一些，配送方式也有一个用户可读的标签，例如标准配送或者可隔天配送，和一个配送金额值。与其它汇总项不同的时，配送方法有一个 [detail](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKShippingMethod_Ref/index.html#//apple_ref/occ/instp/PKShippingMethod/detail) 属性值，例如，7 月 29 日送达或者 24 小时之内送达等等。该属性值说明不同配送方式之间的区别。

为了在委托方法中区分不同的配送方式，你可以使用 [identifier](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKShippingMethod_Ref/index.html#//apple_ref/occ/instp/PKShippingMethod/identifier) 属性。这个属性只被该应用使用，它对于支付框架是不可见。同样，它也不会出现在 UI 中。在创建每个配送方式的时候为其分配一个唯一的标识符。为了便于调试，推荐使用简短字符串或者字符串缩写，例如 “discount”、“standard”、“next-day” 等等。

有些配送方式并不是在所有地区都是可以使用的，或者它们费用会根据配送地址的不同而发生变化。你需要在用户选择配送地址或方法时更新其信息，详情请见 [委托方法更新配送方法与费用](https://developer.apple.com/library/ios/ApplePay_Guide/Authorization.html#//apple_ref/doc/uid/TP40014764-CH4-SW2)。

## 指定应用程序支持的支付处理机制

[supportedNetworks](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentRequest_Ref/index.html#//apple_ref/occ/instp/PKPaymentRequest/supportedNetworks) 属性是一个字符串常量，通过设置该值可以指定应用所支持的支付网络。 [merchantCapabilities](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentRequest_Ref/index.html#//apple_ref/occ/instp/PKPaymentRequest/merchantCapabilities) 属性值说明应用程序支持的支付处理协议。3DS 协议是须支持的支付处理协议， EMV 是可选的支付处理协议。

```
request.supportedNetworks = @[PKPaymentNetworkAmex, PKPaymentNetworkDiscover, PKPaymentNetworkMasterCard, PKPaymentNetworkVisa];
 
// Supports 3DS only
request.merchantCapabilities = PKMerchantCapability3DS;
 
// Supports both 3DS and EMV
request.merchantCapabilities = PKMerchantCapability3DS | PKMerchantCapabilityEMV;
```

## 说明所需的配送信息和账单信息

修改支付授权视图控制器的 [requiredBillingAddressFields](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentRequest_Ref/index.html#//apple_ref/occ/instp/PKPaymentRequest/requiredBillingAddressFields) 属性和 [requiredShippingAddressFields](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentRequest_Ref/index.html#//apple_ref/occ/instp/PKPaymentRequest/requiredShippingAddressFields) 属性可以设置所需的账单信息和配送信息。当你显示视图控制器时，它会提示用户输入必需的账单信息和配送信息。这个域的值是通过这些属性组合而成的，如下所示：

```
request.requiredBillingAddressFields = PKAddressFieldEmail;
request.requiredBillingAddressFields = PKAddressFieldEmail | PKAddressFieldPostalAddress;
```

> 注意：
> > 请求只包括处理支付和配送商品或服务的必需的账单信息和配送信息。额外的非必需信息都会增加处理事务的复杂度。每个多余的步骤都可能会增加用户取消支付的风险。

如果已有最新账单信息以及配送联系信息，你可以直接为支付请求设置这些值。 Apple Pay 会默认使用这些信息。但是，用户仍然可以选择在本次支付中使用其它联系信息。

```
PKContact *contact = [[PKContact alloc] init];
 
NSPersonNameComponents *name = [[NSPersonNameComponents alloc] init];
name.givenName = @"John";
name.familyName = @"Appleseed";
 
contact.name = name;
 
CNMutablePostalAddress *address = [[CNMutablePostalAddress alloc] init];
address.street = @"1234 Laurel Street";
address.city = @"Atlanta";
address.state = @"GA";
address.postalCode = @"30303";
 
contact.postalAddress = address;
 
request.shippingContact = contact;
```

> 请注意：
> > 地址信息可以从 iOS 中很多地方获取到，请在使用它之前确保该信息是有效的。

## 保存其它信息

保存支付中其它与应用相关的信息，例如购物车标识，你可以使用 [applicationData](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentRequest_Ref/index.html#//apple_ref/occ/instp/PKPaymentRequest/applicationData) 属性。这个属性对于系统来说是不可见的。用户授权支付后，应用数据的哈希值也会成为支付令牌的一部分。