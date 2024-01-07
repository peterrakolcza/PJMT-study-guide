# Android Dynamic Analysis

1. [Intro to SSL Pinning](#intro-to-ssl-pinning)
1. [Dynamic Analysis using MobSF](#dynamic-analysis-using-mobsf)
1. [Burp Suite](#burp-suite)
1. [Patching Apps Automatically using Objection](#patching-apps-automatically-using-objection)
1. [Patching Apps Manually](#patching-apps-manually)
1. [Dynamic Analysis Vectors](#dynamic-analysis-vectors)

## Intro to SSL Pinning

* SSL Pinning:

  * security methodology used to ensure app traffic is not being intercepted (MITM)
  * some apps verify that received traffic is coming from a known certificate; we can import a certificate as root, but it still might not be trusted by app
  * can use tools such as ```Burp Suite``` and ```Proxyman``` to break SSL Pinning

* Android interception process:

  1. Start proxy software
  1. Configure proxy software
  1. Set proxy of emulator (or physical device)
  1. Intercept HTTP traffic
  1. Import CA certificate
  1. Trust CA certificate in Android Certificate Store
  1. Intercept HTTPS traffic if possible, else try Objection/Frida

## Dynamic Analysis using MobSF

* We need to setup the [MobSF dynamic analyser](https://mobsf.github.io/docs/#/dynamic_analyzer) for dynamic analysis; we can use the ```Android Studio Emulator``` option.

* Dynamic analysis in MobSF requires the AVD to not have Google Play Store; Android images upto API 28 are supported.

```shell
# setup MobSF dynamic analyzer
# add emulator to path

emulator -list-avds
# lists AVDs

emulator -avd <name of AVD> -writable-system -no-snapshot
# this runs the emulator without Android Studio this time

# open another tab
cd Mobile-Security-Framework-MobSF

./run.sh
# when started, go to link for dynamic analyzer
```

## Burp Suite

* Burp Suite:

  * Under settings for ```Proxy```, configure a proxy listener bound to port 8082 and all interfaces

  * For the running virtual device, under extended controls, do manual proxy configuration for port 8082 at 127.0.0.1

  * From Burp Suite, under ```Proxy``` settings, we have option for ```import/export CA certificate``` - export certificate in DER format

  * Save the certificate file in the format 'filename.CER' for compatibility

  * Drag and drop the cert file into the virtual device running on Android Studio

  * On the virtual device, we can go into Settings > Trusted credentials > Install from SD card - select the '.CER' file, we can give it any time and use for 'VPN & apps'

  * In Burp Suite, toggle ```Intercept``` to On

  * Now, if we go to any website on our virtual device, the traffic would be intercepted by Burp Suite

  * We can now solve Flag 17 on ```InjuredAndroid``` as it involves bypassing SSL pinning and thus intercepting traffic

## Patching Apps Automatically using Objection

* [objection](https://github.com/sensepost/objection) can be used to automatically patch Frida into the app - used for runtime mobile exploration.

```shell
# while the emulator is running in background

pip3 install frida-tools

pip3 install objection

objection patchapk --source injuredandroid-pulled.apk
# uses running emulator
# automates patching apk

ls -la
# saves patched copy as injuredandroid-pulled.objection.apk
# we can install this apk in emulator
# might give certificate error
# so we will have to use manual method
```

## Patching Apps Manually

* [Guide on Android app manual patching process](https://koz.io/using-frida-on-android-without-root/)

```shell
apktool d -r injuredandroid-pulled.apk
# decompile the apk
# -r to not decompile Resources

# we need to inject frida-gadget into the /lib folder
# arch of emulator should be known
# in this case, it is x86_64

# from Frida releases on Github
# search for frida-gadget releases for particular arch

wget https://github.com/frida/frida/releases/download/16.1.1/frida-gadget-16.1.1-android-x86_64.so.xz

# extract contents from file
# to get .so file
# rename extracted file to frida-gadget.so

mv frida-gadget.so ~/injuredandroid-pulled/lib/x86_64
# move frida-gadget file into /lib/x86_64
# along with other .so files

# to follow naming convention of .so files
mv frida-gadget.so libfrida-gadget.so

# now, add reference to frida-gadget to SMALI code
# found in /smali/b3nac/injuredandroid
# in a known exported activity or main activity

# const-string v0, "frida-gadget" invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V

# add the above line into MainActivity.smali or OnboardingActivity.smali
# in the public constructor method

# now recompile the app
apktool b injuredandroid-pulled -o injured-patched.apk
# builds modified apk

# creating keystore on our own
# for signing the app
keytool -genkey -v -keystore demo.keystore -alias demokey -keyalg RSA -keysize 2048 -validity 10000
# we will be prompted password, details of app
# this generates a keystore

# sign apk using jarsigner
jarsigner -sigalg SHA1withRSA -digestalg SHA1 -keystore demo.keystore -storepass password123 injured-patched.apk demokey
# required for running app on Android

jarsigner -verify injured-patched.apk
# verify if app is signed

# zipalign final version
zipalign 4 injured-patched.apk injured-patchedfinal.apk
# move this file into emulator now

# once app is started on emulator
# we can use objection

objection explore
# this identifies the emulator
# and gives us a shell

android sslpinning disable
# disables SSL pinning
```

## Dynamic Analysis Vectors

```shell
# using objection
# while app is running on emulator

objection explore

# in objection shell
# we have several options

android clipboard monitor
# can be used to monitor clipboard

android keystore list
# list keystore

# also use Android Studio's File Explorer
# to check and inspect app files and data

# we can also use Frida Codeshare scripts for extra functionality
```

* Dealing with Split APKs manually:

  * Pull all apks and base apk off device:
  
    ```shell
    # in adb shell
    pm list packages | grep myappname
    
    pm path myapppath
    ```

  * Inject base.apk with objection and sign all split apks:

    ```shell
    adb pull <base.apk, split_config.apk, etc.>

    objection patchapk -s base.apk --use-aapt2

    # after app is signed and patched
    # all split apks must be signed
    objection signapk split_config.apk
    ```
  
  * Install all apks to device:

    ```shell
    adb install-multiple base.objection.apk split_config.apk
    ```

## Frida Codeshare and Startup Scripts
These scripts written in Javascript can be used to add various functionalities.
https://codeshare.frida.re/

After saving locally a script, use the following command to run the script:
```shell
    objection explore --startup-script sslpinninguniversal.js
```

Or run objection commands right after startup:
```shell
    objection explore --s "android root disable"
```
