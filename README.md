# Building for multiple environments with React-Native

Problem build an app that runs on multiple environment (local, staging, production, etc.) and install all of them on the same device.

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/Android.png">

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/ios.png">

# Initialize react-native

```
npx react-native init multi_build --directory multi-build
```

# Android

## Adding flavors for each environment

go to `android/app/build.gradle`, below `buildTypes` block, add the following:

```groovy
flavorDimensions "version"
productFlavors {
    local {
        applicationIdSuffix ".local"
    }
    staging {
        applicationIdSuffix ".staging"
    }
    production {}
}
```

the `local`, `staging`, and `production` are all our the different environments we are going to build for, the `applicationIdSuffix` allows us to add a suffix on the `applicationId` and allow us to install all of them on the same device.

## Displaying different names for each environment

You'll need to display different names to differentiate them together, you can do that by creating `android/app/src/{environment}/res/values/strings.xml` for each environment.

Create `android/app/src/local/res/values/strings.xml` and add the following:

```xml
<resources>
  <string name="app_name">MulBuild LOC</string>
</resources>
```

Do the same thing for `staging` environment.

**NOTE:** For production, you don't have to since it will use the `android/app/src/main/res/values/strings.xml`.

## Running the app

To run the app, you need to do `npm run android -- --variant=$(flavor)$(buildType)` where the `$(flavor)` is one of the environments we defined on `productFlavors`, in this example, we have the following variants:

- `localDebug`
- `localRelease`
- `stagingDebug`
- `stagingRelease`
- `productionDebug`
- `productionRelease`

When developing locally, we'll need to run `npm run android -- --variant=localDebug`

### Known issues

- [Error: Activity class {com.myapp/com.myapp.MainActivity} does not exist.](https://github.com/facebook/react-native/issues/30092)

## Building AAB

To bundle the `AAB` files to be uploaded to the playstore, you can still run `./gradlew bundleRelease` which will bundle release for each of the flavors we have defined and you'll see the outputs on `android/app/build/outputs/bundle`. However, you might only want to bundle one flavor, you can run `./gradlew bundleProductionRelease` which will only bundle `production`.

###### That's it!

# iOS

Using XCode, open `ios/multi_build.xcworkspace`.

## Adding configurations for each environment

In your `ios/Podfile`, add the following code:

```ruby
# ...
platform :ios, '12.0'

# start here <----
project 'multi_build',
        'LocalDebug' => :debug,
        'LocalRelease' => :release,
        'StagingDebug' => :debug,
        'StagingRelease' => :release
# end here

target 'multi_build' do
# ...
```

1. On the left sidebar, click on the `multi_build` xcode project.
2. Then on the second left sidebar, under `PROJECT`, click on the `multi_build` project.
3. Click on `Info` tab.
4. Under `Configuration`, you'll see that there should be `Debug` and `Release` there initially, there's also a `+` button just below that, click on that `+` button.
5. Click on `Duplicate "Debug" Configuration`; name the configuration as `LocalDebug`.
6. Click on that `+` button again.
5. Click on `Duplicate "Release" Configuration`; name the configuration as `LocalRelease`.

Do the same thing for `Staging`

**NOTE:** For production, you don't have to since it will use the `Debug` and `Release` by default.

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20new%20configuration.png">

## Adding schema for each environment

1. On the topbar, click where it says `multi_build >`; Then click on `Edit Schema`

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/Add%20schema%201.png">

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20schema%202.png">

2. A dialog will appear, just click on `Duplicate Schema`; Then name that "Local"

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20schema%203.png">

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20schema%204.png">

3. Where it says "Debug", change it to "LocalDebug"; Wher it says "Release", change it to "LocalRelease"; Do this for all the panels on the left sidebar, to start with, press on `Run` and change the "Build Configuration".

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20schema%205.png">

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20schema%206.png">

Do this same process for the `Staging` environment.

## Using different Bundle ID, display name, product name, and info plist for each environment

Having different names and bundle id for each environment will allow us to install all environments in one device and differentiate them.

### To use different display name for each environment

On `info.plist` change the `Bundle display name` to `$(PRODUCT_NAME)`

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/display%20name.png">

### To use different info plist for each environment (optional)

Now copy the `info.plist` twice and then rename one copy to `info_local.plist` and other `info_staging.plist`.

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20info%20plist%201.png">

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20info%20plist%202.png">

Since you'll be using the `info.plist` for production builds, you can remove the `App Transport Security Settings` entry on it.

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20info%20plist%204.png">

Then add the newly created `info_local.plist` and `info_staging.plist` to the project.

###### Note: the add file dialog might be too small, you can expand it to view more.

By the end, you should have the following:

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20info%20plist%203.png">

### To use different bundle id, product name, and info plist for each environment

On the `targets`, select the `multi_build` and go to `build settings` and click on the plus button and click on `Add User-Defined Setting` and name that `BUNDLE_ID_SUFFIX` and just change the name for each build configurations accordingly then on `info.plist`, change the `Bundle identifier` to `$(PRODUCT_BUNDLE_IDENTIFIER).$(BUNDLE_ID_SUFFIX)`

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/different%20variables%20per%20schema.png">

## Running the App

1. Run `cd ios && pod install && cd ..`
2. Choose schema, choose "Local" for example.
3. Choose a simulator you want to use.
4. Then click on the "Play" button.

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/run%20app.png">

###### That's it!