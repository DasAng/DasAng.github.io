+++
date = "2024-05-28T01:00:00-00:00"
title = "How to add Social Log In for Android Game using Unity and Firebase: Part 4"
image = "/images/unity_firebase.jpg"

+++

Welcome to the fourth and last installment of our series, where we'll guide you through adding social login functionality to your Unity game on Android. We’ll demonstrate how to build a simple Android application in Unity and integrate login using Google accounts, as well as manage users in Firebase. I will share with you the good, the bad and the ugly part of what I've learned during the process.

## What we will cover in Part 4

In **[part3]({{< ref "unity_firebase_part3" >}})**, we've written C# code using the **i5 Toolkit for Unity** to manage the OAuth authentication process. Additionally, we've implemented code for Firebase authentication. During testing, we confirmed that everything functions correctly when running the game within the Unity editor. In this segment of the series, we'll proceed to install and run our game on an Android phone.

- [Build and install on Android](#build-and-install-on-android)
- [Stream logs from Android to your computer](#stream-logs-from-android-to-your-computer)
- [Start game on Android](#start-game-on-android)
- [Troubleshooting with remote inspection using Chrome DevTools](#troubleshooting-with-remote-inspection-using-chrome-devtools)

## Build and install on Android

First we will need to build an Android Package (APK) so we can install it on an Android Phone.

1. Select **File** ➡️ **Build Settings...**
2. Go to **Player Settings...**
3. Make sure to enter the password for both your project key store and project key
4. Click **Build** and choose name and location of where to output the APK file. I have chosen the name of **SocialLogin.apk**

![](/images/unity_player_settings_password.png "Figure 1: Enter password for key store and project key")

Once the build completes we will have an APK file that we can copy over to our Android phone.

Connect your Android phone to your development computer via USB and **transfer the APK file**. Ensure you have enabled **USB Debugging** on your Android phone. We will use this to remotely stream the logs to our computer. Click on the APK file to install it.

## Stream logs from Android to your computer

Once you have installed the APK file on your Android phone, we can start streaming logs to our computer. To do this locate the `adb` tools on your computer. I have mine installed in the following location `C:\Program Files\Unity\Hub\Editor\2022.3.29f1\Editor\Data\PlaybackEngines\AndroidPlayer\SDK\platform-tools\adb.exe`. Execute the following command:

```shell
adb logcat -s Unity
```

The output should look something like:

![](/images/adb_logcat.png "Figure 2: Output from adb")

## Start game on Android

Now start the game and press the **Sign In** button, logs from the game should start appearing in the adb console. Once you have signed in to your Google Account you should be directed back to the game. The output from the logs should show login completed.

![](/images/adb_login_completed.png "Figure 3: Output from game shows login completed")

To verify that our user has been created and linked to our Google Account in Firebase, navigate to Firebase console and in the **users** menu check that our user is present.

## Troubleshooting with remote inspection using Chrome DevTools

In case something goes wrong and you wanted to inspect network requests you can use the Chrome DevTools for remote inspection on your mobile devices.

1. Go to `chrome://inspect#devices` on your chrome browser
2. Click on the remote target page you wish to inspect.

![](/images/chrome_remote_inspect.png "Figure 4: Remote inspect mobile devices browser")

## Conclusion

In this final segment of our series, we've successfully built and installed our game on an Android phone. We've meticulously streamed the logs from our game directly to our computer to verify that everything functions as intended. Additionally, we've ensured that Firebase has created our user account and linked it to our Google Account. Once signed in, we can utilize the user's ID Token to authorize against any Google API services.

Throughout this extensive series, I hope you've gained a solid understanding of authentication using Firebase and the OAuth flow for signing in to Google Accounts. We've delved into how the OpenID Connect client from the **i5 Toolkit for Unity** significantly streamlined the OAuth process.
