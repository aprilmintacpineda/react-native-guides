# Using fastlane to automated deployments to PlayStore and integrating it with github actions

<img src="./images/completed%20job.png">

# Initialize react-native

```
npx react-native init <app name> --directory example
```

# Setup project

## Setup Google Cloud Platform (GCP) Project

### Create project

1. Got to https://console.cloud.google.com/home/dashboard and follow the steps in the screenshots below.

<img src="./images/gcp%20create%20project%201.png">

<img src="./images/gcp%20create%20project%202.png">

### Enable Google Play Android Developer API

Enabling this API will allow you to link this GCP project to your Google Playstore console.

<img src="./images/gcp%20enable%20play%20api%201.png">

<img src="./images/gcp%20enable%20play%20api%202.png">

<img src="./images/gcp%20enable%20play%20api%203.png">

### Create Service Account

<img src="./images/gcp%20create%20credentials%201.png">

<img src="./images/gcp%20create%20credentials%202.png">

<img src="./images/gcp%20create%20credentials%203.png">

Then just fill in the required fields.

On the `Grant this service account access to project`, make sure to select `Service accounts > Service Account User` as the role.

<img src="./images/gcp%20create%20credentials%204.png">

<img src="./images/gcp%20create%20credentials%205.png">

### Generate key

1. Click on the service account you just created, and follow the images below.

**NOTE:** You need to download the generated key right after it has been created, make sure to store it somewhere secured.

<img src="./images/gcp%20create%20key%201.png">

<img src="./images/gcp%20create%20key%202.png">

## Link Google Cloud Platform project to Google Play Console

<img src="./images/link%20gcp%20to%20play%20console.png">

Then on the `Service Accounts`, you should see the service account that you created.

## Granting access to Service Account

### App Permissions

<img src="./images/grant%20access%201.png">

<img src="./images/grant%20access%202.png">

Once you have selected your app, click on apply, and you'll see the permissions, make sure that the releases are all check, some of these may already be checked by default.

### Account permissions

You can uncheck everything here.

Then click on `Invite User` button when you're cone.

# Using Fastlane

## Initialize fastlane

1. Go to `/android/app/build.gradle` and make sure to update your `applicationId`.

2. Create `Gemfile` on the root, and add the following contents:

```gemfile
source "https://rubygems.org"

gem "fastlane"
```

3. Run `bundle config --local set path "./vendor/bundle"`

4. Add the following to your `.gitignore`

```
# Ruby
vendor/
```

5. Run `bundle`

6. Run `cd android`. **You will run all fastlane related commands in this directory**.

7. Run `bundle exec fastlane init`

8. You will be asked series of questions

   8.1. **Package Name (com.krausefx.app):** -- provide your `applicationId`

   8.2. **Path to the json secret file:** -- enter `google-secret-key.json`

   8.3. ** Download existing metadata and setup metadata management? (y/n)** -- select `n`

   8.4. The rest are just notes, just press enter all the way to the end.

## Verify connection to Google Play Console

1. Copy the google secret key that you generated and downloaded on the root directory of the project and name it `google-secret-key.json`. **Make sure to add it to your `.gitignore`.**

2. Run `bundle exec fastlane run validate_play_store_json_key json_key:google-secret-key.json`

## Deploying to PlayStore Internal Track with Fastlane

1. First, you need to create a release in `Internal Testing` and manually upload your first build before you can automatically upload builds using fastlane. You will need to follow `https://reactnative.dev/docs/signed-apk-android` to generate a key that you will use to sign your builds.

1. On your `Appfile`, change the following:

```diff
default_platform(:android)

platform :android do
- desc "Runs all the tests"
- lane :test do
-   gradle(task: "test")
- end

- desc "Submit a new Beta Build to Crashlytics Beta"
+ desc "Build and deploy to internal track"
- lane :beta do
+ lane :internal do
-   gradle(task: "clean assembleRelease")
+   gradle(task: "clean bundleRelease")
-   crashlytics
+   upload_to_play_store(
+     track: "internal",
+     aab: "app/build/outputs/bundle/release/app-release.aab"
+     release_status: "draft"
+   )
  end

- desc "Deploy a new version to the Google Play"
+ desc "Build and deploy to production"
  lane :production do
-   gradle(task: "clean assembleRelease")
+   gradle(task: "clean bundleRelease")
-   upload_to_play_store
+   upload_to_play_store(
+     aab: "app/build/outputs/bundle/release/app-release.aab"
+   )
  end
end
```

Notice that in the above script, `release_status` is set to `draft`, this is because in my case, the app I created is still in "draft" status. If yours in not in draft status, you can remove this argument.

<img src="./images/app%20status.png">

2. In `Appfile`

```diff
json_key_file("google-secret-key.json") # Path to the json secret file - Follow https://docs.fastlane.tools/actions/supply/#setup to get one
-package_name(<your application id>) # e.g. com.krausefx.app
```

3. You can now run `bundle exec fastlane android internal`

# Integration with github actions

## Storing signing key in github secrets

You will need to upload the signing key that you generated, but github secrets does not support file, so we will use `gpg` to transform it into a string that represents a file, and we will also secure it with a passphrase.

```
gpg -c --armor app/my-upload-key.keystore
```

This will produce `my-upload-key.keystore.asc` which contains a string that we can copy-paste into github secrets.

## Create github secrets

1. Create a secret called `ANDROID_SIGNING_KEY` and copy-paste the contents of `my-upload-key.keystore.asc`.

2. Create a secret called `ANDROID_SIGNING_KEY_PASSPHRASE` and put the passphrase you used in `gpg`.

3. Create a secret called `ANDROID_GOOGLE_KEYS` and copy-paste the contents of `google-secret-key.json`.

## Github Actions Workflow

Create `.github/workflows/android-deployment.yaml` and add the following contents:

```yaml
name: android-deployment

# Controls when the action will run. Triggers the workflow on push or pull request
# events but only for the dev branch
on:
  release:
    types: [published]

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest
    continue-on-error: false

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@master

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: 2.6
          bundler-cache: true

      - name: Get yarn cache directory path
        id: yarn-cache-dir-path
        run: echo "::set-output name=dir::$(yarn cache dir)"

      - uses: actions/cache@v2
        with:
          path: ${{ steps.yarn-cache-dir-path.outputs.dir }}
          key: ${{ runner.os }}-yarn-${{ hashFiles('yarn.lock') }}
          restore-keys: |
            ${{ runner.os }}-yarn-

      - name: Setup dependencies
        run: |
          yarn
          echo '${{ secrets.ANDROID_GOOGLE_KEYS }}' > android/google-secret-key.json
          echo "${{ secrets.ANDROID_SIGNING_KEY }}" > my-upload-key.keystore.asc
          gpg -d --passphrase "${{ secrets.ANDROID_SIGNING_KEY_PASSPHRASE }}" --batch my-upload-key.keystore.asc > android/app/my-upload-key.keystore

      - name: Build and deploy
        run: |
          cd android
          bundle exec fastlane android internal
```

This github action workflow will run when you publish a new release. So after you pushed, you need to go to `releases` on your repository and then publish a new release.

# Optional: Handling version changes

When deploying, you can use `npm version [target]` and then have the versions of android and ios automatically incremented as well.

1. `yarn add react-native-version --dev`

2. On your `package.json`, add the following to the scripts:

```diff
"scripts": {
  "android": "react-native run-android",
  "ios": "react-native run-ios",
  "start": "react-native start",
  "test": "jest",
  "lint": "eslint .",
+  "postversion": "react-native-version",
  "bundle-js": "react-native bundle --entry-file index.js --bundle-output ios/main.jsbundle"
},
```

# Credits

- App icon I used here came from https://www.flaticon.com/free-icon/rocket_1541396?term=deployment&page=1&position=3&page=1&position=3&related_id=1541396&origin=search
- I learned the `gpg` trick from this guide https://medium.com/@StefMa/how-to-store-a-android-keystore-safely-on-github-actions-f0cef9413784

# Related reading materials

- [Setup App Icons](../setup_appicon)
- [Using fastlane to automated deployments to TestFlight and integrating it with github actions](../fastlane_testflight)
- Getting started with fastlane for React Native http://docs.fastlane.tools/getting-started/cross-platform/react-native/
