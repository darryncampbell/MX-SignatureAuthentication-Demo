# MX-SignatureAuthentication-Demo

*Samples of how to generate the caller or package signature which can be provided to Zebra's MX layer through StageNow or EMM (via Managed Configurations)*

Zebra's [MX](https://techdocs.zebra.com/mx/) layer exposes configuration and provisioning capabilities for Zebra devices.  Because some of these features can be potentially misused by harmful apps, it is required to specify the signature of the signing certificate used to create the app to ensure only the desired app is given the elevated privileges.

Examples of MX functions that require you to specify the package signature are all found in the [AccessManager](https://techdocs.zebra.com/mx/accessmgr/) (at the time of writing) and include:

- Whitelist which applications can run
- Whitelist which applications can invoke MX capabilities through the [EMDK Profile Manager](https://techdocs.zebra.com/emdk-for-android/latest/guide/profile-manager-guides/)
- Allow or disallow an application to bind to a specified service
- Allow or disallow an application to call a specified service 

Examples of services that an application might want to call:

- Event injection service: Used by remote desktop apps to inject keystrokes / taps into the device.
- OEM Info: used to access many of the device identifier properties removed from standard Android APIs in version 10.

## What is a package signature?

Android applications must be signed in order to run on an Android device, this is the build step if you select `"Build" --> "Generate Signed APK / Bundle"` in Android Studio.  

For debug builds, during development, the application is signed with the Android debug key.  So, the apk is *always* signed with a key unless you deliberately choose to generate an unsigned apk. 

The Android app bundle process complicates this slightly and is discussed later in this document.

For more information on Android app signing please see the [Google documentation](https://developer.android.com/studio/publish/app-signing)

## How to specify the Package Signature

How you specify the signature to MX depends on which MX feature you are calling.  

MX will require either:
- A HEX string representation of the DER encoded certificate

OR

- A certificate file encoded as a binary DER (.crt)

In either case, the data contained within the certificate represents the same thing, **the signature of the certificate that signed the app**

## How to obtain the Package Signature

There are 3 ways to obtain the package signature:

1. Using [Zebra's App Signature Tools](https://techdocs.zebra.com/emdk-for-android/latest/samples/sigtools/) utility to extract the signature from an APK
2. Use the Java keytool utility to extract the certificate from the keystore
3. Use the Android [Package Manager](https://developer.android.com/reference/android/content/pm/PackageManager) API to extract the signature at runtime 


### 1. Using [Zebra's App Signature Tools](https://techdocs.zebra.com/emdk-for-android/latest/samples/sigtools/) utility

Assuming you have a signed apk file you can invoke the signature tools utility as follows:

**Note that you CAN NOT use the app signature tool to generate the signature of an app signed with the Google debug key**

Generate a HEX string

```
java -jar SigTools.jar getcert -inform apk -outform hex -in app.apk -outfile app.hex
```

Generate a certificate file encoded as a binary DER (.crt)
```
java -jar SigTools.jar getcert -inform apk -outform der -in app.apk -outfile app.crt
```

You can then provide either the HEX string or .crt file to MX as required.

### 2. Use the Java keytool utility to extract the certificate from the keystore

*The Java keytool utility is part of the Java JDK installation and can be found in your `JAVA_HOME\bin` directory*

**Apps signed with your app signing key (Build --> Generate Signed APK)**

When you generated your apk file, one of the inputs was a Java keystore (.jks) file.  This .jks file is used as the input to the keytool utility as follows:

```
keytool -exportcert -alias mykeyalias -keystore mykey.jks -file app.crt
```

Note: The .crt created by the above step will be identical to the .crt file output by Zebra's app signature tool, provided the apk was signed by the same key. 

You can convert between the binary certificate and the HEX representation in a number of ways.  A simple python script to perform the conversion would look as follows:

```python
import binascii
filename = 'app.crt'
with open(filename, 'rb') as f:
   content = f.read()
hex = binascii.hexlify(content).decode("utf-8").upper()
with open ('app.hex', 'w') as f:
   f.write(hex)
   f.close()
```

Note: The .hex file created by this conversion will match the .hex file output by Zebra's app signature tool, provided the apk was signed with the same key.

You can then provide either the HEX string or .crt file to MX as required.

**Apps signed with the Android debug key**

For development purposes you will likely also need to provide MX with the signing key for the debug build of your application.

Note that the SDK tools create the debug keystore/key with predetermined names/passwords as follows:

```java
Keystore name: "debug.keystore"
Keystore password: "android"
Key alias: "androiddebugkey"
Key password: "android"
CN: "CN=Android Debug,O=Android,C=US"
```

Locate your debug.keystore file which by default will be in your HOME or USER directory/.android/debug.keystore

```
keytool -exportcert -alias androiddebugkey -keystore debug.keystore -file debug.crt
```

You will be prompted for a password, as stated above, the password is `android`

If required, you can convert the .crt output by the keytool command to a HEX string using the python script given earlier.

### 3. Use the Android [Package Manager](https://developer.android.com/reference/android/content/pm/PackageManager) API 

You can use the [Package Manager](https://developer.android.com/reference/android/content/pm/PackageManager) API to extract the signing certificate signature at runtime.  

A change was introduced in API level 28 to deprecate one of the flags so ensure your application is compatible with the API level you are targeting.  Note that it is possible, though rare, for an application to be signed with multiple certificates.

**API level 27 and earlier**

```java
Signature[] sigs;
sigs = getPackageManager().getPackageInfo(getPackageName(), PackageManager.GET_SIGNATURES).signatures;
for (Signature sig : sigs)
{
   Log.d(TAG, "Signature : " + sig.toCharsString() + " Length: " + sig.toCharsString().length());
}
```

**API level 28 and later**

```java
Signature[] sigs;
SigningInfo signingInfo = new SigningInfo();
signingInfo = getPackageManager().getPackageInfo(getPackageName(), PackageManager.GET_SIGNING_CERTIFICATES).signingInfo;
sigs = signingInfo.getApkContentsSigners();
for (Signature sig : sigs)
{
   Log.d(TAG, "Signature : " + sig.toCharsString() + " Length: " + sig.toCharsString().length());
}
```

The above example(s) will output the signature in HEX format to logcat.

**For obvious reasons, do not output the package signature to logcat in a production build**

To convert these HEX strings to a binary .crt format you can use any HEX to binary conversion, e.g. the python script below:

```python
import binascii
filename = 'app.hex'
with open(filename, 'r') as f:
   content = f.read()
bin = binascii.unhexlify(content)
with open ('app.crt', 'wb') as f:
   f.write(bin)
   f.close()
```

## Android App Bundles

Android App Bundles make it possible to allow Google Play to sign your application for you.  There are a number of advantages to this including dynamic delivery of localized content and smaller download sizes and Google will recommend this as the default approach in their [documentation on app signing](https://developer.android.com/studio/publish/app-signing).  

Zebra's MX is comparing the signature of the **app signing key**, this is important since if you are using app bundles, the app signing is applied by Google rather than the developer. 

You can provide your own signing key for Google to use when signing your app, derived from your keystore thereby allowing you to use the Java keytool utility to extract the certificate used for signing, *though this scenario has not been extensively tested with MX*

Many enterprises choose to continue to [manage their own signing keys](https://developer.android.com/studio/publish/app-signing#opt-out) and not take advantage of app bundles, preferring to not share their signing key with Google. 
