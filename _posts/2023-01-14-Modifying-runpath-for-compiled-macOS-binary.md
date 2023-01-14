---
title: Modifying Runpath for a compiled swift macOS binary
date: 2023-01-14 19:30:00 +0530
tags: [swift, SPM, Command Line Tools]
---

While working on moving an [internal command line tool to a pure SPM project](/2022-03-12-Moving-a-Command-Line-Tool-project-to-Swift-Packages.md), I encountered an interesting problem. 

We use [SwiftSyntax](https://github.com/apple/swift-syntax) in our CLI tool. Adding the package as a dependency is quite straightforward. SPM also resolves SwiftSyntax's dependencies as well for us, one of which is a dynamic library, `lib_InternalSwiftSyntaxParser.dylib`.

The releases for SwiftSyntax are tied to releases of Swift Language itself, as it provides several tools for parsing and transforming Swift code. `lib_InternalSwiftSyntaxParser.dylib` was bundled with Xcode itself, found under `/path_to_Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/macosx`. Just like with dynamic frameworks, `.dylib`s are not compiled into the final binary. Instead, they are added to the `RPATH`, the runtime search path of your compiled executable. So when your executable runs, it looks for the `.dylib`/`.framework` in the specified search path, and then continues its execution.

Now if you are familiar with mac/iOS development, this isn't a problem for a packaged `.ipa`/`.app` App. Because when you ship an app, any 3rd party dynamic framework is copied over to the `Frameworks` folder in the app, making it convenient to ship libraries de-coupled from the main app executable, but still accessible within the app's bundle.

![PackageContents](/assets/postAssets/packagecontents.png)

So why can't we pull the same off with a CLI tool? What happens if you try to distribute it? Your users will be greeted with this message:

```bash
dyld: Library not loaded: @rpath/lib_InternalSwiftSyntaxParser.dylib
```

Because the path against which your executable was linked against would simply not exist elsewhere. A CLI tool written in Swift is just a UNIX executable. There is no package which wraps it. While you can ship both the `framework/dylib` and the main executable together, and it would still work as long as they are in the same folder, it's far from ideal. Futhermore, this makes it complicated to use a package manager like `Homebrew` to ship your CLI tool.

This would be an easy fix if we could get a static library version of `lib_InternalSwiftSyntaxParser.dylib`, since you could just embed it in your final binary. Sadly, it was not shipped separately. However, what I *did* know for certain is that our tool would be run on machines which had Xcode installed. What if, I could extract the path to the dylib, and somehow add it to our CLI tool's `RPATH`, so that `dyld` finds the library at runtime?

Locating the path to the Xcode installation is straightforward. Running `xcode-select -p` gives:

```bash
/Applications/Xcode.app/Contents/Developer  #Or any other path where you have installed Xcode
```

Alright, so assuming we ship a compiled binary, and can add a post-install instruction with it (eg, running a custom script), we can install our binary on any machine, and extract the installation path to Xcode. But how do I update the Runtime Search Path for our executable? Enter `install_name_tool`:

![install_name_tool man page](/assets/postAssets/install_name_tool.png)

`install_name_tool` can modify the search paths in a MACH-O binary. So, running a post-install script like this one looks for the Xcode installation on the end-user's machines, and adds its path to `RPATH` 🚀:

```bash
#!/bin/sh
XCODEPATH=$(xcode-select -p)
LIBSYNTAXPATH=$XCODEPATH/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/macosx/ 

if [ -z "$XCODEPATH" ] ; then
    echo "Xcode not installed. Please install Xcode to continue."
    # remove installed binary (sonar) as well
    rm -rf sonar 
    exit -1

    else
    echo "Setting Xcode Toolchain path for sonar"
    #### install_name_tool ######
    install_name_tool -add_rpath $LIBSYNTAXPATH sonar
fi
```

### Epilogue

`install_name_tool` served me well. However, [SwiftSyntax](https://github.com/apple/swift-syntax) now provides a standalone `.framework` for `lib_InternalSwiftSyntaxParser`, instead of relying on Xcode. 

Also, [there is now a distribution of `lib_InternalSwiftSyntaxParser.dylib` built statically](https://github.com/keith/StaticInternalSwiftSyntaxParser), which allowed embedding the entirety of this library within the executable. I ended up using it in favour of messing around with `RPATH` after installation.

All that said, it was fun figuring out these details. If you want to read more about `RPATH` and `install_name_tool`, I recommend checking out this [excellent blog post by Marcin Krzyżanowski.](https://blog.krzyzanowskim.com/2018/12/05/rpath-what/)







