+++
date = "2024-05-25T10:00:00-00:00"
title = "How to add Social Log In for Android Game using Unity and Firebase: Part 3"
image = "/images/unity_firebase.jpg"

+++

Welcome to the third installment of our series, where we'll guide you through adding social login functionality to your Unity game on Android. We’ll demonstrate how to build a simple Android application in Unity and integrate login using Google accounts, as well as manage users in Firebase. I will share with you the good, the bad and the ugly part of what I've learned during the process.

## What we will cover in Part 3

In [part2]({{< ref "unity_firebase_part2" >}}) we covered how to create Google OAuth Client IDs to enable sign-in using Google Account. We’ve set up a Firebase project and enabled the authentication service to link with our Google Provider. In this part of the series, we’ll delve into using the [i5 Toolkit for Unity](https://rwth-acis.github.io/i5-Toolkit-for-Unity/) to manage the OAuth process directly within our Unity game.

- [Download and install i5 Toolkit for Unity](#download-and-install-i5-toolkit-for-unity)
- [Create OpenID Connect Client data objects](#create-openid-connect-client-data-objects)
- [Registering OpenID Connect service](#registering-openid-connect-service)
    - [Service Core System](#service-core-system)
    - [Bootstrapping](#bootstrapping)
    - [Reference OpenID Client configurations](#reference-openid-client-configurations)
    - [Create GameObject and associated our script](#create-gameobject-and-associated-our-script)
    - [RedirectURI and deep linking](#redirecturi-and-deep-linking)
- [Handling User Authentication with Google Account and Firebase](#handling-user-authentication-with-google-account-and-firebase)

## Download and install i5 Toolkit for Unity

The **i5 Toolkit for Unity** encompasses a variety of features that can be leveraged in Unity projects. Of particular interest to us is the **OpenID Connect Client** feature, which facilitates seamless connections to any OpenID Connect Provider, including Google Accounts. The OpenID Connect client supports both Android and iOS platform. Let's get started.

First we need to download the [i5 Toolkit for Unity](https://github.com/rwth-acis/i5-Toolkit-for-Unity/releases) as of this writing the latest version is 1.9.2. Once downloaded we will need to import the package in our Unity project

1. Right click **Assets** and select **Import Package** ➡️ **Custom Package...**
2. Click on **Import**

You might get prompted with a pop up that asks you to update some of the files, just click yes to that.

## Create OpenID Connect Client data objects

We’ll set up configurations for our OpenID Connect Client. These configurations will include the **client ID** and **client secret** associated with our Google OAuth Client IDs.

1. Right click **Assets** and select **Create** ➡️ **i5 Toolkit** ➡️ **OpenID Connect Client Data**
2. Name it **GameClient**
3. In the inspector window for the **GameClient** object enter the **client id** of the Google OAuth Client that we have created in [part2]({{< ref "unity_firebase_part2#create-oauth-client-id" >}}) for Android application.

![](/images/unity_i5_gameclient.png "Figure 1: Set client id for Android application")

Now repeat the same but this time we want to set the configuration for our web application. This will be used whenever we run the game inside Unity editor.

1. Right click **Assets** and select **Create** ➡️ **i5 Toolkit** ➡️ **OpenID Connect Client Data**
2. Name it **GameClientEditor**
3. In the inspector window for the **GameClientEditor** object enter the **client id** and **client secret** of the Google OAuth Client that we have created in [part2]({{< ref "unity_firebase_part2#create-oauth-client-id" >}}) for Web application.

We’ve set up two OpenID Connect Client data configurations that point to our Android and web application OAuth Client IDs. Now it is time to write some C# code.

## Registering OpenID Connect service

Let’s write a C# script to manage the sign-in process when a player clicks the **Sign In button** in our game.

1. Right click on **Assets/Projects/Scripts** folder and select **Create** ➡️ **C# Script**
2. Name the script **GoogleSignIn**
3. Double click on the script and it will open the file in the associated code editor, mine is Visual Studio Code.

The following C# script, **GoogleSignIn.cs**, was generated:

{{< collapsible csharp>}}
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class GoogleSignIn : MonoBehaviour
{
    // Start is called before the first frame update
    void Start()
    {
        
    }

    // Update is called once per frame
    void Update()
    {
        
    }
}
{{< /collapsible >}}
\
The above code illustrates the basic skeleton for any C# script that can be added to a game object. All C# scripts must implement a class derived from the **MonoBehaviour** class. 🚀

##### Service Core System

At the heart of the i5 toolkit service system lies the ability to incorporate global functionality into our scenes without relying on GameObjects or classes derived from MonoBehaviour. Essentially, we can create native C# classes that act as singletons and register them with the service system. When we require access to these services, we can retrieve them through the service system, ensuring their global accessibility throughout our code.

##### Bootstrapping

The i5 Toolkit offers a bootstrapping mechanism that simplifies the registration of services with the service system. To achieve this, we create a C# class that inherits from the `BaseServiceBootstrapper` class instead of `MonoBehaviour`. Essentially, the BaseServiceBootstrapper itself inherits from MonoBehaviour. Below is the code snippet for the `BaseServiceBootstrapper` class.

{{< collapsible csharp>}}
public abstract class BaseServiceBootstrapper : MonoBehaviour
{
    protected virtual void Start()
    {
        RegisterServices();
    }

    protected virtual void OnDestroy()
    {
        UnRegisterServices();
    }

    protected abstract void RegisterServices();

    protected abstract void UnRegisterServices();
}
{{< /collapsible >}}
\
So the **crucial aspect** is that any derived classes **must** implement the `RegisterServices` and `UnregisterServices` methods.

##### Reference OpenID Client configurations

If you recall we've created the [OpenID Connect client configuration](#create-openid-connect-client-data-objects), now we will need to reference those settings inside our code. Let's update our `GoogleSignIn` class to use these settings.

{{< collapsible csharp>}}
using i5.Toolkit.Core.OpenIDConnectClient;
using i5.Toolkit.Core.ServiceCore;
using UnityEngine;

public class GoogleSignIn : BaseServiceBootstrapper
{
    // reference the GameClient configuration we have created in the Unity editor
    [SerializeField] private ClientDataObject gameClient;
    // reference the GameClientEditor configuration we have created in the Unity editor
    [SerializeField] private ClientDataObject gameClientEditor;

    protected override void RegisterServices()
    {
        throw new System.NotImplementedException();
    }

    protected override void UnRegisterServices()
    {
        throw new System.NotImplementedException();
    }

    // Update is called once per frame
    void Update()
    {
        
    }
}
{{< /collapsible >}}
\
Notice also how we have changed our class to derive from **`BaseServiceBootstrapper`** instead of `MonoBehaviour`. This change is essential for bootstrapping our OpenID Connect Client service later on.

The two fields **`gameClient`** and  **`gameClientEditor`** are references to the OpenID Connect client configurations we have created previously in our Unity Editor.

##### Create GameObject and associated our script

Now that we have a **fundamental C# class** that we can utilize to reference OpenID Client configuration, let's proceed by creating a **GameObject** and associating the code with it.

1. Right click on the **SampleScene** and select **Create empty** and name the GameObejct **GoogleSignIn**
2. Drag and Drop the `GoogleSignIn.cs` script from your **Project** ➡️ **Scripts** folder into the GameObject **GoogleSignIn** inspector window

![](/images/unity_signin_gameobject_combined.png "Figure 2: Create new empty GameObject and attach our code to it")

As we can see our new GameObject now exposes two fields in the inspector window: **Game Client** and **Game Client Editor**. Those two fields are the fields that we have in our C# code. We can now drag and drop our OpenID Connect configurations onto those two fields.

![](/images/unity_signin_gameclients.png "Figure 3: Drag and drop our OpenID Connect Client objects onto the two Game Client and Game Client Editor fields")

Now we need to create our **OpenID Connect service** and register it with the service system. Let's update our code:

{{< collapsible csharp>}}
using i5.Toolkit.Core.OpenIDConnectClient;
using i5.Toolkit.Core.ServiceCore;
using UnityEngine;

public class GoogleSignIn : BaseServiceBootstrapper
{
    [SerializeField] private ClientDataObject gameClient;
    [SerializeField] private ClientDataObject gameClientEditor;

    protected override void RegisterServices()
    {
        OpenIDConnectService oidc = new OpenIDConnectService();
        oidc.OidcProvider = new GoogleOidcProvider();
        #if !UNITY_EDITOR
        oidc.OidcProvider.ClientData = gameClient.clientData;
        oidc.RedirectURI = "com.defaultcompany.sociallogin:/";
        #else
        oidc.OidcProvider.ClientData = gameClientEditor.clientData;
        oidc.ServerListener.ListeningUri = "http://127.0.0.1:5229/";
        #endif
        ServiceManager.RegisterService(oidc);
    }

    protected override void UnRegisterServices()
    {
        throw new System.NotImplementedException();
    }

    // Update is called once per frame
    void Update()
    {
        
    }
}
{{< /collapsible >}}
\
We've completed the implementation of the **`RegisterServices`** methods. Within this method, we've instantiated an instance of **`OpenIDConnectService`** and configured it to use the Google OpenID Connect Provider.

Now, if we are to run our game inside the Unity Editor, we’d like to test our sign-in process. To achieve this, we’ll utilize the OpenID Connect client configuration that we’ve set up for our Google OAuth Client ID for the web application. If we instead are to run the Game on an Android phone we will instead use the configuration for our Google OAuth Client ID for the android application. If you can recall from [part2]({{< ref "unity_firebase_part2#create-oauth-client-id" >}}) of our series, we had already created the Client IDs for both a web and android application.

##### RedirectURI and deep linking

The **RedirectURI** is crucial because it's the URI that the OAuth flow redirects to after completing its authentication process. When running inside the Unity Editor, we spin up a server on localhost to handle requests for the redirect URI. The local server listens on port `5229`. If you recall from [part2]({{< ref "unity_firebase_part2#create-oauth-client-id" >}}) of our series where we created a Google OAuth Client ID for a web application, we've added a redirect URI pointing to `http://127.0.0.1:5229/code?`. After the OAuth flow completes its authentication process it will redirect to the uri `http://127.0.0.1:5229/code?` and this has to match the one we specified when we created the Google OAuth Client ID for our web application.

Now, when running from our Android device, the redirect URI is a bit different. Since we want the OAuth flow to redirect us back to our Android game, we can't use a normal HTTP scheme because there's no HTTP server listening for the redirect. Instead, we'll utilize **deep linking**.

In a nutshell **deep linking is the use of custom URL schemes to direct users to specific pages within an app**.

![](/images/deep_link.png "Figure 4: App opens a web browser and user authenticates against OpenID Connect Provider, then browser redirects back to App using a custom url scheme")

---

In the provided code snippet, we've configured a custom redirect URI using a URL scheme in the format `com.defaultcompany.sociallogin:/`. Let's break it down:

1. **URL Scheme:**
   - The first part of the URL is the **scheme name**, which in this case is `com.defaultcompany.sociallogin`.
   - A scheme is a way to uniquely identify an app or service. It allows other apps or systems to communicate with our app.
   - For example, if another app wants to open our app, it can use this custom scheme to do so.

2. **Path Within the Application:**
   - The second part of the URL is the **path**, which specifies the location within our app.
   - In this case, the path is the **root path**, indicated by `/`.
   - The root path typically corresponds to the main screen or entry point of our app.

3. **Deep Linking:**
   - To make deep linking work correctly, we'll need to update the **`AndroidManifest.xml`** file.
   - Deep linking allows users to open specific screens or content within our app directly from external sources (e.g., a link in an email or a web page).
   - By defining the custom scheme and path in the manifest, we enable our app to handle incoming requests with that URL.

---

Go ahead and update the **Assets/Plugins/Android/AndroidManifest.xml** file witht the following code:

{{< collapsible xml>}}
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" xmlns:tools="http://schemas.android.com/tools">
  <application>
    <activity android:name="com.unity3d.player.UnityPlayerActivity" android:theme="@style/UnityThemeSelector">
      <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>
      <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="com.defaultcompany.sociallogin" />
      </intent-filter>
    </activity>
  </application>
</manifest>
{{< /collapsible >}}
\
Certainly! Here's an improved version of your important note:


**🚨 IMPORTANT: Pay Attention to URL Scheme Case! 🚨**

When defining a custom URL scheme, **ensure that it is in lowercase**. For example, use `com.defaultcompany.sociallogin` instead of any uppercase variations. Why is this crucial? Let me break it down:

1. **Debugging Headaches Avoided:**
   - If you accidentally use uppercase letters in the scheme (e.g., `com.DefaultCompany.SocialLogin`), you might encounter issues during deep linking.
   - Debugging such issues can be time-consuming and frustrating. Save yourself the headache by sticking to lowercase!


## Handling User Authentication with Google Account and Firebase

Now that we've successfully implemented the code for registering our OpenID Connect service, let's focus on the next crucial step: handling user sign-in. Here's how we'll proceed:

1. **Redirect to Google Account Sign-In Page:**
   - When a player initiates sign-in, we'll direct them to the Google Account sign-in page.
   - This page allows users to authenticate using their Google credentials.

2. **Authorization via Firebase:**
   - Once the user signs in through Google, we'll use Firebase for authorization.
   - Firebase provides robust authentication services, allowing us to securely manage user sessions and permissions.


Let's create a new C# script in our **Scripts** folder and name it **`AuthManager.cs`**. The code should look like the following:

{{< collapsible csharp>}}
using UnityEngine;
using Firebase.Auth;
using i5.Toolkit.Core.ServiceCore;
using i5.Toolkit.Core.OpenIDConnectClient;
using System;
using System.Threading.Tasks;

public class AuthManager : MonoBehaviour
{
    private FirebaseAuth auth;

    // Start is called before the first frame update
    void Start()
    {
        auth = FirebaseAuth.DefaultInstance;
    }

    public async void SignIn()
    {
        Debug.Log("Sign in called");
        await SignInWithGoogle();
    }

    private async Task SignInWithGoogle()
    {
        ServiceManager.GetService<OpenIDConnectService>().LoginCompleted += OnLoginCompleted;

        await ServiceManager.GetService<OpenIDConnectService>().OpenLoginPageAsync();
    }

    private void OnLoginCompleted(object sender, EventArgs e)
    {
        Debug.Log("Login completed");
        string accessToken = ServiceManager.GetService<OpenIDConnectService>().AccessToken;
        Debug.Log("Access token: " + accessToken);
        Firebase.Auth.Credential credential =
    Firebase.Auth.GoogleAuthProvider.GetCredential(null, accessToken);
        auth.SignInAndRetrieveDataWithCredentialAsync(credential).ContinueWith(task =>
        {
            if (task.IsCanceled)
            {
                Debug.LogError("SignInAndRetrieveDataWithCredentialAsync was canceled.");
                return;
            }
            if (task.IsFaulted)
            {
                Debug.LogError("SignInAndRetrieveDataWithCredentialAsync encountered an error: " + task.Exception);
                return;
            }

            Firebase.Auth.AuthResult result = task.Result;
            Debug.LogFormat("User signed in successfully: {0} ({1})",
                result.User.DisplayName, result.User.UserId);
        });
    }
}
{{< /collapsible >}}
\
Let's break down the code:

```csharp
private FirebaseAuth auth;

// Start is called before the first frame update
void Start()
{
    auth = FirebaseAuth.DefaultInstance;
}
```

We get a reference to our Firebase authentication instance that we will be using later in the code.

```csharp
public async void SignIn()
{
    Debug.Log("Sign in called");
    await SignInWithGoogle();
}
```

We created a method named `SignIn` that will be triggered when we press the sign-in button in our game. This method will call another method `SignInWithGoogle`.

```csharp
private async Task SignInWithGoogle()
{
    ServiceManager.GetService<OpenIDConnectService>().LoginCompleted += OnLoginCompleted;

    await ServiceManager.GetService<OpenIDConnectService>().OpenLoginPageAsync();
}
```

The **`SignInWithGoogle`** method plays a crucial role in our authentication flow:

1. **Google Account Sign-In Page:**
   - When invoked, this method presents the user with the Google Account sign-in page.
   - The default browser on the Android device opens this page, allowing users to authenticate using their Google credentials.

2. **Login Completed Event:**
   - We've also registered the **`LoginCompleted`** event.
   - This event fires whenever the user successfully signs in.
   - It allows us to handle post-login actions, such as updating UI elements or managing user sessions.


```csharp
private void OnLoginCompleted(object sender, EventArgs e)
{
    Debug.Log("Login completed");
    string accessToken = ServiceManager.GetService<OpenIDConnectService>().AccessToken;
    Debug.Log("Access token: " + accessToken);
    Firebase.Auth.Credential credential =
    Firebase.Auth.GoogleAuthProvider.GetCredential(null, accessToken);
    auth.SignInAndRetrieveDataWithCredentialAsync(credential).ContinueWith(task =>
    {
        if (task.IsCanceled)
        {
            Debug.LogError("SignInAndRetrieveDataWithCredentialAsync was canceled.");
            return;
        }
        if (task.IsFaulted)
        {
            Debug.LogError("SignInAndRetrieveDataWithCredentialAsync encountered an error: " + task.Exception);
            return;
        }

        Firebase.Auth.AuthResult result = task.Result;
        Debug.LogFormat("User signed in successfully: {0} ({1})",
            result.User.DisplayName, result.User.UserId);
    });
}
```

The **`OnLoginCompleted`** method triggers when a user successfully signs in. Within this method, we utilize the Firebase SDK to perform the following tasks:

1. **Sign-In with Firebase:**
   - We use the Firebase authentication service to sign in the user.
   - Firebase handles secure authentication, ensuring that the user's credentials are verified.

2. **Retrieve User Data:**
   - Once authenticated, we retrieve relevant user data.
   - This data might include profile information, access tokens, or other relevant details.


Now that we have the script ready, we will need to assign the `SignIn` method to our UI button.

1. Create a new empty **GameObject** and name it **AuthManager**
2. Drag the **AuthManager.cs** script into the **AuthManager** GameObject.
3. Click on the **Button** object
4. In the inspector window find the **On Click ()** component and press the **+** button
5. Drag and drop the **AuthManager** GameObject onto the **On Click ()** component and choose the function **SignIn**

![](/images/unity_button_click.png "Figure 5: Attach SignIn method to the button click event")