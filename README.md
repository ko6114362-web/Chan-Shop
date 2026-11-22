
name: Build Chan Shop APK

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    env:
      ANDROID_HOME: /usr/local/lib/android/sdk

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Setup Android SDK
        run: |
          echo "$ANDROID_HOME/cmdline-tools/16.0/bin" >> $GITHUB_PATH
          echo "$ANDROID_HOME/platform-tools" >> $GITHUB_PATH
          yes | "$ANDROID_HOME/cmdline-tools/16.0/bin/sdkmanager" --sdk_root="$ANDROID_HOME" --licenses
          "$ANDROID_HOME/cmdline-tools/16.0/bin/sdkmanager" --sdk_root="$ANDROID_HOME" "platforms;android-35" "build-tools;35.0.0"

      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.9'

      - name: Build APK
        run: |
          unzip -q ChanShopAndroid_KBZPay_Project.zip -d source
          gradle -p source/ChanShopAndroid assembleDebug

      - uses: actions/upload-artifact@v4
        with:
          name: ChanShop-APK
          path: source/ChanShopAndroid/app/build/outputs/apk/debug/app-debug.apk
