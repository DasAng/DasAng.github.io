+++
date = "2024-05-23T10:00:00-00:00"
title = "How to add Social Log In for Android Game using Unity and Firebase: Part 1"
image = "/images/unity_firebase.jpg"

+++

Welcome to the first installment of our series, where we'll guide you through adding social login functionality to your Unity game on Android. We’ll demonstrate how to build a simple Android application in Unity and integrate login using Google accounts, as well as manage users in Firebase. I will share with you the good, the bad and the ugly part of what I've learned during the process.

## Background

In an Unity Android game project I was working on, I needed to provide players with a way to log in using their Google accounts. The game’s backend API was on Google Cloud and only allows authorized calls using the player’s identity. I required a method to seamlessly integrate login functionality directly within the game, eliminating the need for users to register on external websites.

There were several options to enable third-party social login, including the following providers:

- **Auth0**
- **AWS Cognito**
- **Firebase**
- **Unity Authentication**
- **Google Sign-In Unity Plugin**

### Unity Authentication

Unity's own [authentication gaming service](https://docs.unity.com/ugs/en-us/manual/authentication/manual/overview) seems like a good fit and supports a wide range of authentication providers and it integrates seamlessly with Unity.

- Google Play Games
- Unity Player Accounts
- Google
- Apple
- Facebook
- Steam
- OpenID
- Custom ID

To authenticate using Google Account you would have to use the [Google Play Games plugin](https://docs.unity.com/ugs/en-us/manual/authentication/manual/platform-signin-google) and only support v0.10.14 or earlier of the plugin. After spending hours attempting to make this work in Unity with version v0.10.14, I successfully signed in with my Google Account. However, obtaining the ID token from Google proved to be a challenge.

Additionally, I encountered issues due to outdated documentation for v0.10.14, where several methods were no longer supported. Ultimately, I opted for a less convoluted approach.

### Google Sign-In Unity Plugin

Another solution was the [Google Sign-In Unity Plugin](https://github.com/googlesamples/google-signin-unity). This plugin supports authentication with Google Accounts through OAuth and is compatible with both Android and iOS. However, please note that this plugin is quite old—almost 6 years old as of this writing. When I attempted to import this plugin into my Unity project, which is running version 2022, I discovered that it is no longer supported.

### AWS Cognito

The AWS Cognito service offers third-party integration with Google Accounts see [here](https://docs.aws.amazon.com/cognito/latest/developerguide/google.html). While it also supports Unity, the challenge lies in the similarity to the **Unity Authentication** approach. Specifically, I would need to utilize the [Google Play Games plugin](https://docs.unity.com/ugs/en-us/manual/authentication/manual/platform-signin-google) version 0.10.14, which is both convoluted and outdated.


### Auth0

Auth0 is a popular SaaS provider that offers a wide range of support for OAuth authentication. However, the challenge lies in the lack of readily available resources and documentation for integrating it with Unity. To achieve this integration, you would need to write a significant amount of custom code.

### Firebase

Firebase has an Authentication service that closely resembles **AWS Cognito User Pool** and allows seamless third-party integration with Google Account. The Firebase SDK also [supports](https://firebase.google.com/docs/auth/unity/google-signin) Unity. The challenge with Firebase is that you would still need to use a way to integrate OAuth2 into your Unity game through custom code or library.

## Final approach

After dedicating countless hours and days to researching and implementing various providers, I’ve ultimately discovered a solution that perfectly aligns with my requirements. I've decided to go with **Firebase** using their Unity plugin and combine it with another Unity library called [i5 Toolkit](https://rwth-acis.github.io/i5-Toolkit-for-Unity/1.9.2/index.html) that supports OpenID connect provider. Youtubes like [Authenticate users using google in Unity and Firebase with OAuth2.0](https://www.youtube.com/watch?v=OCCe1TXDEq8) and reading the documentation has helped me a lot in implementing it myself.

![](/images/unity_firebase_architecture.png "Figure 1: Sign in players with google account using i5 toolkit and get JWT token from Firebase")

## Step-by-step guide

I will guide you through implementing social login for a Unity Android game, allowing users to log in using their Google accounts. We will demonstrate how to integrate [Firebase](https://firebase.google.com/docs/auth/unity/google-signin) SDK and [i5 Toolkit](https://rwth-acis.github.io/i5-Toolkit-for-Unity/1.9.2/index.html) to facilitate the authentication process flow and retrieve the JWT token that can be used to authorize calls to Google cloud API Gateway.

To follow this guide you will need to have the following:

- **Unity version 2022**
- **Visual Studio Code**
- **Android phone (I've tested on Galaxy Z Fold, but any model should be fine)**

## What we will cover in Part 1

In this part of the series we will be covering the following steps:

- [Unity setup](#unity-setup)
    - [Install Android Build Support module](#install-android-build-support-module)
    - [Create new project](#create-new-project)
- [Setup Project structure in Unity](#setup-project-structure-in-unity)
- [Create sign in UI in Unity](#create-sign-in-ui-in-unity)
- [Setup Android Build](#setup-android-build)
    - [Create keystore for signing](#create-keystore-for-signing)
    - [Set Android API Level](#set-android-api-level)

## Unity setup

If you haven’t already, download and install Unity. Make sure to install version 2022.

##### Install Android Build Support module

To enable Android builds, you’ll need to install the **Android Build Support** module. Navigate to the *Installs* menu in your **Unity Hub**, and click on the gear icon next to your Unity version and choose , as depicted in figure 2.

![](/images/unity_android_build_support.png "Figure 2: Install Android Build Support module")

This will open a new window. Put a check mark on the *Android Build Support* ensure that both *OpenJDK* and *Android SDK & NDK Tools* are checked and click *Install* button, see figure 3. In my case I've already installed the tools so the checkbox are grayed out.

![](/images/unity_android_build_support2.png "Figure 3: Install Android Build Support module")

##### Create new project

Next, create a new project named *SocialLogin* and select the 3D Core template.

![](/images/unity_create_project.png "Figure 4: Create a new 3D Core project")

A new Unity Editor will start and load your project, see figure 5.

![](/images/unity_scene.png "Figure 5: Unity Editor")

## Setup Project structure in Unity

Now that we’ve set up a new Unity project, let’s organize it. We’ll create a fresh folder called **Projects** to keep all our custom assets and C# code.
Certainly! Here's an improved version of the instructions:

1. Right-click on the **Assets** folder in your Unity project window.
2. Select **Create** ➡️ **Folder**.
3. Name the new folder **Projects**.
4. Create another folder named **Scripts**.
5. Drag and drop the existing **Scenes** and the new **Scripts** folder into the newly created **Projects** folder.

![](/images/unity_project_folder_combined.png "Figure 6: Organize folders inside Projects folder")

Now that we’ve established the fundamental project structure, our next step is to create a user interface for signing in to a Google account.

## Create sign in UI in Unity

In this step we will be creating a user interface for signing in to a Google account.

1. Right-click on the **SampleScene** in the **Hierarchy** window.
2. Choose **UI** ➡️ **Canvas**.

![](/images/unity_ui_canvas_combined.png "Figure 7: Create new UI canvas")

As shown in **figure 7**, you now have two additional game objects added to the scene: **Canvas** and **EventSystem**.

Next, let's create a sign-in button

1. Right-click on the **Canvas** object and choose **UI** ➡️ **Button - TextMeshPro**.
2. A new popup window will appear; click on **Import TMP Essentials**.
3. A new **Button** object is added to the **Canvas**
4. Assets for the **TextMeshPro** is added to your **Assets** folder.

![](/images/unity_new_button_combined.png "Figure 8: Create new sign in button")

Next, let’s configure the **Canvas**. We want the UI to scale according to the screen size, so if the screen is smaller, the UI should scale proportionally. We also want to set the width and height of the sign in button to something larger.

1. Click on the **Canvas** object, and in the inspector window, select **UI Scale Mode** ➡️ **Scale With Screen Size**.
2. Click on the **Button** object, and in the inspector window choose the **Rect Transform** component and set the **Width** ➡️ 320 and **Height** ➡️ 60.
3. Click on the **Button** ➡️ **Text** object, and in the inspector window choose the **TextMeshPro - Text (UI)** component and set the **Text Input** ➡️ Sign In.

![](/images/unity_signin_button_ui.png "Figure 9: Adjust scale size and set button text and dimension")

Now if you select the **Game** tab in your 3D Scene window you should see a grey button with the text *Sign In*, see figure 10.

![](/images/unity_game_scene.png "Figure 10: Game scene with the Sign In button")

## Setup Android Build

In this step, we will configure Unity to build our scene for an Android app.

1. Choose **File** ➡️ **Build Settings...** from the menu
2. Then click on the **Android** platform and click on the **Swtich Platform** button.

![](/images/unity_android_build_settings.png "Figure 11: Switch to Andoird Platform")

Now your project is ready to be built for Android. 

##### Create keystore for signing

Android requires that all apps be digitally signed with a certificate before they can be installed. So let's configure Unity to create a keystore for signing our Android game.

1. Choose **File** ➡️ **Build Settings...** from the menu
2. Click on the **Player Settings...**
3. Expand the **Publishing Settings**
4. Click on **Keystore Manager...** button
5. Enter a password for the keystore
6. Give the key an alias
7. Enter password for the key
8. Click the **Add key** button

![](/images/unity_keystore_combined.png "Figure 12: Create a new keystore")

A new keystore will be created in the location specified. The keystore necessitates that our game be built with native 64-bit binary support. Consequently, we must configure the following settings.:

1. Still in **Player settings** expand the **Other settings**
2. Set **Configuration** ➡️ **Scripting Backend** ➡️ **IL2CPP**
3. Put a checkmark on **Configuration** ➡️ **Target Architectures** ➡️ **ARM64**

![](/images/unity_64bit.png "Figure 13: Configure 64-bit build support")

##### Set Android API Level

We will be importing packages that require a minimum Android API level of 23 and a target level of 34. This requires configuring the following:

1. In **Player settings** expand the **Other settings**
2. Set **Identification** ➡️ **Minimum API Level** ➡️ **API Level 23**
3. Set **Identification** ➡️ **Target API Level** ➡️ **API Level 34**

![](/images/unity_target_level.png "Figure 14: Set API Level")

## Conclusion

In this part of our series, we’ve covered how to configure Unity for building our Android game. Additionally, we’ve created a user interface (UI) that allows players to sign in using their Google accounts.

This concludes part 1 of our series. In [part2]({{< ref "unity_firebase_part2" >}}), I’ll guide you through creating a Google OAuth Client ID that our game can use for enabling sign-in with Google Accounts.
