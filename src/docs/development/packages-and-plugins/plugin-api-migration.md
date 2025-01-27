---
title: Supporting the new Android plugins APIs
description: How to update a plugin using the old APIs to support the new APIs.
---

{{site.alert.note}}
  You might be directed to this page if the framework detects that
  your app uses a plugin based on the old Android APIs.
{{site.alert.end}}

_If you don't write or maintain an Android Flutter plugin, you can skip this page._

As of the 1.12 release, new plugin APIs are available for the Android platform.
The old APIs based on [`PluginRegistry.Registrar`][] won't be immediately
deprecated, but we encourage you to migrate to the new APIs based on
[`FlutterPlugin`][].

The new API has the advantage of providing a cleaner set of accessors for
lifecycle dependent components compared to the old APIs. For instance
[`PluginRegistry.Registrar.activity()`][] could return null if Flutter isn't
attached to any activities.

In other words, plugins using the old API may produce undefined behaviors when
embedding Flutter into an Android app.
Most of the Flutter plugins provided by [flutter.dev][] have
been migrated already. (Learn how to become a
[verified publisher][] on pub.dev!) For an example of
a plugin that uses the new APIs, see the
[battery package][].

## Upgrade steps

The following instructions outline the steps for supporting the new API:

1. Update the main plugin class (`*Plugin.java`) to implement the
   [`FlutterPlugin`][] interface. For more complex plugins, you can separate the
   `FlutterPlugin` and `MethodCallHandler` into two classes. See the next
   section, [Basic plugin][], for more details on accessing app resources with
   the latest version (v2) of embedding.
   <br><br>
   Also, note that the plugin should still contain the static `registerWith()`
   method to remain compatible with apps that don't use the v2 Android embedding.
   (See [flutter.dev/go/android-project-migration][] for details.) The
   easiest thing to do (if possible) is move the logic from `registerWith()`
   into a private method that both `registerWith()` and `onAttachedToEngine()`
   can call. Either `registerWith()` or `onAttachToEngine()` will be called, not
   both.
   <br><br>
   In addition, you should document all non-overridden public members within the
   plugin. In an add-to-app scenario, these classes will be accessible to a
   developer and require documentation.

1. (Optional) If your plugin needs an `Activity` reference, also implement
   the [`ActivityAware`][] interface.

1. (Optional) If your plugin is expected to be held in a background Service at
   any point in time, implement the [`ServiceAware`][] interface.

1. Update the example app's `MainActivity.java` to use the
   v2 embedding FlutterActivity. See [flutter.dev/go/android-project-migration][]
   for details. You may have to make a public constructor for you plugin class
   if one didn't exist already. For example:

   ```java
    package io.flutter.plugins.firebasecoreexample;

    import io.flutter.embedding.android.FlutterActivity;
    import io.flutter.embedding.engine.FlutterEngine;
    import io.flutter.plugins.firebase.core.FirebaseCorePlugin;

    public class MainActivity extends FlutterActivity {
      // You can keep this empty class or remove it. Plugins on the new embedding
      // now automatically registers plugins.
    }
    ```

1. (Optional) Create an `EmbeddingV1Activity.java` file that uses the v1
   embedding for the example project in the same folder as `MainActivity` to
   keep testing the v1 embedding's compatibility with your plugin. For example:

    ```java
    package io.flutter.plugins.firebasecoreexample;

    import android.os.Bundle;
    import io.flutter.app.FlutterActivity;
    import io.flutter.plugins.GeneratedPluginRegistrant;

    public class EmbeddingV1Activity extends FlutterActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      GeneratedPluginRegistrant.registerWith(this);
    }
    }
    ```

1. (Optional) If you created an `EmbeddingV1Activity` in the step above, add the
   `EmbeddingV1Activity` to the
   <plugin_name>/example/android/app/src/main/AndroidManifest.xml.
   For example:

    ```xml
    <activity
        android:name=".EmbeddingV1Activity"
        android:theme="@style/LaunchTheme"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|layoutDirection|fontScale"
        android:hardwareAccelerated="true"
        android:windowSoftInputMode="adjustResize">
    </activity>
    ```

## Testing your plugin

The remaining steps address testing your plugin, which we encourage,
but aren't required.

1. Update `<plugin_name>/example/android/app/build.gradle`
   to replace references to `android.support.test` with `androidx.test`:

    ```groovy
    defaultConfig {
      ...
      testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
      ...
    }
    ```

    ```groovy
    dependencies {
    ...
    androidTestImplementation 'androidx.test:runner:1.2.0'
    androidTestImplementation 'androidx.test:rules:1.2.0'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.2.0'
    ...
    }
    ```

1. Add tests files for `MainActivity` and `EmbeddingV1Activity`
   in `<plugin_name>/example/android/app/src/androidTest/java/<plugin_path>/`.
   You will need to create these directories. For example:

    ```java
    package io.flutter.plugins.firebase.core;

    import androidx.test.rule.ActivityTestRule;
    import dev.flutter.plugins.e2e.FlutterRunner;
    import io.flutter.plugins.firebasecoreexample.MainActivity;
    import org.junit.Rule;
    import org.junit.runner.RunWith;

    @RunWith(FlutterRunner.class)
    public class MainActivityTest {
      @Rule public ActivityTestRule<MainActivity> rule = new ActivityTestRule<>(MainActivity.class);
    }
    ```

    ```java
    package io.flutter.plugins.firebase.core;

    import androidx.test.rule.ActivityTestRule;
    import dev.flutter.plugins.e2e.FlutterRunner;
    import io.flutter.plugins.firebasecoreexample.EmbeddingV1Activity;
    import org.junit.Rule;
    import org.junit.runner.RunWith;

    @RunWith(FlutterRunner.class)
    public class EmbeddingV1ActivityTest {
      @Rule
      public ActivityTestRule<EmbeddingV1Activity> rule =
          new ActivityTestRule<>(EmbeddingV1Activity.class);
    }
    ```

1. Add `e2e` and `flutter_driver` dev_dependencies to
   `<plugin_name>/pubspec.yaml` and
   `<plugin_name>/example/pubspec.yaml`.

    ```yaml
    e2e: ^0.2.1
    flutter_driver:
      sdk: flutter
    ```

1. Update minimum Flutter version of environment in
   `<plugin_name>/pubspec.yaml`. All plugins moving
   forward will set the minimum version to 1.9.1+hotfix.4
   which is the minimum version for which we can guarantee support.
   For example:

    ```yaml
    environment:
      sdk: ">=2.0.0-dev.28.0 <3.0.0"
      flutter: ">=1.12.13+hotfix.6 <2.0.0"

    ```

1. Create a simple test in `<plugin_name>/test/<plugin_name>_e2e.dart`.
   For the purpose of testing the PR that adds the v2 embedding support,
   we're trying to test some very basic functionality of the plugin.
   This is a smoke test to ensure that the plugin properly registers
   with the new embedder. For example:

    ```dart
    import 'package:flutter_test/flutter_test.dart';
    import 'package:battery/battery.dart';
    import 'package:e2e/e2e.dart';

    void main() {
      E2EWidgetsFlutterBinding.ensureInitialized();

      testWidgets('Can get battery level', (WidgetTester tester) async {
        final Battery battery = Battery();
        final int batteryLevel = await battery.batteryLevel;
        expect(batteryLevel, isNotNull);
      });
    }
    ```

1. Test run the e2e tests locally. In a terminal,
   do the following:

    ```sh
    cd <plugin_name>/example
    flutter build apk
    cd android
    ./gradlew app:connectedAndroidTest -Ptarget=`pwd`/../../test/<plugin_name>_e2e.dart
    ```

## Basic plugin

To get started with a Flutter Android plugin in code,
start by implementing `FlutterPlugin`.

```java
public class MyPlugin implements FlutterPlugin {
  @Override
  public void onAttachedToEngine(@NonNull FlutterPluginBinding binding) {
    // TODO: your plugin is now attached to a Flutter experience.
  }

  @Override
  public void onDetachedFromEngine(@NonNull FlutterPluginBinding binding) {
    // TODO: your plugin is no longer attached to a Flutter experience.
  }
}
```

As shown above, your plugin may or may not be associated with
a given Flutter experience at any given moment in time.
You should take care to initialize your plugin's behavior
in onAttachedToEngine(), and then cleanup your plugin's references
in onDetachedFromEngine().

The FlutterPluginBinding provides your plugin with a few
important references:

**binding.getFlutterEngine()**
: Returns the `FlutterEngine` that your plugin is attached to,
  providing access to components like the DartExecutor,
  FlutterRenderer, and more.

**binding.getApplicationContext()**
: Returns the Android application's Context for the running app.

## UI/Activity plugin

If your plugin needs to interact with the UI,
such as requesting permissions, or altering Android UI chrome,
then you need to take additional steps to define your plugin.
You must implement the ActivityAware interface.

```java
public class MyPlugin implements FlutterPlugin, ActivityAware {
  //...normal plugin behavior is hidden...

  @Override
  public void onAttachedToActivity(ActivityPluginBinding activityPluginBinding) {
    // TODO: your plugin is now attached to an Activity
  }

  @Override
  public void onDetachedFromActivityForConfigChanges() {
    // TODO: the Activity your plugin was attached to was
    // destroyed to change configuration.
    // This call will be followed by onReattachedToActivityForConfigChanges().
  }

  @Override
  public void onReattachedToActivityForConfigChanges(ActivityPluginBinding activityPluginBinding) {
    // TODO: your plugin is now attached to a new Activity
    // after a configuration change.
  }

  @Override
  public void onDetachedFromActivity() {
    // TODO: your plugin is no longer associated with an Activity.
    // Clean up references.
  }
}
```

To interact with an `Activity`, your `ActivityAware` plugin must
implement appropriate behavior at 4 stages. First, your plugin
is attached to an `Activity`. You can access that `Activity` and
a number of its callbacks through the provided `ActivityPluginBinding`.

Since `Activity`s can be destroyed during configuration changes,
you must cleanup any references to the given `Activity` in
`onDetachedFromActivityForConfigChanges()`,
and then re-establish those references in
`onReattachedToActivityForConfigChanges()`.

Finally, in `onDetachedFromActivity()` your plugin should clean
up all references related to `Activity` behavior and return to
a non-UI configuration.

[`PluginRegistry.Registrar.activity()`]: {{site.api}}/javadoc/io/flutter/plugin/common/PluginRegistry.Registrar.html#activity--
[`PluginRegistry.Registrar`]: {{site.api}}/javadoc/io/flutter/plugin/common/PluginRegistry.Registrar.html
[`FlutterPlugin`]: {{site.api}}/javadoc/io/flutter/embedding/engine/plugins/FlutterPlugin.html
[flutter.dev/go/android-project-migration]: /go/android-project-migration
[`ActivityAware`]: {{site.api}}/javadoc/io/flutter/embedding/engine/plugins/activity/ActivityAware.html
[`ServiceAware`]: {{site.api}}/javadoc/io/flutter/embedding/engine/plugins/service/ServiceAware.html
[Basic plugin]: #basic-plugin
[battery package]: {{site.github}}/flutter/plugins/tree/master/packages/battery
[flutter.dev]: {{site.pub}}/flutter.dev/packages
[verified publisher]: {{site.dart-site}}/tools/pub/verified-publishers
