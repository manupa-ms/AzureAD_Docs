# UPN methods deprecation in Intune MAM SDK
As many of you may have noticed in our javadocs, certain MAM SDK API methods that take UPNs to specify identities are deprecated and will be removed completely at the next major version increment as UPN is not a reliable parameter to uniquely identify a user account - it can change overtime; two different accounts could share the same UPN and so on. In order to avoid inconsistencies relating to UPNs, this change has been introduced. Most of the SDK's identity APIs use the provided OID as the identifier for the identity. The MAM SDK APIs will return the OID string as the identity and will require the OID string parameter for the identity. 

New methods that specify identities by OID (also known as AAD User ID, AAD ID or Entra ID) should be used instead. Apps are strongly encouraged to examine their existing usage of UPN-based APIs and migrate to the corresponding OID-based APIs.

#### References:
* Android: https://github.com/microsoftconnect/ms-intune-app-sdk-android/discussions/238
* iOS: https://github.com/microsoftconnect/ms-intune-app-sdk-ios/discussions/460


However we understand there may be scenarios requiring fetching the UPN value for an account rather than the OID. Keeping this in mind, we have added below some guidance on how developers can fetch UPN value on both iOS and Android platforms.

## For iOS:
1. In most cases, you can acquire the UPN value of the account using the [IntuneMAMEnrollmentDelegate](https://learn.microsoft.com/en-us/mem/intune/developer/app-sdk-ios-phase3#status-result-and-debug-notifications) which returns an IntuneMAMEnrollmentStatus object that contains the UPN.

```
- (void)enrollmentRequestWithStatus:(IntuneMAMEnrollmentStatus*)status
{
    NSLog(@"Enrollment result for identity %@ with status code %ld", status.identity, (unsigned long)status.statusCode);
    NSLog(@"Debug Message: %@", status.errorString);
}
```

2. An alternative to the above approach, if yours is a scenario requiring fetching UPN value from the PCA object, you could leverage the following script using MSAL's tenantProfileIdentifier as shown below -

```
import MSAL

let config = MSALPublicClientApplicationConfig(clientId: clientId, redirectUri: redirectUri, authority: authority)
let msalApplication = try? MSALPublicClientApplication(configuration: config)
let parameters = MSALAccountEnumerationParameters(tenantProfileIdentifier: oid)
var error: NSError? = nil
let accounts = msalApplication?.accounts(for: parameters, error: &error)
var upn = ""

if let accounts = accounts {
    for account in accounts {
        upn = account.username
    }
}
```

Ref: https://learn.microsoft.com/en-us/dotnet/api/microsoft.identity.client.tenantprofile?view=msal-dotnet-latest

> **NOTE:** Another commonly known method in the SDK is enrolledAccount(), used to fetch the user's UPN. This has also already been removed in version 20.x for Xcode 16. As an alternative to this, app developers should leverage corresponding MSAL APIs to map the OID to a UPN, if that's what is needed. The API that could help achieve this is the [accountForIdentifier](https://github.com/AzureAD/microsoft-authentication-library-for-objc/blob/8ed4d9d46f1bc86ef2afb32aad86f935663f325d/MSAL/src/public/MSALPublicClientApplication.h#L204) on the MSALPublicClientApplication class. This will return a [MSALAccount](https://github.com/AzureAD/microsoft-authentication-library-for-objc/blob/8ed4d9d46f1bc86ef2afb32aad86f935663f325d/MSAL/src/public/MSALAccount.h#L46) object on which you can query the username property.

## For Android:
In case of Android platform, if you want to retrieve an account from the OID and get the username property to find the UPN value, you could call [getAccount()](https://javadoc.io/doc/com.microsoft.identity.client/msal/2.2.3/index.html#getAccount-java.lang.String-).

```
PublicClientApplication.createMultipleAccountPublicClientApplication(
    ApplicationManager.self(),
    R.raw.auth_config_single_account_debug,
        new IPublicClientApplication.IMultipleAccountApplicationCreatedListener() {

        @Override
            public void onCreated(IMultipleAccountPublicClientApplication application) {
                Thread workerThread = new Thread(() -> {
                    try {
                        List<IAccount> list = application.getAccounts();
                        Log.d("test", "list.get(0).getUsername(): " + list.get(0).getUsername());
                        Log.d("test","list.get(0).getId(): " + list.get(0).getId());
                    }
                }
        )}
    }
)
```

Ref: https://javadoc.io/static/com.microsoft.identity.client/msal/2.2.3/com/microsoft/identity/client/IMultipleAccountPublicClientApplication.html#getAccount-java.lang.String-

### Conclusion
In conclusion, we would reiterate that developers should stop relying on UPNs for account identifiers are they are not suitable unique IDs. The MAM SDK has has switched to using the OID of the account as the unique identifier internally, so we can ensure that the correct MAM policies are used for a given account.

### Review Credit
