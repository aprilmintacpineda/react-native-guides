# Building for multiple environments with React-Native

Problem build an app that runs on multiple environment (local, staging, production, etc.) and install all of them on the same device.

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/Android.png">

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/ios.png">

## Initialize react-native

```
npx react-native init multi_build --directory multi-build
```

## Android

### Adding flavors for each environment

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

### Displaying different names for each environment

You'll need to display different names to differentiate them together, you can do that by creating `android/app/src/{environment}/res/values/strings.xml` for each environment.

Create `android/app/src/local/res/values/strings.xml` and add the following:

```xml
<resources>
  <string name="app_name">MulBuild LOC</string>
</resources>
```

Do the same thing for `staging` environment.

**NOTE:** For production, you don't have to since it will use the `android/app/src/main/res/values/strings.xml`.

### Running the app

To run the app, you need to do `npm run android -- --variant=$(flavor)$(buildType)` where the `$(flavor)` is one of the environments we defined on `productFlavors`, in this example, we have the following variants:

- `localDebug`
- `localRelease`
- `stagingDebug`
- `stagingRelease`
- `productionDebug`
- `productionRelease`

When developing locally, we'll need to run `npm run android -- --variant=localDebug`

### Building AAB

To bundle the `AAB` files to be uploaded to the playstore, you can still run `./gradlew bundleRelease` which will bundle release for each of the flavors we have defined and you'll see the outputs on `android/app/build/outputs/bundle`. However, you might only want to bundle one flavor, you can run `./gradlew bundleProductionRelease` which will only bundle `production`.

###### That's it!

## iOS

### Adding configurations for each environment

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20new%20configuration.png">

Open your `ios/multi_build.xcworkspace` and add configuration, click on the project on the left side, then on the project on the right side, and then on info tab, by default there are only `Debug` and `Release` there.

Create a debug configuration for `Local` environment by clicking on the `plus icon` on the configuration panel, and then click on `Duplicate "Debug" configuration` and name that configuration as `LocalDebug`.

Create a release configuration for `Local` environment by clicking on the `plus icon` on the configuration panel, and then click on `Duplicate "Release" configuration` and name that configuration as `LocalRelease`.

Do the same thing for `Staging` environment.

**NOTE:** For production, you don't have to since it will use the `Debug` and `Release` by default.

### Adding schema for each environment

On the top left, click where it says `multi_build > ` then a drop down will appear, click on `edit schema` then a dialog will appear. By default the `multi_build` is selected, this is the default schema and the one we will use for production. On the bottom there's a `Duplicate Schema` button, click on that and name the new schema as `local`, this is what we will use for the local environment. Then go through each of the tabs on the left side and then checkout the `Build Configuration`, where it says `Debug` replace it withh `LocalDebug` and where it says `Release` change it to `LocalRelease`, and at the bottom make sure to check the checkbox on `Shared`.

Do this same process for the `Staging` environment as well.

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/Add%20schema%201.png">

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20schema%202.png">

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/add%20schema%203.png">

### Using different bundle id for each environment

Having different bundle id will allow us to install all the apps on the same device, to do that, on the `targets`, select the `multi_build` and go to `build settings` and click on the plus button and click on `Add User-Defined Setting` and name that `BUNDLE_ID_SUFFIX` and just change the name for each build configurations accordingly then on `info.plist`, change the `Bundle identifier` to `$(PRODUCT_BUNDLE_IDENTIFIER).$(BUNDLE_ID_SUFFIX)`

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/bundle%20id%20suffix%201.png">

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/bundle%20id%20suffix%202.png">

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/bundle%20id%20suffix%203.png">

### Displaying different display name for each environment

Go to `Build Settings` again on the `Targets > multi_build` and search for `PRODUCT NAME`, and then just change it accordingly for each environment. Then on `info.plist` change the `Bundle display name` to `$(PRODUCT_NAME)`

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/product%20name%201.png">

<img src="https://github.com/aprilmintacpineda/react-native-multiple-build-environments-example/blob/master/resources/images/product%20name%202.png">

### Running the App

You can change the schema by clicking on the same button that you click to edit schema, but this time just choose from the dropdowns displayed, and then just press run.

###### That's it!