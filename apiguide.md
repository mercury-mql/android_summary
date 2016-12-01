#API GUIDE —— System Permission

Android系统是权限分离的操作系统，每一个APP都有一个唯一的系统标识（Linux系统的uid和gid）。



Android is a privilege-separated operating system, in which each application runs with a distinct system identity (Linux user ID and group ID). Parts of the system are also separated into distinct identities. Linux thereby isolates applications from each other and from the system.

Additional finer-grained security features are provided through a "permission" mechanism that enforces restrictions on the specific operations that a particular process can perform, and per-URI permissions for granting ad hoc access to specific pieces of data.

This document describes how application developers can use the security features provided by Android. A more general Android Security Overview is provided in the Android Open Source Project.

Security Architecture
A central design point of the Android security architecture is that no application, by default, has permission to perform any operations that would adversely impact other applications, the operating system, or the user. This includes reading or writing the user's private data (such as contacts or emails), reading or writing another application's files, performing network access, keeping the device awake, and so on.

Because each Android application operates in a process sandbox, applications must explicitly share resources and data. They do this by declaring the permissions they need for additional capabilities not provided by the basic sandbox. Applications statically declare the permissions they require, and the Android system prompts the user for consent at the time the application is installed.

The application sandbox does not depend on the technology used to build an application. In particular the Dalvik VM is not a security boundary, and any app can run native code (see the Android NDK). All types of applications — Java, native, and hybrid — are sandboxed in the same way and have the same degree of security from each other.

Application Signing
All APKs (.apk files) must be signed with a certificate whose private key is held by their developer. This certificate identifies the author of the application. The certificate does not need to be signed by a certificate authority; it is perfectly allowable, and typical, for Android applications to use self-signed certificates. The purpose of certificates in Android is to distinguish application authors. This allows the system to grant or deny applications access to signature-level permissions and to grant or deny an application's request to be given the same Linux identity as another application.

User IDs and File Access
At install time, Android gives each package a distinct Linux user ID. The identity remains constant for the duration of the package's life on that device. On a different device, the same package may have a different UID; what matters is that each package has a distinct UID on a given device.

Because security enforcement happens at the process level, the code of any two packages cannot normally run in the same process, since they need to run as different Linux users. You can use the sharedUserId attribute in the AndroidManifest.xml's manifest tag of each package to have them assigned the same user ID. By doing this, for purposes of security the two packages are then treated as being the same application, with the same user ID and file permissions. Note that in order to retain security, only two applications signed with the same signature (and requesting the same sharedUserId) will be given the same user ID.

Any data stored by an application will be assigned that application's user ID, and not normally accessible to other packages. When creating a new file with getSharedPreferences(String, int), openFileOutput(String, int), or openOrCreateDatabase(String, int, SQLiteDatabase.CursorFactory), you can use the MODE_WORLD_READABLE and/or MODE_WORLD_WRITEABLE flags to allow any other package to read/write the file. When setting these flags, the file is still owned by your application, but its global read and/or write permissions have been set appropriately so any other application can see it.

Using Permissions
A basic Android application has no permissions associated with it by default, meaning it cannot do anything that would adversely impact the user experience or any data on the device. To make use of protected features of the device, you must include in your AndroidManifest.xml one or more <uses-permission> tags declaring the permissions that your application needs.

For example, an application that needs to monitor incoming SMS messages would specify:

<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.android.app.myapp" >
    <uses-permission android:name="android.permission.RECEIVE_SMS" />
    ...
</manifest>
At application install time, permissions requested by the application are granted to it by the package installer, based on checks against the signatures of the applications declaring those permissions and/or interaction with the user. No checks with the user are done while an application is running; the app is either granted a particular permission when installed, and can use that feature as desired, or the permission is not granted and any attempt to use the feature fails without prompting the user.

Often times a permission failure will result in a SecurityException being thrown back to the application. However, this is not guaranteed to occur everywhere. For example, the sendBroadcast(Intent) method checks permissions as data is being delivered to each receiver, after the method call has returned, so you will not receive an exception if there are permission failures. In almost all cases, however, a permission failure will be printed to the system log.

However, in a normal user situation (such as when the app is installed from Google Play Store), an app cannot be installed if the user does not grant the app each of the requested permissions. So you generally don't need to worry about runtime failures caused by missing permissions because the mere fact that the app is installed at all means that your app has been granted its desired permissions.

The permissions provided by the Android system can be found at Manifest.permission. Any application may also define and enforce its own permissions, so this is not a comprehensive list of all possible permissions.

A particular permission may be enforced at a number of places during your program's operation:

At the time of a call into the system, to prevent an application from executing certain functions.
When starting an activity, to prevent applications from launching activities of other applications.
Both sending and receiving broadcasts, to control who can receive your broadcast or who can send a broadcast to you.
When accessing and operating on a content provider.
Binding to or starting a service.
Caution: Over time, new restrictions may be added to the platform such that, in order to use certain APIs, your app must request a permission that it previously did not need. Because existing apps assume access to those APIs is freely available, Android may apply the new permission request to the app's manifest to avoid breaking the app on the new platform version. Android makes the decision as to whether an app might need the permission based on the value provided for the targetSdkVersion attribute. If the value is lower than the version in which the permission was added, then Android adds the permission.
For example, the WRITE_EXTERNAL_STORAGE permission was added in API level 4 to restrict access to the shared storage space. If your targetSdkVersion is 3 or lower, this permission is added to your app on newer versions of Android.
Beware that if this happens to your app, your app listing on Google Play will show these required permissions even though your app might not actually require them.
To avoid this and remove the default permissions you don't need, always update your targetSdkVersion to be as high as possible. You can see which permissions were added with each release in the Build.VERSION_CODES documentation.
Declaring and Enforcing Permissions
To enforce your own permissions, you must first declare them in your AndroidManifest.xml using one or more <permission> tags.

For example, an application that wants to control who can start one of its activities could declare a permission for this operation as follows:

<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.me.app.myapp" >
    <permission android:name="com.me.app.myapp.permission.DEADLY_ACTIVITY"
        android:label="@string/permlab_deadlyActivity"
        android:description="@string/permdesc_deadlyActivity"
        android:permissionGroup="android.permission-group.COST_MONEY"
        android:protectionLevel="dangerous" />
    ...
</manifest>
The <protectionLevel> attribute is required, telling the system how the user is to be informed of applications requiring the permission, or who is allowed to hold that permission, as described in the linked documentation.

The <permissionGroup> attribute is optional, and only used to help the system display permissions to the user. You will usually want to set this to either a standard system group (listed in android.Manifest.permission_group) or in more rare cases to one defined by yourself. It is preferred to use an existing group, as this simplifies the permission UI shown to the user.

Note that both a label and description should be supplied for the permission. These are string resources that can be displayed to the user when they are viewing a list of permissions (android:label) or details on a single permission ( android:description). The label should be short, a few words describing the key piece of functionality the permission is protecting. The description should be a couple sentences describing what the permission allows a holder to do. Our convention for the description is two sentences, the first describing the permission, the second warning the user of what bad things can happen if an application is granted the permission.

Here is an example of a label and description for the CALL_PHONE permission:

    <string name="permlab_callPhone">directly call phone numbers</string>
    <string name="permdesc_callPhone">Allows the application to call
        phone numbers without your intervention. Malicious applications may
        cause unexpected calls on your phone bill. Note that this does not
        allow the application to call emergency numbers.</string>
You can look at the permissions currently defined in the system with the Settings app and the shell command adb shell pm list permissions. To use the Settings app, go to Settings > Applications. Pick an app and scroll down to see the permissions that the app uses. For developers, the adb '-s' option displays the permissions in a form similar to how the user will see them:

$ adb shell pm list permissions -s
All Permissions:
Network communication: view Wi-Fi state, create Bluetooth connections, full
Internet access, view network state
Your location: access extra location provider commands, fine (GPS) location,
mock location sources for testing, coarse (network-based) location
Services that cost you money: send SMS messages, directly call phone numbers
...
Enforcing Permissions in AndroidManifest.xml

High-level permissions restricting access to entire components of the system or application can be applied through your AndroidManifest.xml. All that this requires is including an android:permission attribute on the desired component, naming the permission that will be used to control access to it.

Activity permissions (applied to the <activity> tag) restrict who can start the associated activity. The permission is checked during Context.startActivity() and Activity.startActivityForResult(); if the caller does not have the required permission then SecurityException is thrown from the call.

Service permissions (applied to the <service> tag) restrict who can start or bind to the associated service. The permission is checked during Context.startService(), Context.stopService() and Context.bindService(); if the caller does not have the required permission then SecurityException is thrown from the call.

BroadcastReceiver permissions (applied to the <receiver> tag) restrict who can send broadcasts to the associated receiver. The permission is checked after Context.sendBroadcast() returns, as the system tries to deliver the submitted broadcast to the given receiver. As a result, a permission failure will not result in an exception being thrown back to the caller; it will just not deliver the intent. In the same way, a permission can be supplied to Context.registerReceiver() to control who can broadcast to a programmatically registered receiver. Going the other way, a permission can be supplied when calling Context.sendBroadcast() to restrict which BroadcastReceiver objects are allowed to receive the broadcast (see below).

ContentProvider permissions (applied to the <provider> tag) restrict who can access the data in a ContentProvider. (Content providers have an important additional security facility available to them called URI permissions which is described later.) Unlike the other components, there are two separate permission attributes you can set: android:readPermission restricts who can read from the provider, and android:writePermission restricts who can write to it. Note that if a provider is protected with both a read and write permission, holding only the write permission does not mean you can read from a provider. The permissions are checked when you first retrieve a provider (if you don't have either permission, a SecurityException will be thrown), and as you perform operations on the provider. Using ContentResolver.query() requires holding the read permission; using ContentResolver.insert(), ContentResolver.update(), ContentResolver.delete() requires the write permission. In all of these cases, not holding the required permission results in a SecurityException being thrown from the call.

Enforcing Permissions when Sending Broadcasts

In addition to the permission enforcing who can send Intents to a registered BroadcastReceiver (as described above), you can also specify a required permission when sending a broadcast. By calling Context.sendBroadcast() with a permission string, you require that a receiver's application must hold that permission in order to receive your broadcast.

Note that both a receiver and a broadcaster can require a permission. When this happens, both permission checks must pass for the Intent to be delivered to the associated target.

Other Permission Enforcement

Arbitrarily fine-grained permissions can be enforced at any call into a service. This is accomplished with the Context.checkCallingPermission() method. Call with a desired permission string and it will return an integer indicating whether that permission has been granted to the current calling process. Note that this can only be used when you are executing a call coming in from another process, usually through an IDL interface published from a service or in some other way given to another process.

There are a number of other useful ways to check permissions. If you have the pid of another process, you can use the Context method Context.checkPermission(String, int, int) to check a permission against that pid. If you have the package name of another application, you can use the direct PackageManager method PackageManager.checkPermission(String, String) to find out whether that particular package has been granted a specific permission.

URI Permissions
The standard permission system described so far is often not sufficient when used with content providers. A content provider may want to protect itself with read and write permissions, while its direct clients also need to hand specific URIs to other applications for them to operate on. A typical example is attachments in a mail application. Access to the mail should be protected by permissions, since this is sensitive user data. However, if a URI to an image attachment is given to an image viewer, that image viewer will not have permission to open the attachment since it has no reason to hold a permission to access all e-mail.

The solution to this problem is per-URI permissions: when starting an activity or returning a result to an activity, the caller can set Intent.FLAG_GRANT_READ_URI_PERMISSION and/or Intent.FLAG_GRANT_WRITE_URI_PERMISSION. This grants the receiving activity permission access the specific data URI in the Intent, regardless of whether it has any permission to access data in the content provider corresponding to the Intent.

This mechanism allows a common capability-style model where user interaction (opening an attachment, selecting a contact from a list, etc) drives ad-hoc granting of fine-grained permission. This can be a key facility for reducing the permissions needed by applications to only those directly related to their behavior.

The granting of fine-grained URI permissions does, however, require some cooperation with the content provider holding those URIs. It is strongly recommended that content providers implement this facility, and declare that they support it through the android:grantUriPermissions attribute or <grant-uri-permissions> tag.

More information can be found in the Context.grantUriPermission(), Context.revokeUriPermission(), and Context.checkUriPermission() methods.


# API GUIDE —— App Component

## Intent and IntentFilter

An Intent is a messaging object you can use to request an action from another app component. Although intents facilitate communication between components in several ways, there are three fundamental use-cases:

To start an activity:
An Activity represents a single screen in an app. You can start a new instance of an Activity by passing an Intent to startActivity(). The Intent describes the activity to start and carries any necessary data.
If you want to receive a result from the activity when it finishes, call startActivityForResult(). Your activity receives the result as a separate Intent object in your activity's onActivityResult() callback. For more information, see the Activities guide.
To start a service:
A Service is a component that performs operations in the background without a user interface. You can start a service to perform a one-time operation (such as download a file) by passing an Intent to startService(). The Intent describes the service to start and carries any necessary data.
If the service is designed with a client-server interface, you can bind to the service from another component by passing an Intent to bindService(). For more information, see the Services guide.
To deliver a broadcast:
A broadcast is a message that any app can receive. The system delivers various broadcasts for system events, such as when the system boots up or the device starts charging. You can deliver a broadcast to other apps by passing an Intent to sendBroadcast(), sendOrderedBroadcast(), or sendStickyBroadcast().
Intent Types
There are two types of intents:

Explicit intents specify the component to start by name (the fully-qualified class name). You'll typically use an explicit intent to start a component in your own app, because you know the class name of the activity or service you want to start. For example, start a new activity in response to a user action or start a service to download a file in the background.
Implicit intents do not name a specific component, but instead declare a general action to perform, which allows a component from another app to handle it. For example, if you want to show the user a location on a map, you can use an implicit intent to request that another capable app show a specified location on a map.
When you create an explicit intent to start an activity or service, the system immediately starts the app component specified in the Intent object.



When you create an implicit intent, the Android system finds the appropriate component to start by comparing the contents of the intent to the intent filters declared in the manifest file of other apps on the device. If the intent matches an intent filter, the system starts that component and delivers it the Intent object. If multiple intent filters are compatible, the system displays a dialog so the user can pick which app to use.

An intent filter is an expression in an app's manifest file that specifies the type of intents that the component would like to receive. For instance, by declaring an intent filter for an activity, you make it possible for other apps to directly start your activity with a certain kind of intent. Likewise, if you do not declare any intent filters for an activity, then it can be started only with an explicit intent.

Caution: To ensure your app is secure, always use an explicit intent when starting a Service and do not declare intent filters for your services. Using an implicit intent to start a service is a security hazard because you cannot be certain what service will respond to the intent, and the user cannot see which service starts. Beginning with Android 5.0 (API level 21), the system throws an exception if you call bindService() with an implicit intent.

Building an Intent
An Intent object carries information that the Android system uses to determine which component to start (such as the exact component name or component category that should receive the intent), plus information that the recipient component uses in order to properly perform the action (such as the action to take and the data to act upon).

The primary information contained in an Intent is the following:

Component name
The name of the component to start.
This is optional, but it's the critical piece of information that makes an intent explicit, meaning that the intent should be delivered only to the app component defined by the component name. Without a component name, the intent is implicit and the system decides which component should receive the intent based on the other intent information (such as the action, data, and category—described below). So if you need to start a specific component in your app, you should specify the component name.

Note: When starting a Service, you should always specify the component name. Otherwise, you cannot be certain what service will respond to the intent, and the user cannot see which service starts.

This field of the Intent is a ComponentName object, which you can specify using a fully qualified class name of the target component, including the package name of the app. For example, com.example.ExampleActivity. You can set the component name with setComponent(), setClass(), setClassName(), or with the Intent constructor.

Action
A string that specifies the generic action to perform (such as view or pick).
In the case of a broadcast intent, this is the action that took place and is being reported. The action largely determines how the rest of the intent is structured—particularly what is contained in the data and extras.

You can specify your own actions for use by intents within your app (or for use by other apps to invoke components in your app), but you should usually use action constants defined by the Intent class or other framework classes. Here are some common actions for starting an activity:

ACTION_VIEW
Use this action in an intent with startActivity() when you have some information that an activity can show to the user, such as a photo to view in a gallery app, or an address to view in a map app.
ACTION_SEND
Also known as the "share" intent, you should use this in an intent with startActivity() when you have some data that the user can share through another app, such as an email app or social sharing app.
See the Intent class reference for more constants that define generic actions. Other actions are defined elsewhere in the Android framework, such as in Settings for actions that open specific screens in the system's Settings app.

You can specify the action for an intent with setAction() or with an Intent constructor.

If you define your own actions, be sure to include your app's package name as a prefix. For example:

static final String ACTION_TIMETRAVEL = "com.example.action.TIMETRAVEL";
Data
The URI (a Uri object) that references the data to be acted on and/or the MIME type of that data. The type of data supplied is generally dictated by the intent's action. For example, if the action is ACTION_EDIT, the data should contain the URI of the document to edit.
When creating an intent, it's often important to specify the type of data (its MIME type) in addition to its URI. For example, an activity that's able to display images probably won't be able to play an audio file, even though the URI formats could be similar. So specifying the MIME type of your data helps the Android system find the best component to receive your intent. However, the MIME type can sometimes be inferred from the URI—particularly when the data is a content: URI, which indicates the data is located on the device and controlled by a ContentProvider, which makes the data MIME type visible to the system.

To set only the data URI, call setData(). To set only the MIME type, call setType(). If necessary, you can set both explicitly with setDataAndType().

Caution: If you want to set both the URI and MIME type, do not call setData() and setType() because they each nullify the value of the other. Always use setDataAndType() to set both URI and MIME type.

Category
A string containing additional information about the kind of component that should handle the intent. Any number of category descriptions can be placed in an intent, but most intents do not require a category. Here are some common categories:
CATEGORY_BROWSABLE
The target activity allows itself to be started by a web browser to display data referenced by a link—such as an image or an e-mail message.
CATEGORY_LAUNCHER
The activity is the initial activity of a task and is listed in the system's application launcher.
See the Intent class description for the full list of categories.

You can specify a category with addCategory().

These properties listed above (component name, action, data, and category) represent the defining characteristics of an intent. By reading these properties, the Android system is able to resolve which app component it should start.

However, an intent can carry additional information that does not affect how it is resolved to an app component. An intent can also supply:

Extras
Key-value pairs that carry additional information required to accomplish the requested action. Just as some actions use particular kinds of data URIs, some actions also use particular extras.
You can add extra data with various putExtra() methods, each accepting two parameters: the key name and the value. You can also create a Bundle object with all the extra data, then insert the Bundle in the Intent with putExtras().

For example, when creating an intent to send an email with ACTION_SEND, you can specify the "to" recipient with the EXTRA_EMAIL key, and specify the "subject" with the EXTRA_SUBJECT key.

The Intent class specifies many EXTRA_* constants for standardized data types. If you need to declare your own extra keys (for intents that your app receives), be sure to include your app's package name as a prefix. For example:

static final String EXTRA_GIGAWATTS = "com.example.EXTRA_GIGAWATTS";
Flags
Flags defined in the Intent class that function as metadata for the intent. The flags may instruct the Android system how to launch an activity (for example, which task the activity should belong to) and how to treat it after it's launched (for example, whether it belongs in the list of recent activities).
For more information, see the setFlags() method.

Example explicit intent

An explicit intent is one that you use to launch a specific app component, such as a particular activity or service in your app. To create an explicit intent, define the component name for the Intent object—all other intent properties are optional.

For example, if you built a service in your app, named DownloadService, designed to download a file from the web, you can start it with the following code:

// Executed in an Activity, so 'this' is the Context
// The fileUrl is a string URL, such as "http://www.example.com/image.png"
Intent downloadIntent = new Intent(this, DownloadService.class);
downloadIntent.setData(Uri.parse(fileUrl));
startService(downloadIntent);
The Intent(Context, Class) constructor supplies the app Context and the component a Class object. As such, this intent explicitly starts the DownloadService class in the app.

For more information about building and starting a service, see the Services guide.

Example implicit intent

An implicit intent specifies an action that can invoke any app on the device able to perform the action. Using an implicit intent is useful when your app cannot perform the action, but other apps probably can and you'd like the user to pick which app to use.

For example, if you have content you want the user to share with other people, create an intent with the ACTION_SEND action and add extras that specify the content to share. When you call startActivity() with that intent, the user can pick an app through which to share the content.

Caution: It's possible that a user won't have any apps that handle the implicit intent you send to startActivity(). If that happens, the call will fail and your app will crash. To verify that an activity will receive the intent, call resolveActivity() on your Intent object. If the result is non-null, then there is at least one app that can handle the intent and it's safe to call startActivity(). If the result is null, you should not use the intent and, if possible, you should disable the feature that issues the intent.

// Create the text message with a string
Intent sendIntent = new Intent();
sendIntent.setAction(Intent.ACTION_SEND);
sendIntent.putExtra(Intent.EXTRA_TEXT, textMessage);
sendIntent.setType("text/plain");
// Verify that the intent will resolve to an activity
if (sendIntent.resolveActivity(getPackageManager()) != null) {
    startActivity(sendIntent);
}
Note: In this case, a URI is not used, but the intent's data type is declared to specify the content carried by the extras.

When startActivity() is called, the system examines all of the installed apps to determine which ones can handle this kind of intent (an intent with the ACTION_SEND action and that carries "text/plain" data). If there's only one app that can handle it, that app opens immediately and is given the intent. If multiple activities accept the intent, the system displays a dialog so the user can pick which app to use..


orcing an app chooser

When there is more than one app that responds to your implicit intent, the user can select which app to use and make that app the default choice for the action. This is nice when performing an action for which the user probably wants to use the same app from now on, such as when opening a web page (users often prefer just one web browser) .

However, if multiple apps can respond to the intent and the user might want to use a different app each time, you should explicitly show a chooser dialog. The chooser dialog asks the user to select which app to use for the action every time (the user cannot select a default app for the action). For example, when your app performs "share" with the ACTION_SEND action, users may want to share using a different app depending on their current situation, so you should always use the chooser dialog, as shown in figure 2.

To show the chooser, create an Intent using createChooser() and pass it to startActivity(). For example:

Intent sendIntent = new Intent(Intent.ACTION_SEND);
...
// Always use string resources for UI text.
// This says something like "Share this photo with"
String title = getResources().getString(R.string.chooser_title);
// Create intent to show the chooser dialog
Intent chooser = Intent.createChooser(sendIntent, title);
// Verify the original intent will resolve to at least one activity
if (sendIntent.resolveActivity(getPackageManager()) != null) {
    startActivity(chooser);
}
This displays a dialog with a list of apps that respond to the intent passed to the createChooser() method and uses the supplied text as the dialog title.

Receiving an Implicit Intent
To advertise which implicit intents your app can receive, declare one or more intent filters for each of your app components with an <intent-filter> element in your manifest file. Each intent filter specifies the type of intents it accepts based on the intent's action, data, and category. The system will deliver an implicit intent to your app component only if the intent can pass through one of your intent filters.

Note: An explicit intent is always delivered to its target, regardless of any intent filters the component declares.

An app component should declare separate filters for each unique job it can do. For example, one activity in an image gallery app may have two filters: one filter to view an image, and another filter to edit an image. When the activity starts, it inspects the Intent and decides how to behave based on the information in the Intent (such as to show the editor controls or not).

Each intent filter is defined by an <intent-filter> element in the app's manifest file, nested in the corresponding app component (such as an <activity> element). Inside the <intent-filter>, you can specify the type of intents to accept using one or more of these three elements:

<action>
Declares the intent action accepted, in the name attribute. The value must be the literal string value of an action, not the class constant.
<data>
Declares the type of data accepted, using one or more attributes that specify various aspects of the data URI (scheme, host, port, path, etc.) and MIME type.
<category>
Declares the intent category accepted, in the name attribute. The value must be the literal string value of an action, not the class constant.
Note: In order to receive implicit intents, you must include the CATEGORY_DEFAULT category in the intent filter. The methods startActivity() and startActivityForResult() treat all intents as if they declared the CATEGORY_DEFAULT category. If you do not declare this category in your intent filter, no implicit intents will resolve to your activity.

For example, here's an activity declaration with an intent filter to receive an ACTION_SEND intent when the data type is text:

<activity android:name="ShareActivity">
    <intent-filter>
        <action android:name="android.intent.action.SEND"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data android:mimeType="text/plain"/>
    </intent-filter>
</activity>
It's okay to create a filter that includes more than one instance of <action>, <data>, or <category>. If you do, you simply need to be certain that the component can handle any and all combinations of those filter elements.

When you want to handle multiple kinds of intents, but only in specific combinations of action, data, and category type, then you need to create multiple intent filters.


An implicit intent is tested against a filter by comparing the intent to each of the three elements. To be delivered to the component, the intent must pass all three tests. If it fails to match even one of them, the Android system won't deliver the intent to the component. However, because a component may have multiple intent filters, an intent that does not pass through one of a component's filters might make it through on another filter. More information about how the system resolves intents is provided in the section below about Intent Resolution.

Caution: To avoid inadvertently running a different app's Service, always use an explicit intent to start your own service and do not declare intent filters for your service.

Note: For all activities, you must declare your intent filters in the manifest file. However, filters for broadcast receivers can be registered dynamically by calling registerReceiver(). You can then unregister the receiver with unregisterReceiver(). Doing so allows your app to listen for specific broadcasts during only a specified period of time while your app is running.

Example filters

To better understand some of the intent filter behaviors, look at the following snippet from the manifest file of a social-sharing app.

<activity android:name="MainActivity">
    <!-- This activity is the main entry, should appear in app launcher -->
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
<activity android:name="ShareActivity">
    <!-- This activity handles "SEND" actions with text data -->
    <intent-filter>
        <action android:name="android.intent.action.SEND"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data android:mimeType="text/plain"/>
    </intent-filter>
    <!-- This activity also handles "SEND" and "SEND_MULTIPLE" with media data -->
    <intent-filter>
        <action android:name="android.intent.action.SEND"/>
        <action android:name="android.intent.action.SEND_MULTIPLE"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data android:mimeType="application/vnd.google.panorama360+jpg"/>
        <data android:mimeType="image/*"/>
        <data android:mimeType="video/*"/>
    </intent-filter>
</activity>
The first activity, MainActivity, is the app's main entry point—the activity that opens when the user initially launches the app with the launcher icon:

The ACTION_MAIN action indicates this is the main entry point and does not expect any intent data.
The CATEGORY_LAUNCHER category indicates that this activity's icon should be placed in the system's app launcher. If the <activity> element does not specify an icon with icon, then the system uses the icon from the <application> element.
These two must be paired together in order for the activity to appear in the app launcher.

The second activity, ShareActivity, is intended to facilitate sharing text and media content. Although users might enter this activity by navigating to it from MainActivity, they can also enter ShareActivity directly from another app that issues an implicit intent matching one of the two intent filters.

Note: The MIME type, application/vnd.google.panorama360+jpg, is a special data type that specifies panoramic photos, which you can handle with the Google panorama APIs.

Using a Pending Intent
A PendingIntent object is a wrapper around an Intent object. The primary purpose of a PendingIntent is to grant permission to a foreign application to use the contained Intent as if it were executed from your app's own process.

Major use cases for a pending intent include:

Declare an intent to be executed when the user performs an action with your Notification (the Android system's NotificationManager executes the Intent).
Declare an intent to be executed when the user performs an action with your App Widget (the Home screen app executes the Intent).
Declare an intent to be executed at a specified time in the future (the Android system's AlarmManager executes the Intent).
Because each Intent object is designed to be handled by a specific type of app component (either an Activity, a Service, or a BroadcastReceiver), so too must a PendingIntent be created with the same consideration. When using a pending intent, your app will not execute the intent with a call such as startActivity(). You must instead declare the intended component type when you create the PendingIntent by calling the respective creator method:

PendingIntent.getActivity() for an Intent that starts an Activity.
PendingIntent.getService() for an Intent that starts a Service.
PendingIntent.getBroadcast() for a Intent that starts an BroadcastReceiver.
Unless your app is receiving pending intents from other apps, the above methods to create a PendingIntent are the only PendingIntent methods you'll probably ever need.

Each method takes the current app Context, the Intent you want to wrap, and one or more flags that specify how the intent should be used (such as whether the intent can be used more than once).

More information about using pending intents is provided with the documentation for each of the respective use cases, such as in the Notifications and App Widgets API guides.

Intent Resolution
When the system receives an implicit intent to start an activity, it searches for the best activity for the intent by comparing the intent to intent filters based on three aspects:

The intent action
The intent data (both URI and data type)
The intent category
The following sections describe how an intents are matched to the appropriate component(s) in terms of how the intent filter is declared in an app's manifest file.

Action test

To specify accepted intent actions, an intent filter can declare zero or more <action> elements. For example:

<intent-filter>
    <action android:name="android.intent.action.EDIT" />
    <action android:name="android.intent.action.VIEW" />
    ...
</intent-filter>
To get through this filter, the action specified in the Intent must match one of the actions listed in the filter.

If the filter does not list any actions, there is nothing for an intent to match, so all intents fail the test. However, if an Intent does not specify an action, it will pass the test (as long as the filter contains at least one action).

Category test

To specify accepted intent categories, an intent filter can declare zero or more <category> elements. For example:

<intent-filter>
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    ...
</intent-filter>
For an intent to pass the category test, every category in the Intent must match a category in the filter. The reverse is not necessary—the intent filter may declare more categories than are specified in the Intent and the Intent will still pass. Therefore, an intent with no categories should always pass this test, regardless of what categories are declared in the filter.

Note: Android automatically applies the the CATEGORY_DEFAULT category to all implicit intents passed to startActivity() and startActivityForResult(). So if you want your activity to receive implicit intents, it must include a category for "android.intent.category.DEFAULT" in its intent filters (as shown in the previous <intent-filter> example.

Data test

To specify accepted intent data, an intent filter can declare zero or more <data> elements. For example:

<intent-filter>
    <data android:mimeType="video/mpeg" android:scheme="http" ... />
    <data android:mimeType="audio/mpeg" android:scheme="http" ... />
    ...
</intent-filter>
Each <data> element can specify a URI structure and a data type (MIME media type). There are separate attributes — scheme, host, port, and path — for each part of the URI:

<scheme>://<host>:<port>/<path>

For example:

content://com.example.project:200/folder/subfolder/etc

In this URI, the scheme is content, the host is com.example.project, the port is 200, and the path is folder/subfolder/etc.

Each of these attributes is optional in a <data> element, but there are linear dependencies:

If a scheme is not specified, the host is ignored.
If a host is not specified, the port is ignored.
If both the scheme and host are not specified, the path is ignored.
When the URI in an intent is compared to a URI specification in a filter, it's compared only to the parts of the URI included in the filter. For example:

If a filter specifies only a scheme, all URIs with that scheme match the filter.
If a filter specifies a scheme and an authority but no path, all URIs with the same scheme and authority pass the filter, regardless of their paths.
If a filter specifies a scheme, an authority, and a path, only URIs with the same scheme, authority, and path pass the filter.
Note: A path specification can contain a wildcard asterisk (*) to require only a partial match of the path name.

The data test compares both the URI and the MIME type in the intent to a URI and MIME type specified in the filter. The rules are as follows:

An intent that contains neither a URI nor a MIME type passes the test only if the filter does not specify any URIs or MIME types.
An intent that contains a URI but no MIME type (neither explicit nor inferable from the URI) passes the test only if its URI matches the filter's URI format and the filter likewise does not specify a MIME type.
An intent that contains a MIME type but not a URI passes the test only if the filter lists the same MIME type and does not specify a URI format.
An intent that contains both a URI and a MIME type (either explicit or inferable from the URI) passes the MIME type part of the test only if that type matches a type listed in the filter. It passes the URI part of the test either if its URI matches a URI in the filter or if it has a content: or file: URI and the filter does not specify a URI. In other words, a component is presumed to support content: and file: data if its filter lists only a MIME type.
This last rule, rule (d), reflects the expectation that components are able to get local data from a file or content provider. Therefore, their filters can list just a data type and do not need to explicitly name the content: and file: schemes. This is a typical case. A <data> element like the following, for example, tells Android that the component can get image data from a content provider and display it:

<intent-filter>
    <data android:mimeType="image/*" />
    ...
</intent-filter>
Because most available data is dispensed by content providers, filters that specify a data type but not a URI are perhaps the most common.

Another common configuration is filters with a scheme and a data type. For example, a <data> element like the following tells Android that the component can retrieve video data from the network in order to perform the action:

<intent-filter>
    <data android:scheme="http" android:type="video/*" />
    ...
</intent-filter>
Intent matching

Intents are matched against intent filters not only to discover a target component to activate, but also to discover something about the set of components on the device. For example, the Home app populates the app launcher by finding all the activities with intent filters that specify the ACTION_MAIN action and CATEGORY_LAUNCHER category.

Your application can use intent matching in a similar way. The PackageManager has a set of query...() methods that return all components that can accept a particular intent, and a similar series of resolve...() methods that determine the best component to respond to an intent. For example, queryIntentActivities() returns a list of all activities that can perform the intent passed as an argument, and queryIntentServices() returns a similar list of services. Neither method activates the components; they just list the ones that can respond. There's a similar method, queryBroadcastReceivers(), for broadcast receivers.


### Common Intents