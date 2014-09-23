title: 在iOS下访问通讯录
date: 2014-09-23 23:16:59
tags: iOS
---

现在写应该晚了很多了，毕竟iOS 6已经出来大半年了，不过我知道昨天才搞明白iOS 6下通讯录权限正确的获取方法，就在这做个记录吧。

啥也不说，先上代码。

```objc 
- (void)initAddressBook
{
    _addressBook = NULL;
    __block BOOL accessGranted = NO;

    dispatch_semaphore_t sema = dispatch_semaphore_create(0);
    if (ABAddressBookRequestAccessWithCompletion != NULL) { // we're on iOS 6
        _addressBook = ABAddressBookCreateWithOptions(NULL, nil);
        ABAddressBookRequestAccessWithCompletion(_addressBook, ^(bool granted, CFErrorRef error) {
        accessGranted = granted;
        dispatch_semaphore_signal(sema);
    });
    dispatch_semaphore_wait(sema, DISPATCH_TIME_FOREVER);
    dispatch_release(sema);
    if (!accessGranted)
        _addressBook = nil;
    } else { // we're on iOS 5 or older
        _addressBook = ABAddressBookCreate();
    }
} 
```

<!-- more -->

首先判断当前运行平台是 6.0 还是 5.0，判断标准就是是否存在 ABAddressBookRequestAccessWithCompletion 这个函数（当然用别的 iOS 6 上新加的函数作为判断依据也可以），如果不存在这个函数，说明在不是 iOS 6，不需要请求权限，直接走 ABAddressBookCreate() 结束；如果是在 iOS 6 下，就需要获得用户授权方可访问通讯录，要不然就悲剧了，轻者出现逻辑bug，重者崩溃。

获得通讯录授权的步骤如下：

使用`ABAddressBookCreateWithOptions(NULL, nil)`返回一个通讯录指针`_addressBook`，注意，这时只是获得一个暂时的指针，由于没有获得授权，还不能使用。
再调用`ABAddressBookRequestAccessWithCompletion(_addressBook, ^(bool granted, CFErrorRef error) { ... })`，向用户请求通讯录授权。如果是第一次运行，到这句话时，iPhone会弹出一个对话框，"xxx想访问你的日历"，对就是日历，估计苹果认为在中国日历和通讯录是一个东西（不得不吐槽一下苹果这个翻译上的低级bug了）。如果用户点击了“好”，`_addressBook`就生效了，以后你想怎么搞就怎么搞了；但是，如果用户点击了“不允许”，`_addressBook`就不能接着使用了，接着使用的下场就是一个字儿：跪了！

那么`dispatch_semaphore_t sema`这个东西是干嘛的呢？先看看苹果官方开发文档上对`ABAddressBookRequestAccessWithCompletion`函数的解释是。
> Users are able to grant or deny access to contact data on a per-app basis. To request access to contact data, call the ABAddressBookRequestAccessWithCompletion function after calling the ABAddressBookCreateWithOptions function. The ABAddressBookRequestAccessWithCompletion function does not block the app while the user is being asked to grant or deny access. Until access has been granted, the ABAddressBookRef object will not contain any contacts, and any attempt to modify contacts fails with a kABOperationNotPermittedByUserError error. The user is prompted only the first time access is requested; any subsequent calls to ABAddressBookCreateWithOptions will use the existing permissions. The completion handler is called on an arbitrary queue. If the ABAddressBookRef object is used throughout the app, then all usage must be dispatched to the same queue to use ABAddressBookRef in a thread-safe manner.
> （用户可以同意或拒绝应用访问联系人数据。为了获取联系人数据访问权限，要在 `ABAddressBookCreateWithOptions`后调用`ABAddressBookRequestAccessWithCompletion`，__当用户被询问是否同意授权时，应用不会被挂起__。直到用户同意授权时，系统不会返回一个包含联系人数据的`ABAddressBookRef`对象，此时，一切修改联系人的尝试都会引起`kABOperationNotPermittedByUserError`错误。__只有第一次请求授权时，用户才会看到弹出的对话框__；之后的请求会根据已经存在的权限处理。Completion句柄是在其他线程队列里被调用的，如果整个应用中都在使用`ABAddressBookRef`对象，那么一定要保证线程安全。）

这里有两处加粗的地方需要注意，咱们从前往后说：

* 用户被询问时，应用不会被挂起。也就是说，在`ABAddressBookRequestWithCompletion`函数执行后，会同时发生两件事儿：第一件、iPhone弹出对话框，询问用户是否同意；第二件、进入到`if (!accessGranted)`这个判断语句中。一个前提是用户的手肯定没那么快，不可能在这条判断进行时就已经点了"好"。也就是说，此时`accessGranted`一定是NO，那么无论你怎么挣扎，必然会有`_addressBook = nil`，好，永远都无法访问通讯录了。为了防止这样的悲剧发生，就需要加入一个信号量来同步这两件事儿。创建一个信号量`sema`，并把它的初始值设为0，一方面主线程等待这个`sema`变为正值之后，再执行对`accessGranted`的判断；另一方面，在`ABAddressBookRequestWithCompletion`的回调函数中，对信号量`sema`执行v操作`(dispatch_semaphore_signal)`，让它变为正值，这样就实现了两个线程的同步，保证在用户同意授权与否后判断用户的授权结果`accessGranted`。
* 只有在第一次请求授权时才会弹出询问对话框，系统会把这次的授权情况记住，在以后的授权请求时，使用相同的授权方式。那么如果用户第一次点击了“不允许”，我们怎么提醒它开启授权呢？就用这个函数吧，`ABAddressBookGetAuthorizationStatus()`，它会返回一个枚举类型`ABAuthorizationStatus`，这个枚举类型有四个值，`kABAuthorizationStatusNotDetermined`（未确定，还未询问授权过时的值）/`kABAuthorizationStatusRestricted`（受限制，一般不会出现，应该是在系统出现了bug的情况下的值）/`kABAuthorizationStatusDenied`（拒绝，用户第一次点了“不允许”）/`kABAuthorizationStatusAuthorized`（同意，用户第一次点了“好”）。这样就可以提出一个改进方案了，在请求授权之前，先判断一下当前的授权情况：如果是未确定，就用`ABAddressBookRequestAccessWithCompletion`请求授权；如果是允许，那就直接读通讯录；如果是拒绝，就弹出对话框提醒用户可以在iPhone的设置-&gt;隐私-&gt;通讯录中开启授权。

另外，再说两个小技巧。通讯录授权这种事儿在调试的时候比较困难，因为你一次询问了，系统就会记住，即使卸了重装，它也会把之前的的授权情况完整的记录下来。为了绕开这个问题，可以在每次调试的之前把应用的Bundle Identifier改一下，只要每次的Bundle Identifier不一样，iPhone就会认为这是一个新的应用，再次弹出授权提示了。授权提示的文案也是可以自定义的，只要在应用的.plist文件中加一个键“Privacy - Contacts Usage Description”，把这个键的值设成自己想要的就OK了。
