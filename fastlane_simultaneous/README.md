# Simultaneously deploying to PlayStore and TestFlight

<img src="./images/jobs%20completed.png">

The only thing you need to do to simultaneously deploy to TestFlight and PlayStore is to have two different github workflows, one for android and one for iOS. If you follow both guides, one after the other, you should already have achieved this.

```
/ root
|- .bundle
|  |- config
|- .github
|  |- workflows
|  |  |- android-deployment.yaml
|  |  |- ios-deployment.yaml
|- android
|  |- fastlane
|  |  |- Appfile
|  |  |- Fastfile
|  |  |- README.md
|- ios
|  |- fastlane
|  |  |- Appfile
|  |  |- Fastfile
|  |  |- README.md
|- Gemfile
|- Gemfile.lock
```

# Ruby

## Gemfile

```gemfile
source "https://rubygems.org"

gem "fastlane"
```

## .bundle/config

```
---
BUNDLE_SET: "path ./vendor/bundle"
```

# Github workflows

## .github/workflows/android-deployment.yaml

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
          echo '${{ secrets.ANDROID_GOOGLE_KEYS }}' > android/google-secret-key.json
          echo "${{ secrets.ANDROID_SIGNING_KEY }}" > my-upload-key.keystore.asc
          gpg -d --passphrase "${{ secrets.ANDROID_SIGNING_KEY_PASSPHRASE }}" --batch my-upload-key.keystore.asc > android/app/my-upload-key.keystore
          yarn

      - name: Build and deploy
        run: |
          cd android
          bundle exec fastlane android internal
```

## .github/workflows/ios-deployment.yaml

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
          echo "${{ secrets.P8_AUTH_KEY }}" > FastlaneDeploymentCI.p8
          yarn
          cd ios
          pod install

      - name: Build and deploy
        run: |
          yarn bundle-js
          cd ios
          bundle exec fastlane ios release
```

# iOS Fastlane files

## ios/fastlane/Appfile

```ruby
apple_id(ENV["APPLE_ID"]) # Your Apple email address

itc_team_id(ENV["ITUNES_TEAM_ID"]) # App Store Connect Team ID
team_id(ENV["APPSTORE_TEAM_ID"]) # Developer Portal Team ID
```

## ios/fastlane/Matchfile

```ruby
git_url(ENV["MATCH_REPO_SSH"])

storage_mode("git")

type("appstore") # The default type, can be: appstore, adhoc, enterprise or development

app_identifier(["com.aprmp.fastlanedeploymentsexample"])
username(ENV["APPLE_ID"]) # Your Apple Developer Portal username
```

## ios/fastlane/Fastfile

```ruby
# This file contains the fastlane.tools configuration
# You can find the documentation at https://docs.fastlane.tools
#
# For a list of all available actions, check out
#
#     https://docs.fastlane.tools/actions
#
# For a list of all available plugins, check out
#
#     https://docs.fastlane.tools/plugins/available-plugins
#

# Uncomment the line if you want fastlane to automatically update itself
# update_fastlane

######## IOS CONFIGURATIONS
# If you want to make the build automatically available to external groups,
# add the name of the group to the array below, after "App Store Connect Users"
groups = ["App Store Connect Users"]
workspace = "fastlane_deployment.xcworkspace"

configuration = 'release' # configuration to build, if you build for multiple envs
scheme = "fastlane_deployment" # scheme to build, if you build for multiple schema

key_id = ENV["APPSTORE_KEY_ID"] # The key id of the p8 file
issuer_id = ENV["APPSTORE_ISSUER_ID"] # issuer id on appstore connect
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
      clean: true,
      configuration: configuration
    )

    upload_to_testflight(
      api_key: api_key,
      groups: groups
    )

    clean_build_artifacts
  end
end
```

# Android Fastlane files

## android/fastlane/Appfile

```ruby
json_key_file("google-secret-key.json") # Path to the json secret file - Follow https://docs.fastlane.tools/actions/supply/#setup to get one
```

## android/fastlane/Fastfile

```ruby
# This file contains the fastlane.tools configuration
# You can find the documentation at https://docs.fastlane.tools
#
# For a list of all available actions, check out
#
#     https://docs.fastlane.tools/actions
#
# For a list of all available plugins, check out
#
#     https://docs.fastlane.tools/plugins/available-plugins
#

# Uncomment the line if you want fastlane to automatically update itself
# update_fastlane

default_platform(:android)

platform :android do
  desc "Build and deploy to internal track"
  lane :internal do
    gradle(task: "clean bundleRelease")

    upload_to_play_store(
      package_name: "com.aprmp.fastlane_deployment",
      track: "internal",
      aab: "app/build/outputs/bundle/release/app-release.aab",
      release_status: "draft"
    )

    # sh "your_script.sh"
    # You can also use other beta testing services here
  end

  desc "Build and deploy to production"
  lane :production do
    gradle(task: "clean bundleRelease")

    upload_to_play_store(
      package_name: "com.aprmp.fastlane_deployment",
      aab: "app/build/outputs/bundle/release/app-release.aab"
    )
  end
end
```
