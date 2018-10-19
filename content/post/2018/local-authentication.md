---
title: "Touch ID and Face ID on iOS"
date: 2018-05-05
tags:
- swift
- dev
---

## Introduction

Adding support for Touch ID and Face ID to your app is not always completely straightforward, especially given that the documentation from Apple on the APIs is somewhat sparse and in some cases incorrect. I recently added Face ID and Touch ID support to my company's app and I thought it would be helpful to document what I found.

There is only a single class, [LAContext](https://developer.apple.com/documentation/localauthentication/lacontext) in the [LocalAuthentication](https://developer.apple.com/documentation/localauthentication/) framework, so it's surprising how complex it can get.

The most naive implementation of a biometric (Touch ID or Face ID) authentication request is as in Apple's example:

<script src="https://gist.github.com/mluisbrown/2979bb69e81d9ba4eba3f7c15abc9fa4.js"></script>

## What is actually supported?

However, that will almost certainly be insufficient for a production app. For a start, before you even get as far as attempting a local authentication, you will probably want to know which form (Touch ID or Face ID, or neither) is available, so that you can change your UI appropriately. Apple [recommends](https://developer.apple.com/ios/human-interface-guidelines/user-interaction/authentication/) that you *"Don't reference Touch ID on a device that supports Face ID [and] don't reference Face ID on a device that supports Touch ID"*.

You can use the [`biometryType`](https://developer.apple.com/documentation/localauthentication/lacontext/2867583-biometrytype) property of `LAContext` which will return an `LABiometryType` of either `none`, `faceID`, or `touchID`. Be aware that the `none` case was only added in iOS 11.2. Since you will already be doing `if #available` checks for iOS 10 (Touch ID only) and iOS 11, adding another check for iOS 11.2 just to see if you can use `none` is a real pain. I opted to just use `default` in my switch on the biometry type to handle `none`. In the documentation for `biometryType` there is an important error that Apple [knows about](http://www.openradar.me/36064151) but has yet to fix. It states:

> This property is only set when canEvaluatePolicy(_:error:) succeeds for a biometric policy. The default value is LABiometryNone.

That is incorrect. Once you call `canEvaluatePolicy` on a context, the `biometryType` **is** set, regardless if the call failed or succeeded. That's important to know, because if what the docs state were true, if the call fails it would be impossible to tell if it failed because the device doesn't support Touch ID or Face ID, or because they are disabled, not enrolled or locked out. You might have UI (eg in a settings pane) which references Touch ID or Face ID even if they are currently disabled, and you want to know *which* of them it should reference.

I ended up creating my own abstraction to make it easier to track the full state of biometry support on the device and to limit all OS version checks to a single place:

<script src="https://gist.github.com/mluisbrown/70ef39d99f9801e3e532686f006053d2.js"></script>

I am using `notAvailable` to cover both the "not available" and "not enrolled" states that the API can return. Incidentally, if the `LAError.Code` returned from the API is `biometryNotAvailable` that includes the situation where Face ID has been disabled by the user for your app in Settings.app.

I created a `BiometricAuth` class to deal with everything related to local authentication. Below is the implementation of the `supportedBiometry` calculated property which returns a `BiometrySupport` enum:

<script src="https://gist.github.com/mluisbrown/5503dd94fa71a60a55e33f3cf90b1dce.js"></script>

As you can see, between OS version checks and checking both success and error conditions, just getting all the information about the supported biometry on the device is quite a bit more complex than a single API call!

## Authenticating

Ok, so you know what's supported, now how do you actually authenticate? First, just so it's clear, you should only attempt biometric authentication if you know it's available. You don't need to call `evaluatePolicy` using the same `LAContext` on which `canEvaluatePolicy` was called (as in the Apple example), but you should call it fairly soon after to minimize that chance that circumstances have changed.

To invoke the biometric authentication, create an `LAContext` and call [`evaluatePolicy(_:localizedReason:reply:)`](https://developer.apple.com/documentation/localauthentication/lacontext/1514176-evaluatepolicy). The `localizedReason` parameter is displayed in the Touch ID alert subtitle to tell the user why you're requesting authentication. It's not used for Face ID as far as I can tell, as Face ID does not display an alert, it just "happens". If you want to provide the user with a fallback method of authentication (eg a password or a PIN code), you should set the `localizedFallbackTitle` on the `LAContext` before calling `evaulatePolicy`. If you don't set it (ie it remains `nil`), the default is "Enter password", which may not be appropriate for your app. In another bit of questionable (and undocumented) API design by Apple, if you don't want a fallback option to appear at all, you have to set `localizedFallbackTitle` to `""` 🙈. When testing this, be aware that the fallback option only appears once Touch ID or Face ID has failed at least once. If the user taps the fallback option, the `LAError.Code` returned in the `Error` of the `reply` handler will be `LAError.userFallback`.

The call to `evaluatePolicy` is obviously asynchronous, so you have to handle the response in the `reply` closure. The thread on which this closure is called is not the main thread, so make sure that you `DispatchQueue.main.async` the outcome to update your UI as appropriate (if you're using a [reactive framework](https://github.com/ReactiveCocoa/ReactiveSwift) this is so much easier 😛).

## What are you authenticating?

When integrating Touch ID and Face ID, you need to think about what it is that you are actually protecting and how, because if you were dependent on the user's password for, say, making an API call to a server to get an auth token, then you won't have that password if you're using biometric authentication. There are a couple of options:

* You could store the password (well, a hash of it) in the keychain and use that.
* For APIs, you could store a persistent auth token in the keychain, and use that. This requires that the user authenticates using their password when you're enabling biometric authentication in your app, so that you have a token to store.

For many apps this may be sufficiently secure. However, with Jailbreak, it's trivial to read the contents of the keychain and / or patch your app to bypass the `evaluatePolicy` calls and act as if they had always succeeded. A successful Jailbreak, whilst maintaining the user's data, requires the user's device passcode. Also, if an attacker has the device passcode, they can register their own finger for Touch ID or reset Face ID with their own face, and then access your app that way. For applications with sensitive data such as financial or health, there are ways to prevent both the Jailbreak and the "re-register" attacks.

## Restricting keychain item access to Touch ID and Face ID

When storing keys or tokens in the keychain, you can restrict them to be only accessible using Touch ID or Face ID by creating a [`SecAccessControl`](https://developer.apple.com/documentation/security/secaccesscontrol) with the `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` protection and the `touchIDCurrentSet` or `touchIDAny` flag (`touchIDCurrentSet` restricts access to the currently registered Touch ID fingerprints or the currently registered Face ID face):

NOTE: `touchIDCurrentSet` and `touchIDAny` have been deprecated in iOS 11.3 in favour of `biometryCurrentSet` and `biometryAny`.

```
// create an Access Control
func createAccessControl() -> SecAccessControl? {
    var accessControlError: Unmanaged<CFError>?
    guard let accessControl = SecAccessControlCreateWithFlags(
        kCFAllocatorDefault,
        kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
        [.touchIDCurrentSet], &accessControlError) else {
        // couldn't create accessControl
        return nil
    }

    return accessControl
}
```

And then passing that access control as the value for the `kSecAttrAccessControl` key when calling `SecItemAdd`:

```
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrService as String: "my service name",
    kSecUseAuthenticationUI as String: kSecUseAuthenticationUIFail,
    kSecValueData as String: "my secret data",
    kSecAttrAccessControl as String: accessControl,
    ]

SecItemAdd(query as CFDictionary, nil)
```

Incidentally, if you're storing data in the keychain with Touch ID / Face ID access control restrictions, you don't actually *need* to call `LAContext.evaluatePolicy` at all. Just attempting to access them using `SecItemCopyMatching` will automatically create an `LAContext` and invoke the biometric authentication process for you (synchronously). However, I wouldn't recommend that, as it not only gives you less control but it make the thread calling `SecItemCopyMatching` block during authentication. What I recommend is:

1. Create an `LAContext`
2. Call `evaluatePolicy` on it
3. If successful, pass the context as the value for the `kSecUseAuthenticationContext` key in the query dictionary passed to `SecItemCopyMatching`.

## Restricting to the currently registered biometrics

`LAContext` has a property [`evaluatedPolicyDomainState`](https://developer.apple.com/documentation/localauthentication/lacontext/1514150-evaluatedpolicydomainstate), which is an opaque `Data?` object that is set whenever `canEvaluatePolicy` or `evaluatePolicy` succeeds, otherwise it is `nil`. If any new fingerprints are added to Touch ID, or Face ID is reset and a new (or even the same) face is added, then the `evaluatedPolicyDomainState` data will change.

So, in order to detect such a change, you need to store this value after a successful call to `evaluatePolicy`. You can store it in `UserDefaults`, there is no need to store it securely. After that, when you call `canEvaluatePolicy`, compare the value in the context against the value you stored previously (you can just use `==` comparison of the `Data` objects). If they are different, it means the registered biometric data has changed, and you can then force the user to provide their password again before allowing Touch ID or Face ID to be used again.

The `touchIDCurrentSet` `SecAccessControl` flag as shown above also restricts access to keychain items to the biometrics registered at the time the item was added to the keychain.

This will solve the situation where the attacker has the user's device passcode, but it will not protect against Jailbreak, where all these checks can be bypassed, and where the keychain can be decrypted. But that can also be solved.

## Using the Secure Enclave for the highest security

Even restricting keychain items to biometric access control, and restricting biometric authentication to the currently registered biometric data set won't help you if a device is jailbroken, because on a jailbroken device the entire contents of the keychain can be dumped out in plain text, regardless of access restrictions. The only really secure storage on an iOS device is the Secure Enclave, and only devices with Touch ID or Face ID have it. It is where the biometric data is stored. The good news is that as of iOS 9 Apple introduced an API for apps to store "data" in the Secure Enclave. However, since allowing user apps the ability to read anything from the Secure Enclave would make it also susceptible to jailbreak attacks, the caveat is that anything you store in the Secure Enclave can never be read back! Fortunateley, that's not as utterly useless as it might appear.

In fact, the **only** data that user apps can store in the Secure Enclave is the private key of a 256-bit elliptic curve public / private key pair. The key is generated in the Secure Enclave itself, and it never leaves there. All you can access is the public key and a reference to the private key, which can only be obtained after a successful biometric authentication.

### How can this be used?

When the user chooses to use biometric auth, create a public / private key pair for storing in the Secure Enclave, and store the public key in the keychain for later use. Then the public key can be used for either:

#### Encryption

User data can be encrypted using the public key, secure in the knowledge that it can only be decrypted by the Secure Enclave which contains the matching private key, and which can only be accessed after a successful biometric authentication.

#### Signing

The public key can be sent to a server and associated with a user and device. Subsequently, in order to get a session token from the server:

* the app makes a request to the server for a string of random data (a [nonce](https://en.wikipedia.org/wiki/Cryptographic_nonce)).

* after a successful biometric authentication, the app *signs* the nonce using the private key in the Secure Enclave, and sends the signed nonce back to the server.

* the server can then use the public key which was registered for the user to *verify* the signature of the nonce, indicating that the user has been authenticated on the device, and so then provides a session token. This would substitute the normal username plus password hash being sent for server authentication.

Code examples for the various steps above are provided below. These are just examples without any error handling.

I highly recommend looking at the [SecureEnclaveCrypto](https://github.com/trailofbits/SecureEnclaveCrypto) repo in GitHub from where I got most of this code, and on which I based my implementation. The repo hasn't been updated for iOS 11 or Swift 4.x unfortunately.
The [EllipticCurveKeyPair](https://github.com/agens-no/EllipticCurveKeyPair) repo is also very useful to look at, and is where I got the algorithm for converting a public key into DER and PEM formats.

## Code for Secure Enclave keys, signing and encryption

### Create a key pair and store the public key

<script src="https://gist.github.com/mluisbrown/3e0611e5ddd9ed7875c183ca7b716a6b.js"></script>

* `kSecAttrKeyTypeECSECPrimeRandom` is the Elliptic Curve key type used by the Secure Enclave. It is equivalent to the `prime256v1` key type in OpenSSL.
* `kSecAttrTokenIDSecureEnclave` indicates the key is stored in the Secure Enclave.

### Obtain the public key

<script src="https://gist.github.com/mluisbrown/62ba0141e1e3e9f99182e7b85f2fd3bd.js"></script>

To get the `SecKey` reference from the `CFDictionary` of the public key above:

```
let converted = publicKey as! [String: Any]
let keyRef = converted[kSecValueRef as String] as! SecKey
```

To get the actual key data from the public key:
```
let converted = publicKey as! [String: Any]
let data = converted[kSecValueData as String] as! Data
```

If you're sending this key data to a server, you'll almost certainly want it in the DER or PEM formats that OpenSSL understands:

<script src="https://gist.github.com/mluisbrown/f3cdb3d0e620679d155caba2d266c38e.js"></script>

### Obtain a private key reference

You will need to provide an `LAContext` on which `evaluatePolicy` has succeeded. If you don't, one will be created for you and used. The `kSecUseOperationPrompt` is not used if you pass a context where you have already authenticated.

<script src="https://gist.github.com/mluisbrown/06063646c27a7ac5c4222d4a7dea940b.js"></script>

### Sign some data

<script src="https://gist.github.com/mluisbrown/7da7c905e1835b3da0ad5f868d98a5ae.js"></script>

### Encrypt some data

<script src="https://gist.github.com/mluisbrown/c489d31654194b3adccc655e998443af.js"></script>

### Decrypt some data

<script src="https://gist.github.com/mluisbrown/dacc6c96103baa640ad1fc01a3c53801.js"></script>

## Conclusion

I hope I've been able to shed some light on an area of iOS development that isn't particularly well documented, and that has a lot more nuances and edge cases than might appear at first sight.
