# 授权支付

支付授权过程是由支付授权视图控制器与其委托合作完成的。支付授权视图控制器做了两件事：

 1. 让用户选择支付请求所需的账单信息与配送信息。
 2. 让用户授权支付操作。

用户与视图控制器交互时，委托方法会被系统调用，所以在这些方法中你的应用可以更新所要显示的信息。例如在配送地址修改后更新配送价格。在用户授权支付请求后此方法还会被调用一次。

> 注意：
> > 在实现这些委托方法时，你应该谨记它们会被多次调用并且这些方法调用的顺序是取决与用户的操作顺序的。

所有的这些委托方法在授权过程中都会被调用，传入该方法的其中一个参数是一个完成块 (completion block)。支付授权视图控制器等待一个委托完成相应的方法后 (通过调用完成块) 再依次调用其它的委托方法。[paymentAuthorizationViewControllerDidFinish: ](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentAuthorizationViewControllerDelegate_Ref/index.html#//apple_ref/occ/intfm/PKPaymentAuthorizationViewControllerDelegate/paymentAuthorizationViewControllerDidFinish:) 方法是唯一例外：它并不需要一个完成块作为参数，它可以在任何时候被调用。

完成块接受一个输入参数，该参数为应用程序根据信息判断得到的支付事务的当前状态。如果支付事务一切正常，则应传入值 PKPaymentAuthorizationStatusSuccess。否则，可以传入能识别出错误的值。

创建 [PKPaymentAuthorizationViewController](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentAuthorizationViewController_Ref/index.html#//apple_ref/occ/cl/PKPaymentAuthorizationViewController) 类的实例时，你需要将已初始化后的支付请求传递给视图控制器初始化函数。接着，设置视图控制器的委托；然后再显示它：

```
PKPaymentAuthorizationViewController *viewController = [[PKPaymentAuthorizationViewController alloc] initWithPaymentRequest:request];
if (!viewController) { /* ... Handle error ... */ }
viewController.delegate = self;
[self presentViewController:viewController animated:YES completion:nil];
```

当用户与视图控制器交互时，视图控制器就会调用其委托方法：

> 注意：
> > 在 Xcode 7.0 及其后的版本中，你可以在模拟器中测试支付授权视图控制器。这些版本的模拟器提供了支持所有支付网络的虚拟卡，它会以纯文本的方式返回虚拟支付数据。在设备上时，这些数据会使用商户 ID 进行加密。

> > 虽然模拟器可以方便快捷地测试支付代码，但是你仍然需要在物理设备上测试你的支付功能。

> > 如果你使用的是较早版本的 Xcode，那么你就只能在物理设备上测试你的支付功能了。

## 使用委托方法更新配送方式与配送费用

当用户输入配送信息时，授权视图控制器会调用委托的 [paymentAuthorizationViewController:didSelectShippingContact:completion:](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentAuthorizationViewControllerDelegate_Ref/index.html#//apple_ref/occ/intfm/PKPaymentAuthorizationViewControllerDelegate/paymentAuthorizationViewController:didSelectShippingContact:completion:) 方法和 [paymentAuthorizationViewController:didSelectShippingMethod:completion:](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentAuthorizationViewControllerDelegate_Ref/index.html#//apple_ref/occ/intfm/PKPaymentAuthorizationViewControllerDelegate/paymentAuthorizationViewController:didSelectShippingMethod:completion:) 方法。你可以实现这两个方法来更新你的支付请求。

```
- (void) paymentAuthorizationViewController:(PKPaymentAuthorizationViewController *)controller
                   didSelectShippingContact:(CNContact *)contact
                                 completion:(void (^)(PKPaymentAuthorizationStatus, NSArray *, NSArray *))completion
{
    self.selectedContact = contact;
    [self updateShippingCost];
    NSArray *shippingMethods = [self shippingMethodsForContact:contact];
    completion(PKPaymentAuthorizationStatusSuccess, shippingMethods, self.summaryItems);
}
 
- (void) paymentAuthorizationViewController:(PKPaymentAuthorizationViewController *)controller
                    didSelectShippingMethod:(PKShippingMethod *)shippingMethod
                                 completion:(void (^)(PKPaymentAuthorizationStatus, NSArray *))completion
{
    self.selectedShippingMethod = shippingMethod;
    [self updateShippingCost];
    completion(PKPaymentAuthorizationStatusSuccess, self.summaryItems);
}
```

> 注意：
> > 为了保护用户隐私，提供给方法 [paymentAuthorizationViewController:didSelectShippingContact:completion:](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentAuthorizationViewControllerDelegate_Ref/index.html#//apple_ref/occ/intfm/PKPaymentAuthorizationViewControllerDelegate/paymentAuthorizationViewController:didSelectShippingContact:completion:) 的配送信息是经过匿名化处理后的数据。返回的 contact 包含了计算配送费用的所有信息同时隐藏了用户的敏感信息。只有在用户授权支付后，你才能得到用户完整的配送信息。此外， contact 中的数据会随着国家的不同而不同，同时还会随着版本的更新而变化。请仔细测试你的应用程序。

## 支付被授权时创建了一个支付令牌

当用户授权一个支付请求时，支付框架的 Apple 服务器与安全模块会协作创建一个支付令牌。你可以在委托方法 [paymentAuthorizationViewController:didAuthorizePayment:completion:](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentAuthorizationViewControllerDelegate_Ref/index.html#//apple_ref/occ/intfm/PKPaymentAuthorizationViewControllerDelegate/paymentAuthorizationViewController:didAuthorizePayment:completion:) 中将支付信息以及其它你需要处理的信息，例如配送地址和购物车标识符，一起发送至你的服务器。这个过程如下所示：

- 支付框架将支付请求发送至安全模块。只有安全模块会访问令牌化后的设备相关的支付卡号。
- 安全模块将特定卡的支付数据和商家信息一起加密(加密后的数据只有 Apple 可以访问)，然后将加密后的数据发送至支付框架。支付框架再将这些数据发送至 Apple 的服务器。
- Apple 服务器使用商家标识证书将这些支付数据重新加密。这些令牌只能由你以及那些与你共享商户标识证书的人读取。随后服务器生成支付令牌再将其发送至设备。
- 支付框架调用 [paymentAuthorizationViewController:didAuthorizePayment:completion:](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentAuthorizationViewControllerDelegate_Ref/index.html#//apple_ref/occ/intfm/PKPaymentAuthorizationViewControllerDelegate/paymentAuthorizationViewController:didAuthorizePayment:completion:)  方法将令牌发送至你的委托。你在委托方法中再将其发送至你的服务器。

在服务器上的处理操作取决于你是自己处理支付还是使用其它支付平台。不过，在两种情况下服务器都得处理订单再将处理结果返回给设备。在设备上，委托再将处理结果传入完成处理方法中，详细过程请参阅 [处理支付](.\processing-payments.md)

```
- (void) paymentAuthorizationViewController:(PKPaymentAuthorizationViewController *)controller
                        didAuthorizePayment:(PKPayment *)payment
                                 completion:(void (^)(PKPaymentAuthorizationStatus))completion
{
    NSError *error;
    ABMultiValueRef addressMultiValue = ABRecordCopyValue(payment.billingAddress, kABPersonAddressProperty);
    NSDictionary *addressDictionary = (__bridge_transfer NSDictionary *) ABMultiValueCopyValueAtIndex(addressMultiValue, 0);
    NSData *json = [NSJSONSerialization dataWithJSONObject:addressDictionary options:NSJSONWritingPrettyPrinted error: &error];
 
    // ... Send payment token, shipping and billing address, and order information to your server ...
 
    PKPaymentAuthorizationStatus status;  // From your server
    completion(status);
}
```

## 委托方法中释放支付授权视图控制器

支付框架显示完支付事务状态后，授权视图控制器会调用委托的 [aymentAuthorizationViewControllerDidFinish:](https://developer.apple.com/library/ios/documentation/PassKit/Reference/PKPaymentAuthorizationViewControllerDelegate_Ref/index.html#//apple_ref/occ/intfm/PKPaymentAuthorizationViewControllerDelegate/paymentAuthorizationViewControllerDidFinish:) 方法。在此方法的实现中，你应该释放授权视图控制器然后再显示与应用相关的支付信息界面。

```
- (void) paymentAuthorizationViewControllerDidFinish:(PKPaymentAuthorizationViewController *)controller
{
    [controller dismissViewControllerAnimated:YES completion:nil];
}
```