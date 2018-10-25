# Fingerprint Authentication

# Android

### Updating the manifest

Our app is going to require access to the device’s touch sensor in order to receive fingertip touch events. However, the Android operating system runs on a wide range of devices, and not every one of these devices includes a touch sensor.

```xml
<uses-feature
android:name="android.hardware.fingerprint"
android:required="false"/>
```

```xml
<uses-permission
  android:name="android.permission.USE_FINGERPRINT" />
```
Since this permission is categorized as a normal permission , there is no need to add it as runtime permission.

### The UI

Google has supplied the red lines for creating a Fingerprint Authentication dialog:
![Fingerprint Authencication dialog](https://res.cloudinary.com/practicaldev/image/fetch/s--hsZlHzPm--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://thepracticaldev.s3.amazonaws.com/i/pzr4i4dd9zzi54vgsyhw.png)

It is recommanded to follow the Google's [Material Design](https://material.io/design/#)

You can find all necessary icons 
[here](https://material.io/tools/icons/?icon=fingerprint&style=baseline)

### The general process

The authentication is performed in a two step operation. 

In the first half, we’re going to focus on checking that the device has the hardware, software and settings required to support fingerprint authentication, and in the second half we’re going to create the key, cipher and CryptoObject that we’ll use to perform the actual authentication.

Specifically, in this first part of our MainActivity file we’re going to check that:

* **The device is running Android 6.0 or higher.** If your project’s minSdkversion is 23 or higher, then you won’t need to perform this check.

* **The device features a fingerprint sensor.** If you marked android.hardware.fingerprint as something that your app requires (android:required=”true”) then you don’t need to perform this check.

* **The user has granted your app permission to access the fingerprint sensor.**

* **The user has protected their lockscreen.** Fingerprints can only be registered once the user has secured their lockscreen with either a PIN, pattern or password, so you’ll need to ensure the lockscreen is secure before proceeding.
* **The user has registered at least one fingerprint on their device.**

If any of the above requirements aren’t met, then your app should gracefully disable all features that rely on fingerprint authentication and explain why the user cannot access these features. You may also want to provide the user with an alternative method of confirming their identity, for example by giving them the option to create a password and username.

In the second half of our MainActivity file, we’re going to complete the following:

* Gain access to the Android keystore, by generating a Keystore instance. The Android keystore allows you to store cryptographic keys in a way that makes them more difficult to extract from the device. The keystore also restricts how and when each key can be used. To create that fingerprint authentication effect, you just need to specify that the user has to authenticate their identity with a fingerprint every time the want to use this key.
* Create a new method (I’m going to use generateKey) that’ll be responsible for generating the app’s encryption key.
* Use the generateKey function to generate the app’s encryption key.
* Create a new method (I’m using initCipher) that we’ll use to initialize the cipher.
* Use the Cipher instance to create an encrypted CryptoObject instance.
* Assign the CryptoObject to the instantiated FingerprintManager.


### Initializing the KeyStore and generating key

A secret key should be created before authentication process. Generated key will be stored securely on device by using [`KeyStore`](https://developer.android.com/training/articles/keystore) instance and used for initializing cipher object a little later. First , gain access to keystore.

```java
final KeyStore keyStore = KeyStore.getInstance("AndroidKeyStore");
keyStore.load(null);
```

Generate a secret key after getting [`KeyGenerator`](https://developer.android.com/reference/javax/crypto/KeyGenerator) instance. Arguments below are using to specify type of the key. `KEY_STORE_ALIAS` is container for key to be saved.

```java
final KeyGenerator keyGenerator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES,
        "AndroidKeyStore");
keyGenerator.init(new
        KeyGenParameterSpec.Builder(KEY_STORE_ALIAS,
        KeyProperties.PURPOSE_ENCRYPT | KeyProperties.PURPOSE_DECRYPT)
        .setBlockModes(KeyProperties.BLOCK_MODE_CBC)
        .setUserAuthenticationRequired(true)
        .setEncryptionPaddings(
         KeyProperties.ENCRYPTION_PADDING_PKCS7)
        .build());
keyGenerator.generateKey();
```

Before every use of the generated key , `setUserAuthenticationRequired(true)` method is used to decide whether this key is authorized to be used only if user has been authenticated. That is applied to secret key or private key operations.


#### Keyguard Manager
This class provides access to your lock screen.

```java
KeyguardManager keyguardManager = (KeyguardManager)
        .getSystemService(Context.KEYGUARD_SERVICE);
keyguardManager.isKeyguardSecure();
```

Returns whether the keyguard is secured by a PIN, pattern, password or a SIM card is currently locked.

#### Fingerprint Manager
[`FingerprintManager`](https://developer.android.com/reference/android/hardware/fingerprint/FingerprintManager) is fundamental of using fingerprint scan. Most of the fingerprint related methods can be found under this class.

```java
FingerprintManager fingerprintManager = (FingerprintManager)
        .getSystemService(Context.FINGERPRINT_SERVICE);
```

After getting instance of FingerprintManager , you will be able to use some methods of this class such as `isHardwareDetected()`, `hasEnrolledFingerprints()` and `authenticate(…)` .

* **`fingerprintManager.isHardwareDetected()`** Checks  wheter fingerprint sensor does exist or is available

* **`fingerprintManager.hasEnrolledFingerprints()`** Checks if there is at least one enrolled fingerprint on device.

* **`fingerprintManager.`[`authenticate`](https://developer.android.com/reference/android/hardware/fingerprint/FingerprintManager#authenticate%28android.hardware.fingerprint.FingerprintManager.CryptoObject,%20android.os.CancellationSignal,%20int,%20android.hardware.fingerprint.FingerprintManager.AuthenticationCallback,%20android.os.Handler%29)`(`[`FingerprintManager.CryptoObject`](https://developer.android.com/reference/android/hardware/fingerprint/FingerprintManager.CryptoObject) `crypto, `[`CancellationSignal`](https://developer.android.com/reference/android/os/CancellationSignal) `cancel, int flags,` [`FingerprintManager.AuthenticationCallback`](https://developer.android.com/reference/android/hardware/fingerprint/FingerprintManager.AuthenticationCallback) `callback,` [`Handler`](https://developer.android.com/reference/android/os/Handler) `handler)`** Tries to authenticate cypto object. The fingerprint hardware starts listening for scan after that call. Before calling this method , appropriate parameters should be prepared.

##### Parameter 1: CryptoObject

Before creating new instance of CryptoObject , you need to initialize Cipher object as follows

```java
try {
    final Cipher cipher = Cipher.getInstance(
                   KeyProperties.KEY_ALGORITHM_AES + "/"
                    + KeyProperties.BLOCK_MODE_CBC + "/"
                    + KeyProperties.ENCRYPTION_PADDING_PKCS7);
    final SecretKey key;
    final KeyStore keyStore =  KeyStore.getInstance(ANDROID_KEY_STORE);
keyStore.load(null);
    key = (SecretKey) keyStore.getKey(KEY_STORE_ALIAS, null);
    cipher.init(Cipher.ENCRYPT_MODE, key);
    return cipher;
} catch (KeyPermanentlyInvalidatedException e) {
    return false;
} catch (KeyStoreException | CertificateException | UnrecoverableKeyException | IOException
        | NoSuchAlgorithmException | InvalidKeyException | NoSuchPaddingException e) {
    throw new RuntimeException("Failed to init Cipher", e);
}
```

This initialization of Cipher object will be used to create CryptoObject instance. While initializing cipher , generated and stored key in keystore container is used. Successfully initialized cipher means that previously stored key is not invalidated and it is still available for use.

###### Passing CryptoObject as an argument

```java
if (initCipher()) {
cancellationSignal = new CancellationSignal();
fingerprintManager.authenticate(new FingerprintManager.CryptoObject(cipher),
        cancellationSignal,
        0,     // flags
        this,  // authentication callback 
        null); // handler
}
```

After cipher object is initialized , crypto object is created by using this cipher instance. Generated crypto object should be passed as a parameter for authenticate() method. That method has some useful callbacks which I’ll talk about later.

##### Parameter 2: CancellationSignal

```java
private void stopListeningAuthentication() {
    if (cancellationSignal != null) {
        cancellationSignal.cancel();
        cancellationSignal = null;
    }
```

`CancellationSignal` is used to cancel scan in progress. By calling cancel method and setting this object to null , you are able to stop listening fingerprint scan anytime. (error comes or user presses cancel button… etc.)

##### Parameter 3: Flags
It represents optional flags and can be set to zero.

##### Parameter 4: Authentication Callback
A class instance extends from [`FingerprintManager.AuthenticationCallback`](https://developer.android.com/reference/android/hardware/fingerprint/FingerprintManager.AuthenticationCallback) to receive authentication results.

##### Parameter 5: Handler
It’s an optional parameter. If it is set optionally , `FingerprintManager` will use Looper class from this handler for its inner `MyHandler` class.

#### KeyPermanentlyInvalidatedException

[This exception](https://developer.android.com/reference/android/security/keystore/KeyPermanentlyInvalidatedException) is thrown when fingerprint enrollment has changed on device while initializing cipher. This exception occurs for keys which are to be used only if the user has been authenticated.
Keys are permanently invalidated once a new fingerprint is enrolled on device or all fingerprints on device are removed. Also, if secure lock screen is disabled or forcibly reset , keys are permanently and irreversibly invalidated.
If you need to authenticate user despite this exception , you must call `initCipher()` method again with a valid key.

> You should generate a new key before initializing cipher because the old one is invalidated.

### Authentication Callbacks

After a helper class extends from [`FingerprintManager.AuthenticationCallback`](https://developer.android.com/reference/android/hardware/fingerprint/FingerprintManager.AuthenticationCallback) is created, the following callbacks are called depending on result of authenticate(CryptoObject, CancellationSignal, int, AuthenticationCallback, Handler) method.

```java
/**
 * Called when an unrecoverable error has been encountered and the operation is complete.
 * No further callbacks will be made on this object.
 * @param errorCode An integer identifying the error message
 * @param errString A human-readable error string that can be shown in UI
 */
public void onAuthenticationError(int errorCode, CharSequence errString) { }
/**
 * Called when a recoverable error has been encountered during authentication. The help
 * string is provided to give the user guidance for what went wrong, such as
 * "Sensor dirty, please clean it."
 * @param helpCode An integer identifying the error message
 * @param helpString A human-readable string that can be shown in UI
 */
public void onAuthenticationHelp(int helpCode, CharSequence helpString) { }
/**
 * Called when a fingerprint is recognized.
 * @param result An object containing authentication-related data
 */
public void onAuthenticationSucceeded(AuthenticationResult result) { }
/**
 * Called when a fingerprint is valid but not recognized.
 */
public void onAuthenticationFailed() { }
```

> You can find defined help and error codes by following this [link](https://developer.android.com/reference/android/hardware/fingerprint/FingerprintManager.html#FINGERPRINT_ACQUIRED_GOOD).


## Best Practices

* **Consider backwards compatibility.** Fingerprint authentication didn’t find its way into the Android platform until version 6.0. While it does have plenty to offer and can greatly improve the user experience, chances are you’re not wild about the idea of creating an app that’s incompatible with every Android device running Lollipop or earlier! We’ve already explored using Build.VERSION checks and @TargetApi annotations to include fingerprint authentication in your app while remaining backwards compatible with earlier versions of Android. However, you can also use the [v4 support library](https://developer.android.com/reference/android/support/v4/hardware/fingerprint/package-summary), which provides compatibility version of many of the fingerprint classes introduced in Marshmallow. If you do use this library, then when your app is installed on Lollipop or earlier it’ll behave as though the device doesn’t feature a fingerprint sensor – and overlook the fact that the operating system is incapable of supporting fingerprint authentication.
* **Provide alternate methods of authentication.** There are a number of reasons why the user might be unable to use your app’s fingerprint authentication. Maybe they’re running a pre-Marshmallow version of Android, maybe their device doesn’t include a fingerprint sensor, or maybe they haven’t configured their device to support fingerprint authentication. However there may also be some users who simply don’t want to use fingerprint authentication – some people may simply prefer to use a traditional password. In order to provide the best possible experience for all your users, you should consider providing an alternate method of authentication for users who are unable or unwilling to use your app’s fingerprint authentication.
* **Clearly indicate when your app is “listening” for user input.** Don’t leave the user wondering whether they’re supposed to press their finger to the sensor now, or wait for further instructions. Google recommends that you display the standard fingerprint icon whenever you app is ready to receive a touch event, but depending on the context and your target audience you may want to consider supplementing this icon with clear text instructions – which is exactly what we’re doing with our sample app’s “instructions” string.
* **If the device cannot support finger authentication, then explain why.** There’s a list of requirements that a device needs to meet before it can support fingerprint authentication. If the device doesn’t fulfil one or more of these requirements then you should disable all of your app’s fingerprint features, but disabling sections of your app without providing an explanation is never a good idea! Best case scenario, you’ll leave the user wondering what they’ve done wrong – worst case scenario, they’ll assume that your app is broken and leave you a negative review on Google Play. You should always let the user know why they can’t access part of your app and, ideally, provide them with instructions on how they can ‘unlock’ this part of your app’s functionality.
* **Provide the user with plenty of feedback.** Whenever the user touches their device’s fingerprint sensor, the authentication can succeed, fail or an error can occur – and you should never leave your users wondering which one has just happened! Imagine you press your fingertip to your device’s touch sensor when prompted, and nothing happens. What went wrong? Maybe some dirt on the sensor interfered with the authentication process; maybe you didn’t press on the sensor long enough, or maybe the app is broken and you should give it a negative review on Google Play immediately? To ensure your users can navigate your app’s fingerprint authentication successfully, use the fingerprint authentication callback methods to provide the user with all the information they need to understand when authentication has failed, and why.



## Sources

* [How to add fingerprint authentication to your Android app](https://www.androidauthority.com/how-to-add-fingerprint-authentication-to-your-android-app-747304/) - Android Authority

* [Integrate fingerprint authentication into your Android apps](https://medium.com/commencis/integrate-fingerprint-authentication-into-your-android-apps-dcb977b2e846) - medium

# IOS

## Apple documentation:

### Overview:

Many users rely on biometric authentication like Face ID or Touch ID to enable secure, effortless access to their devices. As a fallback option, and for devices without biometry, a passcode or password serves a similar purpose. Use the LocalAuthentication framework to leverage these mechanisms in your app and extend authentication procedures your app already implements.

![](https://docs-assets.developer.apple.com/published/08a1846d5e/b32218fc-f538-412c-80d7-183c920d9429.png)

To maximize security, your app never gains access to any of the underlying authentication data. You can’t access any fingerprint images, for example. The Secure Enclave, a hardware-based security processor isolated from the rest of the system, manages this data out of reach even of the operating system. Instead, you specify a particular policy and provide messaging that tells the user why you want them to authenticate. The framework then coordinates with the Secure Enclave to carry out the operation. Afterward, you receive only a Boolean result indicating authentication success or failure.

### Set the Face ID Usage Description

In any project that uses biometrics, include the [`NSFaceIDUsageDescription`](https://developer.apple.com/library/content/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW75) key in your app’s `Info.plist` file. Without this key, the system won’t allow your app to use Face ID. The value for this key is a string that the system presents to the user the first time your app attempts to use Face ID. The string should clearly explain why your app needs access to this authentication mechanism. The system doesn’t require a comparable usage description for Touch ID.

### Create and Configure a Context

You perform biometric authentication in your app using an [`LAContext`](https://developer.apple.com/documentation/localauthentication/lacontext) instance, which brokers interaction between your app and the Secure Enclave. Begin by creating a context:

```objective-c
var context = LAContext()
```

You can customize the messaging used by the context to guide the user through the flow. For example, you can set a custom message for the Cancel button that appears in various alert views:

```objective-c
context.localizedCancelTitle = "Enter Username/Password"
```

This helps the user understand that when they tap the button, they’ll be reverting to your normal authentication procedure.

### Test Policy Availability

Before attempting to authenticate, test to make sure that you actually have the ability to do so by calling the [`canEvaluatePolicy:error:`](https://developer.apple.com/documentation/localauthentication/lacontext/1514149-canevaluatepolicy?language=objc) method:

```objective-c
var error: NSError?
if context.canEvaluatePolicy(.deviceOwnerAuthentication, error: &error) {
```

Choose a value from the [`LAPolicy`](https://developer.apple.com/documentation/localauthentication/lapolicy?language=objc) enumeration for which to test. The policy controls how the authentication behaves. For example, the [`LAPolicyDeviceOwnerAuthentication`](https://developer.apple.com/documentation/localauthentication/lapolicy/lapolicydeviceownerauthentication?language=objc) policy used in this sample indicates that reverting to a passcode is allowed when biometrics fails or is unavailable. Alternatively, you can indicate the [`LAPolicyDeviceOwnerAuthenticationWithBiometrics`](https://developer.apple.com/documentation/localauthentication/lapolicy/lapolicydeviceownerauthenticationwithbiometrics?language=objc) policy, which doesn’t allow reverting to the device passcode.

### Evaluate a Policy

When you’re ready to authenticate, call the [`evaluatePolicy:localizedReason:reply:`](https://developer.apple.com/documentation/localauthentication/lacontext/1514176-evaluatepolicy?language=objc) method, using the same policy you already tested:

```objective-c
let reason = "Log in to your account"
context.evaluatePolicy(.deviceOwnerAuthentication, localizedReason: reason ) { success, error in

    if success {

        // Move to the main thread because a state update triggers UI changes.
        DispatchQueue.main.async { [unowned self] in
            self.state = .loggedin
        }

    } else {
        print(error?.localizedDescription ?? "Failed to authenticate")

        // Fall back to a asking for username and password.
        // ...
    }
}
```

For Touch ID, or when the user enters a passcode, the system displays the reason for authenticating that you provided in the method call. It’s important to provide a clear explanation, localized for any regions in which you operate, of why your app is asking the user to authenticate. The name of your app already appears before the reason you give, so you don’t need to include that in your message.

### Optionally, Adjust Your User Interface to Accommodate Face ID
Aside from the biometric scanning operation itself, Face ID exhibits an important difference from Touch ID. When your app tries to use Touch ID, a system message asks the user to present a finger to scan. The user has time to think about and possibly abort the operation by canceling the prompt. When your app invokes Face ID, the device begins scanning the user’s face right away. Users don’t get a final opportunity to cancel. To accommodate this behavioral difference, you might want to provide a different UI, depending on the kind of biometrics available on the device.

The sample app provides a different UI by including a text label with a message that warns Face ID users that tapping the button results in an immediate Face ID scan. The app hides the label by default, revealing it only for users with Face ID who are currently logged out. Users with Touch ID (or no biometry at all) don’t need the message because they can cancel the operation before the scan actually takes place.
You test what kind of biometry the device supports by reading the context’s [`biometryType`](https://developer.apple.com/documentation/localauthentication/lacontext/2867583-biometrytype?language=objc) parameter:

```objective-c
faceIDLabel.isHidden = (state == .loggedin) || (context.biometryType != .faceID)
```

This parameter only contains a meaningful value after you run the [`canEvaluatePolicy:error:`](https://developer.apple.com/documentation/localauthentication/lacontext/1514149-canevaluatepolicy?language=objc) method on the context at least once.

### Provide a Fallback Alternative to Biometrics

For various reasons, authentication sometimes fails or is unavailable:

* The user’s device doesn’t have Touch ID or Face ID.
* The user isn’t enrolled in biometrics, or doesn’t have a * passcode set.
* The user cancels the operation.
* Touch ID or Face ID fails to recognize the user.
* You’ve previously invalidated the context with a call to the [`invalidate`](https://developer.apple.com/documentation/localauthentication/lacontext/1514192-invalidate?language=objc) method.

For a complete list of possible error conditions, see [`LAError`](https://developer.apple.com/documentation/localauthentication/laerror?language=objc).

This sample app doesn’t implement alternative authentication. In a real app, if you encounter a local authentication error, fall back to your own authentication scheme, like asking for a username and password. Use biometrics as a supplement to something you’re already doing. Don’t depend on biometrics as your only authentication option.


## Implementation

```objective-c
LAContext *myContext = [[LAContext alloc] init];
NSError *authError = nil;
NSString *myLocalizedReasonString = @"Used for quick and secure access to the test app";
 
if ([myContext canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics error:&authError]) {
    [myContext evaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics
                  localizedReason:myLocalizedReasonString
                            reply:^(BOOL success, NSError *error) {
            if (success) {
                // User authenticated successfully, take appropriate action
            } else {
                // User did not authenticate successfully, look at error and take appropriate action
            }
        }];
} else {
    // Could not evaluate policy; look at authError and present an appropriate message to user
}
```

**Line 1:** Here we create an LAContext object. The LAContext class is responsible for handling the context for the authentication. Put simply, we use an LAContext object to check if a type of authentication is available. In the case of this tutorial, we will later be checking “if” touch ID is an option.

**Line 2:** We need an NSError so that the LAContext can use it to return if there is an error.

**Line 3:** We set an NSString with a description that it put on screen to let the user know why the touch ID view has appeared on screen.

**Line 5:** This is where we put the LAContext constant to use by calling the canEvaluatePolicy: method and sending it an LAPolicy constant as an argument. In this case, we pass LAPolicyDeviceOwnerAuthenticationWithBiometrics. If this fails, either touch ID is not configured on a compatible device, or touch ID is not available on the device… think an iPhone 4S, 5 or 5c running the app. Also, this doesn’t take in to account a device running iOS 7, so if you plan to run finger print authentication on an app, make sure you check that you are working with a compatible device and if not, make other options available such as password on pin code to access the app.

**Lines 6, 7 and 8:** If the user can authenticate with biometrics we can now call the evaluatePolicy method on our LAContext object. We do this by passing the same constant over, LAPolicyDeviceOwnerAuthenticationWithBiometrics, as well as passing our reason string and then specifying a block for the response to be handled.

We will get either a YES or a NO as a result. If a YES then **line 10** is where we put code for a positive response. Likewise, **line 12** is where we put our failure code.

Finally on **line 15**, we have the ELSE statement which runs if **line 5** fails the test… i.e., biometrics is not available. We can check the authError pointer to get the reason and present it to the user if needed.

Finally, to get this to not show errors, we need to import the local authentication framework in to our project:

```objective-c
#import "ViewController.h"
#import <LocalAuthentication/LocalAuthentication.h>
 
@interface AuthenticationViewController ()
 
@end
```

## Sources:

[Touch ID Tutorial - Objective-C](https://www.devfright.com/touch-id-tutorial-objective-c/) - DEVFRIGHT

[Logging a User into Your App with Face ID or Touch ID](https://developer.apple.com/documentation/localauthentication/logging_a_user_into_your_app_with_face_id_or_touch_id?language=objc) - Apple Developer
