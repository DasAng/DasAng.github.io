+++
date = "2024-05-24T10:00:00-00:00"
title = "How to add Social Log In for Android Game using Unity and Firebase: Part 2"
image = "/images/unity_firebase.jpg"

+++

Welcome to the second installment of our series, where we'll guide you through adding social login functionality to your Unity game on Android. We’ll demonstrate how to build a simple Android application in Unity and integrate login using Google accounts, as well as manage users in Firebase. I will share with you the good, the bad and the ugly part of what I've learned during the process.

## What we will cover in Part 2

In [part1]({{< ref "unity_firebase_part1" >}}) we covered how to setup Unity to build our Android Game. In this part of the series, we’ll explore how to set up OAuth Client IDs in Google Cloud, enabling players to sign in using their Google Accounts. Additionally, we’ll establish our first Firebase Authentication service and link it to our Google Cloud Provider

- [Create Google API Credentials](#create-google-api-credentials)
    - [Get Package name](#get-package-name)
    - [Get SHA-1 certificate fingerprint](#get-sha-1-certificate-fingerprint)
    - [Create OAuth Client ID](#create-oauth-client-id)
- [Create Firebase project and register Unity application](#create-firebase-project-and-register-unity-application)
    - [Setup Google sign in provider](#setup-google-sign-in-provider)
    - [Setup Firebase SDK in Unity](#setup-firebase-sdk-in-unity)
    - [Set up Gradle to build Android](#set-up-gradle-to-build-android)
    - [Fix Firebase Loading issues](#fix-firebase-loading-issues)

## Create Google API Credentials

To enable sign-in using a Google Account in our Android game, we require an **OAuth Client ID** that our Android game will utilize during the sign-in authentication process. But before we can create the OAuth Client ID, we need to first configure the **OAuth Consent Screen**. This screen provides the information that will be presented to the user whenever they attempt to sign in using their Google Account.

If you haven’t already created a **Google Cloud Project**, you can do so. Then navigate to **APIs & Services** in the Google Console.

1. Select **OAuth consent screen**. For user type select **External** and click the **Create** button
2. Next in the app registration form select a name that will be displayed for users when signing in to Google Account. Here you can enter whatever name you wish. You also need to provide email and developer email that will be presented to the user.
3. Next you will need to choose the OAuth scopes. Click on the **Add or remove scopes**
4. Select **auth/userinfo.email** and **auth/userinfo.profile**.
5. Next you can add a test user. Click on the **Add users** and enter an email address for your test user.

![](/images/oauth_consent_screen_combined.png "Figure 1: Configure OAuth Consent Screen")

Once the OAuth consent screen has been configured we can create an OAuth Client ID. We will need to provide two important information when we create the OAuth Client ID: **Package name** and **SHA-1 certificate fingerprint**.

##### Get Package name

For the **Package name** you must use the same name as the one configured in Unity. It is located in **File** ➡️ **Build settings...** ➡️ **Player settings** ➡️ **Other settings** ➡️ **Identification** ➡️ **Package name** , see **figure 2**.

![](/images/unity_package_name.png "Figure 2: How to get the package name of your Android Game")

##### Get SHA-1 certificate fingerprint

To obtain the **SHA-1 certificate fingerprint**, you'll need to use a tool called **keytool**, which is typically included with your Java or OpenJDK installation. In my case, I've installed OpenJDK as part of the Android Build Support module, and you can find the tool at this location `C:\Program Files\Unity\Hub\Editor\2022.3.29f1\Editor\Data\PlaybackEngines\AndroidPlayer\OpenJDK\bin\keytool.exe`. Now, recall that when we initially created the [keystore file](#create-keystore-for-signing), we’ll need to extract the SHA-1 fingerprint of the key within that keystore. To accomplish this, execute the following command:

```shell
keytool -keystore <path-to-keystore> -list -v
```

Replace `<path-to-keystore>` above with the location where you stored the keystore file. The result should look like **figure 3**.

![](/images/unity_sha1_fingerprint.png "Figure 3: Extract SHA-1 fingerprint from keystore")

Copy the SHA-1 fingerprint from the result we will need it in the next step when creating the OAuth Client ID.

##### Create OAuth Client ID

Back to Google Cloud Console navigate to **APIs & Services**.

1. Select **Credentials** menu and then choose **Create Credentials** ➡️ **OAuth Client ID**.
2. Under **Application type** choose **Android**. Enter a name for the client id, this can be anything. It is important to enable the **Custom URI scheme** so deep linking will work. We will get back to this in more details in the upcoming parts of this series. The **SHA-1 certificate fingerprint** and **Package name** can be found at [Get Package name](#get-package-name) and [Get SHA-1 certificate fingerprint](#get-sha-1-certificate-fingerprint).

![](/images/oauth_client_id_combined.png "Figure 4: Create OAuth Client ID")

This will create a new OAuth Client ID that we can use in our Android Game. Now if we wanted to test our Android Game inside the Unity Editor we would also need to create another OAuth Client ID of type **Web application**, this will make it much easier to test the authentication process before installing the Android Game on a real phone. To do this repeat the above step but instead of selecting **Android** for **Application Type** choose **Web application** 

1. Select **Credentials** menu and then choose **Create Credentials** ➡️ **OAuth Client ID**.
2. Under **Application type**, choose **Web application**. Enter a name for the client ID; it can be anything you like. Additionally, provide a redirect URI using this address: `http://127.0.0.1:5229/code?`. The redirect URI is the address where the OAuth flow will redirect once it completes the authentication process. We've specified a localhost address here because in the upcoming parts of this series, we'll set up a local server listening on localhost. It will all make sense when we get to the C# code.

![](/images/oauth_web_app.png "Figure 5: Create OAuth Client ID for web application to be used for testing inside Unity Editor")

Now we have two OAuth Client IDs: one for use on an Android phone and the other for use inside the Unity Editor.

We will need to obtain the **client id** and **client secret** of the OAuth Client for our web application, this can be done by clicking on the name of the OAuth Client ID we've created for our web application and copy the **client id** and **client secret**, see **figure 6**.

![](/images/oauth_client_id_secret.png "Figure 6: Copy the client id and client secret")

Do the same for the OAuth Client ID for our Android application, but this time only copy the **client id**. These client ids and secret are needed when we get to the part where we setup Unity to sign in to the Google Account.

## Create Firebase project and register Unity application

We’ll create a project in Firebase, register a new Android application, and then enable the Google Sign-In provider for this Android app. Let's get started.

A Firebase project is in reality just a Google Cloud Project behind the scene. If you have not already a Firebase project go ahead and create one. 

Once it's created we will register a new Unity application:

1. Check the **Register as Android app**
2. Enter a package name this **must not be the same** one as the one in your Unity project. Click on **Register app**
3. Click on the link to download the configuration file **google-services.json**
4. Click on the link to download the **Firebase SDK**

![](/images/firebase_register_app_combined.png "Figure 7: Register new Unity application")

##### Setup Google sign in provider

Now that we have registered our Unity application in Firebase we will proceed to configure the Authentication service in Firebase to allow players to sign in using Google Accounts.

1. Select **Authentication** service from Firebase products.
2. Navigate to **Sign-in method** tab
3. Click on the **Enable** switch. Enter your developer email. Click **save** button. Once it is saved, we will need to expand **Web SDK configuration** section and overwrite the **Web client ID** and **Web client secret** with the values we have created in our Google OAuth Client ID for web application. See [here](#create-oauth-client-id).
4. Go back to your firebase project settings and enter the **SHA-1 certificate fingerprint** you have extracted earlier [here](#get-sha-1-certificate-fingerprint). Then click and download the configuration file **google-services.json** again.

![](/images/firebase_authentication_combined.png "Figure 8: Setup Authentication service to use Google Sign-in")

Now that you've configured your Unity Android application in Firebase, all we need to do is add the **Firebase SDK** and include the **google-services.json** file in our Unity project.

##### Setup Firebase SDK in Unity

Extract the Firebase SDK that we have downloaded earlier and then in our Unity project do the following:

1. Right click on the **Assets** folder and select **Import package** ➡️ **Custom Package...**
2. Select the **FirebaseAuth.unitypackage** package
3. Click **Import**
4. Once imported your **Assets** should have the following folders added: **Editor Default Resources**, **ExternalDependencyManager**, **Firebase**, **Plugins**.

![](/images/unity_firebase_sdk_combined.png "Figure 9: Import Firebase SDK as custom package")

We also need to add the **google-services.json** file to our Unity project.

1. Create a new folder in Unity under **Assets**. Name the folder **StreamingAssets**
2. Copy the **google-services.json** file into the **StreamingAssets** folder.

##### Set up Gradle to build Android

With the Firebase SDK added to our game. We will need to configure the **Gradle** build system for Android.

1. Open **Project settings...** and select **Player** ➡️ **Publishing Settings**.
2. Enable the following options: **Custom Main Manifest**, **Custom Main Gradle Template**, **Custom Gradle Properties Template**, **Custom Gradle Settings Template**

![](/images/unity_custom_gradle.png "Figure 10: Enable custom Gradle templates")

Next we will force Unity to resolve all external dependencies made to our Android setup.

1. Select from menu **Assets** ➡️ **External Dependency Manager** ➡️ **Android Resolver** ➡️ **Force resolve**

![](/images/unity_android_resolve.png "Figure 11: Resolve all external dependencies")

##### Fix Firebase Loading issues

Now, this section is crucial and has caused hours of frustration. When we click play in Unity to start the game, we encounter two errors:

![](/images/unity_firebase_error.png "Figure 12: Error loading Firebase.Editor.dll")

The error that says `Assembly 'Assets/Firebase/Editor/Firebase.Editor.dll' will not be loaded due to errors` indicated that Firebase could not be loaded due to dependency on some iOS packages. Since we targeting Android we really don't care about iOS. To fix this we error we can inform Unity not to validate references for the Firebase dll.

1. Select the **Assets** ➡️ **Firebase** ➡️ **Editor** ➡️ **Firebase.Editor.dll**
2. In the inspector window **uncheck** the **Validate References**
3. Click **Apply**
4. Now run the game again the console should not show the error again.

![](/images/unity_firebase_disable_reference.png "Figure 13: Disable reference validation for the Firebase.Editor.dll")

We can also fix the other error that says:

```shell
Assembly 'Assets/ExternalDependencyManager/Editor/1.2.179/Google.IOSResolver.dll' will not be loaded due to errors:
Unable to resolve reference 'UnityEditor.iOS.Extensions.Xcode'. Is the assembly missing or incompatible with the current platform?
Reference validation can be disabled in the Plugin Inspector
```

1. Select the **Assets** ➡️ **ExternalDepdencyManager** ➡️ **Editor** ➡️ **1.2.179** ➡️ **Google.IOSResolver.dll**
2. In the inspector window choose **Platform settings** ➡️ **macOS**
3. Click **Apply**

![](/images/unity_google_ios_fixed.png "Figure 14: Fixed Google.IOSResolver.dll to only load if we are on iOS")

## Conclusion

In this segment of our series, we’ve covered the process of creating Google OAuth credentials that allow our game to sign in using Google Accounts. Additionally, we’ve established a new Firebase project and registered our Unity game. Finally, we’ve integrated the Firebase SDK into our Unity game and configured Unity to build the entire game with the Firebase SDK.

This concludes Part 2 of our series. In Part 3, I'll guide you through integrating the **i5 Toolkit for Unity** library to handle much of the OAuth2 authentication flow. 🎮🔐