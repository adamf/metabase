# Build Metabase

## Install Prerequisites

These are the tools which are required in order to complete any build of the Metabase code. Follow the links to download and install them on your own before continuing.

1. [Clojure (https://clojure.org)](https://clojure.org/guides/getting_started) - install the latest release by following the guide depending on your OS
2. [Java Development Kit JDK (https://adoptopenjdk.net/releases.html)](https://adoptopenjdk.net/releases.html) - you need to install JDK 11 ([more info on Java versions](./operations-guide/java-versions.md))
3. [Node.js (http://nodejs.org/)](http://nodejs.org/) - latest LTS release
4. [Yarn package manager for Node.js](https://yarnpkg.com/) - latest release of version 1.x - you can install it in any OS by doing `npm install --global yarn`

On a most recent stable Ubuntu/Debian, all the tools above, with the exception of Clojure, can be installed by using:

```
sudo apt install openjdk-11-jdk nodejs && sudo npm install --global yarn
```
If you have multiple JDK versions installed in your machine, be sure to switch your JDK before building by doing `sudo update-alternatives --config java` and selecting Java 11 in the menu

If you are developing on Windows, make sure to use Ubuntu on Windows and follow instructions for Ubuntu/Linux instead of installing ordinary Windows versions.

Alternatively, without the need to explicitly install the above dependencies, follow the guide [on using Visual Studio Code](developers-guide-vscode.md) and its remote container support.

## Build Metabase Uberjar

The entire Metabase application is compiled and assembled into a single .jar file which can run on any modern JVM. There is a script which will execute all steps in the process and output the final artifact for you. You can pass the environment variable MB_EDITION before running the build script to choose the version that you want to build. If you don't provide a value, the default is `oss` which will build the Community Edition.

    ./bin/build

After running the build script simply look in `target/uberjar` for the output .jar file and you are ready to go.

## Building macOS App (`Metabase.app`)

NOTE: These instructions are only for packaging a built Metabase uberjar into `Metabase.app`. They are not useful if your goal is to work on Metabase itself; for development, please see above.

### First-Time Configuration

<details>
<summary>
Steps
</summary>

#### Building

The following steps need to be done before building the Mac App:

1. Install XCode.

1. Add a JRE to the `/path/to/metabase/repo/OSX/Metabase/jre`

   You must acquire a copy of a JRE (make sure you get a JRE rather than JDK) and move it to the correct location in the Mac App source directory so it can be included as part of the Mac App. To ship Java applications as Mac Apps, you must ship them with their own JRE. In this case we want to get a JRE from somewhere (more on this below) and move the `Contents/Home` directory from the JRE archive into `OSX/Metabase/jre`. (`OSX/Metabase` already exists inside the `metabase/metabase` repo.)

   <details><summary>Option 1: Download from AdoptOpenJDK (currently broken -- do not use)</summary>

    You can download a copy of a JRE from https://adoptopenjdk.net/releases.html?jvmVariant=hotspot — make sure you download a JRE rather than JDK. Move the `Contents/Home` directory from the JRE archive into `OSX/Metabase/jre`. (`OSX/Metabase` already exists inside the `metabase/metabase` repo.) For example:

   ```bash
   # IMPORTANT -- DO NOT COPY THIS -- THIS JRE DOESN'T WORK
   cd /path/to/metabase/repo
   wget https://github.com/AdoptOpenJDK/openjdk11-binaries/releases/download/jdk-11.0.8%2B10/OpenJDK11U-jre_x64_mac_hotspot_11.0.8_10.tar.gz
   tar -xzvf OpenJDK11U-jre_x64_mac_hotspot_11.0.8_10.tar.gz
   mv jdk-11.0.8+10-jre/Contents/Home OSX/Metabase/jre
   ```

   **VERY IMPORTANT!**

   Make sure the JRE version you use is one that is known to work successfully with notarization. We have found out the one linked above does not work.
   I have found a nightly build that *does* work, but it's no longer available for download. Cam has a copy of a JRE that is known to work. Refer to Option 2.

   If you get notarization errors like

   > The executable does not have the hardened runtime enabled.

   (Referring to files in `Metabase.app/Contents/Resources/jre/bin/`) then use a different build of the JRE.

   Assuming the OpenJDK folks have resolved this issue going forward, you are fine to use whatever the latest JRE version available is. I have been using the HotSpot JRE instead of the OpenJ9 one but it ultimately shouldn't make a difference.
   </details>

   <details><summary>Option 2: Ask Cam for known working JRE</summary>

    Have Cam ZIP up their `/path/to/metabase/repo/OSX/Metabase/jre` folder and send it to you. Don't try Option 1 until we know the issues are fixed
    </details>
   
1. Copy Metabase uberjar to OSX resources dir (only needed for testing build in Xcode)
   
    Normally the release script does this for you automatically, but for purposes of making sure things are working at this point independently of the release script you should do this yourself just this once. Download a Metabase uberjar from `https://downloads.metabase.com` and copy it to the `Resources` directory:

    ```bash
    cp /path/to/metabase.jar OSX/Resources/metabase.jar
    ```

At this point, you should try opening up the Xcode project and building the Mac App in Xcode by clicking the run button. The app should build and launch at this point. If it doesn't, ask Cam for help!

#### Releasing

The following steps are prereqs for *releasing* the Mac App:

1)  Install XCode command-line tools. In `Xcode` > `Preferences` > `Locations` select your current Xcode version in the `Command Line Tools` drop-down.

1)  Install AWS command-line client (if needed)

    ```bash
    brew install awscli
    ```

1)  Configure AWS Credentials for `metabase` profile (used to upload artifacts to S3)

    You'll need credentials that give you permission to write the metabase-osx-releases S3 bucket.
    You just need the access key ID and secret key; use the defaults for locale and other options.

    ```bash
    aws configure --profile metabase
    ```

1)  Obtain a copy of the private key for signing app updates (ask Cam) and put a copy of it at `OSX/dsa_priv.pem`

    ```bash
    cp /path/to/private/key.pem OSX/dsa_priv.pem
    ```

1)  Add `Apple Developer ID Application Certificate` to your computer's keychain.

    1) Generate a Certificate Signing Request from the Keychain Access app.

        1) `Keychain Access` > `Certificate Assistant` > `Request a Certificate From a Certificate Authority`.

        1) Enter the email associated with your Apple Developer account.

        1) Leave "CA Email Address" blank

        1) Choose "Save to Disk"

    1) Have Cam go to [the Apple Developer Site](https://developer.apple.com/account/mac/certificate/) and generate a `Developer ID Application` certificate for you by uploading the Certificate Signing Request you creating in the last step.

    1) Load the generated certificate on your computer.

1)  Export your Apple ID for building the app as `METABASE_MAC_APP_BUILD_APPLE_ID`. (This Apple ID must be part of the Metabase org in the Apple developer site. Ask Cam to add you if it isn't.)

    ```bash
    #  Add this to .zshrc or .bashrc
    export METABASE_MAC_APP_BUILD_APPLE_ID=my_email@whatever.com
    ```

1)  Create an App-Specific password for the Apple ID in the previous step

    1.  Go to https://appleid.apple.com/account/manage then `Security` > `App-Specific Passwords` > `Generate Password`

    1.  Store the password in Keychain

        ```bash
        xcrun altool \
        --store-password-in-keychain-item "METABASE_MAC_APP_BUILD_PASSWORD" \
        -u "$METABASE_MAC_APP_BUILD_APPLE_ID" \
        -p <secret_password>
        ```

1) Install Clojure CLI. See [the instructions on
clojure.org](https://www.clojure.org/guides/getting_started) for more details.

    ```bash
    brew install clojure/tools/clojure
    ```

</details>

### Building & Releasing the Mac App

After following the configuration steps above, to build and release the app you can use the build script:

1. Make sure release is *published* on GitHub and release notes are ready. The script copies these for the update release notes.

1. Make sure you're on the appropriate release branch locally. The script reads the version number from the most recent tag

1. Bundle entire app, and upload to s3

   ```bash
   ./OSX/release.sh
   ```

   **Important Note** Do not let your computer lock the screen while running the script — if the screen is locked, macOS will not allow the script to access your Apple Developer credentials from the Keychain (needed for notarization) and the script will fail.
