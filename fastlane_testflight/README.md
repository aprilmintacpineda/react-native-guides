# Using fastlane to automated deployments to TestFlight and integrating it with github actions

<img src="./images/completed%20job.png">

# Initialize react-native

```
npx react-native init <app name> --directory example
```

# Using Fastlane

## Initialize fastlane

1. Open up xcode workspace and change `Bundle Identifier` to `com.<yourcompany>.fastlanedeploymentsexample` or wharever your are using.

2. Create a `Gemfile` and add the following contents:

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

6. Run `cd ios`. **You will run all your fastlane related commands from this directory.**

7. Run `bundle exec fastlane init`

8. You will be asked series of questions.

   8.1. **What would you like to use fastlane for?** -- select `3` which is `3. 🚀 Automate App Store distribution`

   8.2. **Parsing your local Xcode project to find the available schemes and the app identifier. Select Scheme:** -- select `1` which is `1. fastlane deployment`

   8.3. **Please enter your Apple ID developer credentials** -- you'll need to login with your apple account where you want to deploy the app to.

   8.4 **Do you want fastlane to create the App ID for you on the Apple Developer Portal? (y/n)** -- if you get asked this, you can answer `y`, it will create an identifier on the developer portal, you can also do this manually -- https://developer.apple.com/account/resources/identifiers/list

   8.5. **Would you like fastlane to create the App on App Store Connect for you? (y/n)** -- if you get asked this, you can enter `y`, which will create an app on `app store connect`, you can also do this manually -- https://appstoreconnect.apple.com/apps

   8.6. **Would you like fastlane to manage your app's metadata? (y/n)** -- if you get asked this, just answer `n`.

   8.7. After that, everything will just be pressing enter.

9. You'll see two files generated inside `fastlane/` named `Appfile` and `Fastfile`.

## Initializing fastlane match

1. Open your xcode workspace and go to `Signing & Capability`, and uncheck `Automatically manaage signing`.

2. Create an **empty** and **private** git repository, this will contain all ios signing files that will be managed by fastlane match.

3. Run `bundle exec fastlane match init`

4. You will be asked series of questions.

   4.1. **fastlane match supports multiple storage modes, please select the one you want to use** -- select `1`, which is `1. git`.

   4.2. **URL of the Git Repo:** -- paste in the ssh connection string of the git repository you just created.

5. Run `bundle exec fastlane match development`, this will create all certificates and other files necessary for you to do local development on an ios device.

6. You will be asked series of questions.

   6.1. **Passphrase for Match storage:** -- enter any string, you'll need to remember this as it will be the passphrase that you'll use to be able to decrypt the generated files.

7. Run `bundle exec fastlane match appstore`, this will create all certificates and other files necessary for you to deploy to testflight. Since this is the second time your run `match`, it remembers your `passphrase` and uses it.

## Setting up xcode project

<img src="./images/setup%20xcode%201.png">

1. For `Code Signing identity`, set `Debug` to `Apple Development` and `Release` to `Apple Distribution` (number 5 in the figure above).

2. For `Provisioning Profile`, set `Debug` to `match Development com.<yourcompany>.fastlanedeploymentsexample` and `Release` to `match AppStore com.<yourcompany>.fastlanedeploymentsexample`. (number 6 in the figure above)

3. If you go to `Signing & Capabilities`, this is what you should have by now.

<img src="./images/setup%20xcode%202.png">

4. Run your app on an iphone device or simulator, it should work just fine.

## Setting up fastlane to build and deploy to TestFlight

1. Update `fastlane/Matchfile` to the following, changing the placeholders with the appropriate information:

```diff
git_url("<your repo here>")

storage_mode("git")

-type("development") # The default type, can be: appstore, adhoc, enterprise or development
+type("appstore") # The default type, can be: appstore, adhoc, enterprise or development

-# app_identifier(["tools.fastlane.app", "tools.fastlane.app2"])
-# username("user@fastlane.tools") # Your Apple Developer Portal username
+app_identifier(["com.<yourcompany>.fastlanedeploymentsexample"])
+username("<your apple account email here>") # Your Apple Developer Portal username
```

If you do [multiple build for each environments](../multi_env_build/README.md), you need to provide all your app identifiers in the `app_identifier`, this case, I only have one.

Example:

```diff
+app_identifier([
  "com.<yourcompany>.fastlanedeploymentsexample",
  "com.<yourcompany>.fastlanedeploymentsexample.staging",
  "com.<yourcompany>.fastlanedeploymentsexample.local"
])
```

2. Update `fastlane/Appfile` to the following:

```diff
-app_identifier("com.aprmp.fastlanedeploymentsexample") # The bundle identifier of your app
apple_id("<your apple account email here>") # Your Apple email address

itc_team_id("<your team id here>") # App Store Connect Team ID
team_id("<your team id here>") # Developer Portal Team ID
```

3. Generate auth key in Apple Development Portal, go to https://appstoreconnect.apple.com/access/api

   3.1. You might need to press on `Request Access Key` for the first time before you can create auth keys.

   3.2. Create key, name it `FastlaneDeploymentCI` and make sure to give it `App Manager` access.

   3.3. Download the key and store it somewhere secured, it will be a `.p8` file. You can only download this file once, if you lost it, you need to generate a new one.

   3.4. For the meantime, copy-paste the `.p8` file to `ios/FastlaneDeploymentCI.p8`. **Make sure to add it in the `.gitignore`**

4. On `fastlane/Fastfile`, replace the existing code with the following, changing the placeholder with the appropriate information.

The image below shows you where you can get your `key id` and `issuer id`.

<img src="./images/setup%20fastlane%201.png">

```ruby
######## IOS CONFIGURATIONS
# If you want to make the build automatically available to external groups,
# add the name of the group to the array below, after "App Store Connect Users"
groups = ["App Store Connect Users"]
workspace = "<app name>.xcworkspace"

# If you build for multiple environments, you might wanna set this specifically on build_app
configuration = 'release'
scheme = "<app name>"

key_id = "<your key id>" # The key id of the p8 file
issuer_id = "<your issuer id>" # issuer id on appstore connect
key_filepath = "FastlaneDeploymentCI.p8" # The path to p8 file generated on appstore connect
######## END IOS CONFIGURATIONS

default_platform(:ios)

platform :ios do
  desc "Push a new build to TestFlight"

  lane :release do
    setup_ci

    api_key = app_store_connect_api_key(
      key_id: key_id,
      issuer_id: issuer_id,
      key_filepath: key_filepath,
      in_house: true
    )

    match(api_key: api_key)

    build_app(
      workspace: workspace,
      scheme: scheme,
      configuration: configuration,
      clean: true,
      silent: true
    )

    upload_to_testflight(
      api_key: api_key,
      groups: groups
    )

    clean_build_artifacts
  end
end
```

5. [Setup app icons.](../setup_appicon/README.md)

6. Let's build and deploy to testflight.

   6.1. Run `yarn start`. We need it for the `main.jsbundle`.

   6.2. On another terminal, run `bundle exec fastlane ios release`

# Integration with Github Actions

## Setup deploy keys for match repository

1. Go to your match repository, then on `settings` > `Deploy keys`.

2. Generate ssh key pair: `ssh-keygen -t rsa -b 4096 -C "<your apple email here>"`, and save it where ever you like, we're only going to use this once. **Just leave the passphrase empty**.

3. Add deploy key.

4. Run `cat path/to/the/key.pub` (the public key you generated a while ago) and copy the contents of that file to the value of the secret, then `save` it, the name is optional.

## Setup github actions secrets for the main repository

1. Create a secret called `MATCH_PASSWORD` and put in your `match passphrase`, the one you entered a while ago, as the value.

2. Run `cat path/to/the/key` (the private key you generate a while ago) and then create a secret called `MATCH_REPO_KEY` and copy-paste the content of the private key to the value.

3. Create a secret called `P8_AUTH_KEY` and copy the contents of `FastlaneDeploymentCI.p8` to that.

4. Create a secret called `APPSTORE_KEY_ID` and put your `key id` as the value.

5. Create a secret called `APPSTORE_ISSUER_ID` and put your `issuer id` as the value.

6. Create a secret called `APPSTORE_TEAM_ID` and put your `team_id` as the value, you can find it in `fastlane/Appfile`.

7. Create a secret called `ITUNES_TEAM_ID` and put your `itc_team_id` as the value, you can find it in `fastlane/Appfile`.

8. Create secret called `APPLE_ID` and put your `apple_id` as the value, you can find it in `fastlane/Appfile`.

9. Create secret called `MATCH_REPO_SSH` and put your `git_url` as the value, you can find it in `fastlane/Matchfile`

## Update Fastfile

Update your `fastlane/Fastfile` to the following:

```diff
######## IOS CONFIGURATIONS
# If you want to make the build automatically available to external groups,
# add the name of the group to the array below, after "App Store Connect Users"
groups = ["App Store Connect Users"]
workspace = "<app name>.xcworkspace"

configuration = 'release' # configuration to build, if you build for multiple envs
scheme = "<app name>" # scheme to build, if you build for multiple schema

-key_id = "YRR68P9HUT" # The key id of the p8 file
-issuer_id = "1f560005-2d5f-47f8-9678-f41b2fdd8769" # issuer id on appstore connect
+key_id = ENV["APPSTORE_KEY_ID"] # The key id of the p8 file
+issuer_id = ENV["APPSTORE_ISSUER_ID"] # issuer id on appstore connect
key_filepath = "FastlaneDeploymentCI.p8" # The path to p8 file generated on appstore connect
######## END IOS CONFIGURATIONS
```

## Update Matchfile

```diff
-git_url("<your repo here>")
+git_url(ENV["MATCH_REPO_SSH"])

storage_mode("git")

type("appstore") # The default type, can be: appstore, adhoc, enterprise or development

app_identifier(["com.aprmp.fastlanedeploymentsexample"])
-username("<your apple account email here>") # Your Apple Developer Portal username
+username(ENV["APPLE_ID"]) # Your Apple Developer Portal username
```

## Update Appfile

```diff
-apple_id("<your apple account email here>") # Your Apple email address
+apple_id(ENV["APPLE_ID"]) # Your Apple email address

-itc_team_id("<your team id here>") # App Store Connect Team ID
-team_id("<your team id here>") # Developer Portal Team ID
+itc_team_id(ENV["ITUNES_TEAM_ID"]) # App Store Connect Team ID
+team_id(ENV["APPSTORE_TEAM_ID"]) # Developer Portal Team ID
```

## Setup github actions workflow

6. Create the file `./github/workflows/ios-deployment.yaml`

7. On your `package.json`, add the following to scripts:

```diff
"scripts": {
  "android": "react-native run-android",
  "ios": "react-native run-ios",
  "start": "react-native start",
  "test": "jest",
  "lint": "eslint .",
+  "bundle-js": "react-native bundle --entry-file index.js --bundle-output ios/main.jsbundle"
},
```

7. Add the following code in it:

```yaml
name: ios-deployment

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
    runs-on: macos-latest
    continue-on-error: false
    env:
      MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
      APPSTORE_KEY_ID: ${{ secrets.APPSTORE_KEY_ID }}
      APPSTORE_ISSUER_ID: ${{ secrets.APPSTORE_ISSUER_ID }}
      APPLE_ID: ${{ secrets.APPLE_ID }}
      ITUNES_TEAM_ID: ${{ secrets.ITUNES_TEAM_ID }}
      APPSTORE_TEAM_ID: ${{ secrets.APPSTORE_TEAM_ID }}
      MATCH_REPO_SSH: ${{ secrets.MATCH_REPO_SSH }}

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@master

      - uses: webfactory/ssh-agent@v0.5.0
        with:
          ssh-private-key: ${{ secrets.MATCH_REPO_KEY }}

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

      - uses: actions/cache@v2
        with:
          path: Pods
          key: ${{ runner.os }}-pods-${{ hashFiles('ios/Podfile.lock') }}
          restore-keys: |
            ${{ runner.os }}-pods-

      - name: Versions
        run: |
          echo "Yarn: $(yarn --version)"
          echo "Node: $(node --version)"
          echo "Ruby: $(ruby --version)"
          echo "Bundler: $(bundle --version)"

      - name: Setup dependencies
        run: |
          yarn
          cd ios
          pod install
          echo "${{ secrets.P8_AUTH_KEY }}" > FastlaneDeploymentCI.p8

      - name: Build and deploy
        run: |
          yarn bundle-js
          cd ios
          bundle exec fastlane ios release
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

# Related reading materials

- [Setup App Icons](../setup_appicon)
- [Using fastlane to automated deployments to PlayStore and integrating it with github actions](../fastlane_playstore)
- Getting started with fastlane for React Native http://docs.fastlane.tools/getting-started/cross-platform/react-native/
