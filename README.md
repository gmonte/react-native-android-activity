# react-native-android-activity

This sample, which grew out of a [question on Stack Overflow](https://stackoverflow.com/questions/42253397/call-android-activity-from-react-native-code/43675819), demonstrates the interface between React Native JavaScript and Java code in the Android host application. It can do the following:

* Call from JavaScript into native modules:
  * These use a custom native module called `ActivityStarter`:
    * Navigate from React Native to a Java activity internal to the host app;
    * Start an external intent to dial a phone number, passing data from JavaScript;
    * Query the host app for information.
  * This uses the native module `Clipboard`, which [comes with React Native out of the box](https://github.com/facebook/react-native/blob/master/ReactAndroid/src/main/java/com/facebook/react/modules/clipboard/ClipboardModule.java):
    * Copy information to the clipboard.
* Call a JavaScript method from Java. This one invokes `ActivityStarter.callJavaScript`, which in turn calls a JavaScript method from Java (using an officially undocumented approach).
* Verify that custom edit menu extensions work with React Native `TextInput`.
* Add a custom menu option to React Native debug menu.

Technically there is no difference between the `ActivityStarter` and `Clipboard` native modules, except one is defined here, while the other ships as part of React Native.

The starting point for this sample is a slightly tweaked standard React Native project as generated by `react-native init`. We add five buttons to the generated page:

![Android Demo App](img/AndroidScreenShot.png)

## Getting started

<!-- markdownlint-disable MD031 -->

* Install [Git](https://git-scm.com/downloads).
* Install [Node.js](https://nodejs.org/en/download/). Use a shell with Node and Git in the path (or the Node.js shell itself) for all commands.
* Clone this project: `git clone https://github.com/petterh/react-native-android-activity.git`
  * Alternatively, create your own fork and clone that.
* `cd react-native-android-activity`
* Update `npm`: `npm install -g npm`
* Run `npm install` to download dependencies
* Install [Android Studio](https://developer.android.com/studio/install.html) (follow instructions [on this page](https://facebook.github.io/react-native/docs/getting-started.html)).
* By default, the debug build of the app loads the JS bundle from your dev box, so start a packager:
  ```cmd
  npm run start
  ```
* Connect an Android device via USB, or use an emulator. [Don't forget to enable USB Debugging in Developer options](https://developer.android.com/studio/run/device).
* Open the app in Android Studio and run it.
* If this fails with the message "Could not get BatchedBridge, make sure your bundle is packaged correctly", your packager is likely not running.
* If it complains about connecting to the dev server, run `adb reverse tcp:8081 tcp:8081`
* If it crashes while opening the ReactNative controls, try to modify the following phone settings:
  **Android Settings -> Apps -> Settings once again (the gear) to go to Configure Apps view -> Draw over other apps -> Allow React Native Android Activity Demo to draw over other apps**. (The demo app *should* ask for this automatically, though.)
* To embed the bundle in the apk (and not have to run the packager), do two changes:
  * In `MainApplication`, make `getUseDeveloperSupport` return `false`.
  * In `app/build.gradle`, set `bundleInDebug: true`.

<!-- markdownlint-enable MD031 -->

## The React Native side

The gist of the JavaScript code looks like this:

```javascript
import { ..., NativeModules, ... } from 'react-native';

export default class ActivityDemoComponent extends Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native!
        </Text>
        <Text style={styles.instructions}>
          To get started, edit index.android.js
        </Text>
        <!-- Menu buttons: https://facebook.github.io/react-native/docs/debugging -->
        <Text style={styles.instructions}>
          Double tap R on your keyboard to reload,{'\n'}
          Shake or press menu button for dev menu
        </Text>
        <View style={styles.buttonContainer}>
          <Button
            onPress={() => NativeModules.ActivityStarter.navigateToExample()}
            title='Start example activity'
          />
          <Button
            onPress={() => NativeModules.ActivityStarter.dialNumber('+1 (234) 567-8910')}
            title='Dial +1 (234) 567-8910'
          />
          <Button
            onPress={() => NativeModules.ActivityStarter.getName((name) => { alert(name); })}
            title='Get activity name'
          />
          <Button
            onPress={() => NativeModules.Clipboard.setString("Hello from JavaScript!")}
            title='Copy to clipboard'
          />
        </View>
      </View>
    );
  }
}
```

The first three buttons use three methods on `NativeModules.ActivityStarter`. Where does this come from?

## The Java module

`ActivityStarter` is just a Java class that implements a React Native Java interface called `NativeModule`. The heavy lifting of this interface is already done by `BaseJavaModule`, so one normally extends either that one or `ReactContextBaseJavaModule`:

```java
class ActivityStarterModule extends ReactContextBaseJavaModule {

    ActivityStarterModule(ReactApplicationContext reactContext) {
        super(reactContext);
    }

    @Override
    public String getName() {
        return "ActivityStarter";
    }

    @ReactMethod
    void navigateToExample() {
        ReactApplicationContext context = getReactApplicationContext();
        Intent intent = new Intent(context, ExampleActivity.class);
        context.startActivity(intent);
    }

    @ReactMethod
    void dialNumber(@Nonnull String number) {
        Intent intent = new Intent(Intent.ACTION_DIAL, Uri.parse("tel:" + number));
        getReactApplicationContext().startActivity(intent);
    }

    @ReactMethod
    void getActivityName(@Nonnull Callback callback) {
        Activity activity = getCurrentActivity();
        if (activity != null) {
            callback.invoke(activity.getClass().getSimpleName());
        }
    }
}
```

The name of this class doesn't matter; the `ActivityStarter` module name exposed to JavaScript comes from the `getName()` method.

Each method annotated with a `@ReactMethod` attribute is accessible from JavaScript. Overloads are not allowed, though; you have to know the method signatures. (The out-of-the-box `Clipboard` module isn't usually accessed the way I do it here; React Native includes [`Clipboard.js`](https://github.com/facebook/react-native/blob/master/Libraries/Components/Clipboard/Clipboard.js), which [makes the thing more accessible from JavaScript](https://facebook.github.io/react-native/docs/clipboard.html) &ndash; if you're creating modules for public consumption, consider doing something similar.)

A `@ReactMethod` must be of type `void`. In the case of `getActivityName()` we want to return a string; we do this by using a callback.

## Connecting the dots

The default app generated by `react-native init` contains a `MainApplication` class that initializes React Native. Among other things it extends `ReactNativeHost` to override its `getPackages` method:

```java
@Override
protected List<ReactPackage> getPackages() {
    return Arrays.<ReactPackage>asList(
            new MainReactPackage()
    );
}
```

This is the point where we hook our Java code to the React Native machinery. Create a class that implements `ReactPackage` and override `createNativeModules`:

```java
class ActivityStarterReactPackage implements ReactPackage {
    @Override
    public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
        List<NativeModule> modules = new ArrayList<>();
        modules.add(new ActivityStarterModule(reactContext));
        return modules;
    }

    @Override
    public List<Class<? extends JavaScriptModule>> createJSModules() {
        return Collections.emptyList();
    }

    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }
}
```

Finally, update `MainApplication` to include our new package:

```java
public class MainApplication extends Application implements ReactApplication {

    private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {
        @Override
        public boolean getUseDeveloperSupport() {
            return BuildConfig.DEBUG;
        }

        @Override
        protected List<ReactPackage> getPackages() {
            return Arrays.<ReactPackage>asList(
                    new ActivityStarterReactPackage(), // This is it!
                    new MainReactPackage()
            );
        }
    };

    @Override
    public ReactNativeHost getReactNativeHost() {
        return mReactNativeHost;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        SoLoader.init(this, false);
    }
}
```

## Calling JavaScript from Java

This demo is invoked by the last button on the page:

```javascript
<Button
    onPress={() => NativeModules.ActivityStarter.callJavaScript()}
    title='Call JavaScript from Java'
/>
```

The Java side looks like this (in `ActivityStarterReactPackage` class):

```java
@ReactMethod
void callJavaScript() {
    Activity activity = getCurrentActivity();
    if (activity != null) {
        MainApplication application = (MainApplication) activity.getApplication();
        ReactNativeHost reactNativeHost = application.getReactNativeHost();
        ReactInstanceManager reactInstanceManager = reactNativeHost.getReactInstanceManager();
        ReactContext reactContext = reactInstanceManager.getCurrentReactContext();

        if (reactContext != null) {
            CatalystInstance catalystInstance = reactContext.getCatalystInstance();
            WritableNativeArray params = new WritableNativeArray();
            params.pushString("Hello, JavaScript!");
            catalystInstance.callFunction("JavaScriptVisibleToJava", "alert", params);
        }
    }
}
```

The JavaScript method we're calling is defined and made visible to Java as follows:

```javascript
import BatchedBridge from "react-native/Libraries/BatchedBridge/BatchedBridge";

export class ExposedToJava {
  alert(message) {
      alert(message);
  }
}

const exposedToJava = new ExposedToJava();
BatchedBridge.registerCallableModule("JavaScriptVisibleToJava", exposedToJava);
```

## Summary

1. The main application class initializes React Native and creates a `ReactNativeHost` whose `getPackages` include our package in its list.
1. `ActivityStarterReactPackage` includes `ActivityStarterModule` in its native modules list.
1. `ActivityStarterModule` returns "ActivityStarter" from its `getName` method, and annotates three methods with the `ReactMethod` attribute.
1. JavaScript can access `ActivityStarter.getActivityName` and friends via `NativeModules`.

## Addendum

I just added a second version of `ActivityStarterModule.getActivityName` called `getActivityNameAsPromise`, with a corresponding button.

## Addendum 2

[I just added event triggering, another way to communicate](https://github.com/petterh/react-native-android-activity/commit/e63706e2ca828d4de4db1bf7cf85fe5be28d648d). Tap **Start Example Activity**, then **Trigger event**.

## Further reading

[Native Modules](https://facebook.github.io/react-native/docs/native-modules-android.html)

## iOS

This is primarily an Android sample &ndash; but if I can lay my hands on a Mac, or enlist the help of someone with a Mac, I'll add the corresponding functionality to the iOS version of the app.
