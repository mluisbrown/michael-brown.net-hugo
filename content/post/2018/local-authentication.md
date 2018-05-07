---
title: "Touch ID and Face ID on iOS"
date: 2018-05-05
tags: 
- swift
- dev
---

Adding support for Touch ID and Face ID to your app is not always completely straightforward, especially given that the documentation from Apple on the APIs is somewhat sparse and in some cases incorrect. I recently added Face ID and Touch ID support to my company's app and I thought it would be helpful to document what I found.

There is only a single class, [LAContext](https://developer.apple.com/documentation/localauthentication/lacontext) in the a [LocalAuthentication](https://developer.apple.com/documentation/localauthentication/) framework, so it's surprising how complex it can get.

The most naive implementation of a biometric (Touch ID or Face ID) authentication request is as in Apple's example:

```swift
let myContext = LAContext()
let myLocalizedReasonString = <#String explaining why app needs authentication#>
 
var authError: NSError?
if #available(iOS 8.0, macOS 10.12.1, *) {
    if myContext.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &authError) {
        myContext.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, localizedReason: myLocalizedReasonString) { success, evaluateError in
            if success {
                // User authenticated successfully, take appropriate action
            } else {
                // User did not authenticate successfully, look at error and take appropriate action
            }
        }
    } else {
        // Could not evaluate policy; look at authError and present an appropriate message to user
    }
} else {
    // Fallback on earlier versions
}
```

However, that will almost certainly be insufficient for a production app. For a start, before you even get as far as attempting a local authentication, you will probably want to know which form (Touch ID or Face ID, or neither) is available, so that you can change your UI appropriately. Apple [recommends](https://developer.apple.com/ios/human-interface-guidelines/user-interaction/authentication/) that you *"Don't reference Touch ID on a device that supports Face ID [and] don't reference Face ID on a device that supports Touch ID"*.

You can use the [`biometryType`](https://developer.apple.com/documentation/localauthentication/lacontext/2867583-biometrytype) property of `LAContext` which will return an `LABiometryType` of either `none`, `faceID`, or `touchID`. Be aware that the `none` case was only added in iOS 11.2. Since you will already be doing `if #available` checks for iOS 10 (Touch ID only) and iOS 11, adding another check for iOS 11.2 just to see if you can use `none` is a real pain. I opted to just use `default` in my switch on the biometry type to handle `none`. In the documentation for `biometryType` there is an important error that Apple [knows about](http://www.openradar.me/36064151) but has yet to fix. It states:

> This property is only set when canEvaluatePolicy(_:error:) succeeds for a biometric policy. The default value is LABiometryNone.

That is incorrect. Once you call `canEvaluatePolicy` on a context, the `biometryType` **is** set, regardless if the call failed or succeeded. That's important to know, because if what the docs state were true, if the call fails it would be impossible to tell if it failed because the device doesn't support Touch ID or Face ID, or because they are disabled, not enrolled or locked out. You might have UI (eg in a settings pane) which references Touch ID or Face ID even if they are currently disabled, and you want to know *which* of them it should reference. 

I ended up creating my own abstraction to make it easier to track the full state of biometry support on the device and to limit all OS version checks to a single place:


```swift
public enum BiometrySupport {
    public enum Biometry {
        case touchID
        case faceID
    }

    case available(Biometry)
    case lockedOut(Biometry)
    case notAvailable(Biometry)
    case none

    public var biometry: Biometry? {
        switch self {
        case .none, .lockedOut, .notAvailable:
            return nil
        case let .available(biometry):
            return biometry
        }
    }
}
```

I am using `notAvailable` to cover both the "not available" and "not enrolled" states that the API can return. Incidentally, if the `LAError.Code` returned from the API is `biometryNotAvailable` that includes the situation where Face ID has been disabled by the user for your app in Settings.app.

I created a `BiometricAuth` class to deal with everything related to local authentication. Below is the implementation of the `supportedBiometry` calculated property which returns a `BiometrySupport` enum:

```swift
public final class BiometricAuth {

    public var supportedBiometry: BiometrySupport {
        let context = LAContext()
        var error: NSError?

        if context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) {
            if #available(iOS 11.0, *) {
                switch context.biometryType {
                case .faceID:
                    return .available(.faceID)
                case .touchID:
                    return .available(.touchID)
                // NOTE: LABiometryType.none was introduced in iOS 11.2
                // which is why it can't be used here even though Xcode
                // errors with "non-exhaustive switch" if you don't use it 🤷🏼‍♀️
                default:
                    return .none
                }
            }

            return .available(.touchID)
        }

        // NOTE: despite what Apple Docs state, the biometryType
        // property *is* set even if canEvaluatePolicy fails
        // See: http://www.openradar.me/36064151
        if let error = error {
            let code = LAError(_nsError: error as NSError).code
            if #available(iOS 11.0, *) {
                switch (code, context.biometryType) {
                case (.biometryLockout, .faceID):
                    return .lockedOut(.faceID)
                case (.biometryLockout, .touchID):
                    return .lockedOut(.touchID)
                case (.biometryNotAvailable, .faceID), (.biometryNotEnrolled, .faceID):
                    return .notAvailable(.faceID)
                case (.biometryNotAvailable, .touchID), (.biometryNotEnrolled, .touchID):
                    return .notAvailable(.touchID)
                default:
                    return .none
                }
            } else {
                switch code {
                case .touchIDLockout:
                    return .lockedOut(.touchID)
                case .touchIDNotEnrolled, .touchIDNotAvailable:
                    return .notAvailable(.touchID)
                default:
                    return .none
                }
            }
        }

        return .none
    }
}
```

As you can see, between OS version checks and checking both success and error conditions, just getting all the information about the supported biometry on the device is quite a bit more complex than a single API call!

