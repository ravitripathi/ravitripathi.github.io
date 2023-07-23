---
title: Analyzing .ipa files - Instagram's Threads app
date: 2023-07-23 18:30:00 +0530
tags: [iOS, IPA, frameworks]
---

Sometime back, there was a thread on Twitter about how some iOS apps ended up leaking API keys via `Info.plist` files present in the app bundle. [Link](https://twitter.com/RhoTau/status/1647209380826451968?s=20)

![Tweet](/assets/postAssets/TweetScreenshot.png)

Due to the nature of how apps are delivered on the App Store, and how in a vast majority of workflows, developers never need to interact with the generated `.ipa` files for their projects, the structure of the compiled app is not very well-known nowadays. So, I decided to try out exploring and documenting the structure of the newest popular app - Instagram's Threads App.

## Acquiring Threads' `.ipa`

This is a surprisingly convoluted process. Unlike Android's `.apk` files, the distribution of `ipa` files is not exceedingly common, as unsigned `.ipa` files can not be installed on devices, and you can not simply copy over `ipa` from a physical device. Local backups for iOS devices made by macOS do contain the installed applications. However, the backups are encrypted.

Luckily, Apple provides APIs for interacting with the App Store. [Majd Alfhaily's](https://github.com/majd) [ipatool](https://github.com/majd/ipatool) allows users to search and download ipa files from App Store, after authenticating with their Apple ID. 

After authenticating with `ipatool auth login --email <APP_STORE_EMAIL>`, I searched for the Threads app, in order to obtain its bundle ID:

![ipatool search output](/assets/postAssets/threadsoutput.png)

Bingo! So Threads' bundle ID is `com.burbn.barcelona`. Bundle IDs are usually reversed domain names, followed by the name of the application. Instagram was [originally called "Burbn"](https://www.theatlantic.com/technology/archive/2014/07/instagram-used-to-be-called-brbn/373815/), which explains the domain name. `barcelona` could probably be the internal code name for the Threads App.

Now, once the bundle ID is known, the `.ipa` can be downloaded using ipatool:

![ipatool download output](/assets/postAssets/ipatooldownload.png)

## Extracting the Application from the package

This is quite straightforward. Just change the extension of the downloaded `.ipa` to `.zip`, and extract it. The application shows up under the `Payload` folder in the extracted directory.

## Inspecting contents of the application

Right-click on the application "Barcelona", and select "Show Package Contents"

![Application Content](/assets/postAssets/applicationcontent.png)

We finally arrived at the contents of the application. Lot of interesting items lying around.

![Package Contents](/assets/postAssets/appcontents.png)

### Assets

Loads of assets. First up, we see the compiled contents of `xcassets`, which show up under `Assets.car`. While there is no direct way of accessing the contents inside this file, [Samra](https://github.com/SerenaKit/Samra), an open-source macOS app, makes it a breeze to inspect the contents of Asset Catalogues. 

![Asset Contents](/assets/postAssets/assetcontents.png)

It contains the usual suspects (icons used across the app), along with some icons indicating payment-related features.

A lot of font files as well. The Cosmopolitan Script looks pretty interesting.

![Fonts](/assets/postAssets/threadfonts.png)

### Libraries and Frameworks

This did not yield a lot of interesting results. The app uses *no dynamic frameworks*, so the `Frameworks` folder contains nothing of significance. This implies that the preferred choice of linking 3rd party libraries in the app is static linking, a trend which is becoming increasingly common, due to the benefits of shorter app launch times. We can expect static libraries in production builds to become more prevalent with the [upcoming mergeable libraries feature in Xcode 15](https://medium.com/@SanjuNaik14/meet-mergeable-libraries-790a40aa89b8).

### Other interesting tidbits

Localizations. Loads of them. 36 lproj files

There are signs of a lot of *native* iOS APIs, which is surprising for Meta. Some of them include:

- #### Presence of Metal Shaders
![Metal Shaders](/assets/postAssets/metalshaders.png)

There are a bunch of OpenGL shaders as well.

- #### CoreML models. The names of some of these models might be indicative of what they do. (Notice the one for Reels Tab)
![CoreML](/assets/postAssets/coremlmodels.png)

Quite interesting to see Threads leveraging *native on-device ML APIs* for optimizing their feeds.

- #### Info.plist file

The `Info.plist` file is an important entity in an application, serving multiple purposes. For instance, permission prompt messages reside here. The CI/CD setup in Meta might also require certain pieces of information to be passed onto the plist file. A key `FBBuildBranchName` exists, with the value `fbobjc/releases/release-igios-2023.07.17`. 
![Plist](/assets/postAssets/plistfile.png)

The `DTXcode` value points to `1431`, which implies the Xcode version used for building this application was `14.3.1`.

## Conclusion

The structure of compiled applications makes it easy to unzip and pry them open. Assets are quite easy to extract as well. Plist files are simply copied over to the bundle, so it is **never** a good idea to store sensitive information, like API keys in it. 