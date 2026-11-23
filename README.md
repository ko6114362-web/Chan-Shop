
name: Build Chan Shop APK

on:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Setup Android SDK
        uses: android-actions/setup-android@v4
        with:
          packages: |
            platforms;android-35
            build-tools;35.0.0

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.9'

      - name: Extract Android Project
        run: |
          unzip -q ChanShopAndroid_KBZPay_Project.zip -d source

      - name: Fix Java Kotlin Version
        run: |
          GRADLE_FILE=$(find source -path "*/app/build.gradle.kts" | head -n 1)

          if [ -z "$GRADLE_FILE" ]; then
            echo "ERROR: app/build.gradle.kts not found"
            exit 1
          fi

          echo "Found: $GRADLE_FILE"

          python3 - "$GRADLE_FILE" <<'PY'
          import sys
          import re

          path = sys.argv[1]

          with open(path, "r") as f:
              text = f.read()

          java_config = """
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
          """

          if "compileOptions" in text:
              text = re.sub(
                  r'compileOptions\s*\{.*?\}',
                  java_config.strip(),
                  text,
                  flags=re.S
              )
          else:
              text = text.replace(
                  "android {",
                  "android {\n" + java_config,
                  1
              )

          kotlin_config = """
          kotlinOptions {
              jvmTarget = "17"
          }
          """

          if "kotlinOptions" in text:
              text = re.sub(
                  r'kotlinOptions\s*\{.*?\}',
                  kotlin_config.strip(),
                  text,
                  flags=re.S
              )
          else:
              text = text.replace(
                  "android {",
                  "android {\n" + kotlin_config,
                  1
              )

          with open(path, "w") as f:
              f.write(text)

          print("Java and Kotlin target set to 17")
          PY

      - name: Build APK
        run: |
          PROJECT_DIR=$(find source -name settings.gradle.kts -o -name settings.gradle | head -n 1 | xargs dirname)

          if [ -z "$PROJECT_DIR" ]; then
            echo "ERROR: Gradle project not found"
            exit 1
          fi

          echo "Project: $PROJECT_DIR"

          gradle -p "$PROJECT_DIR" assembleDebug \
            -Dorg.gradle.java.home="$JAVA_HOME" \
            --stacktrace

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: ChanShop-APK
          path: source/**/app/build/outputs/apk/debug/*.apk
          if-no-files-found: error
