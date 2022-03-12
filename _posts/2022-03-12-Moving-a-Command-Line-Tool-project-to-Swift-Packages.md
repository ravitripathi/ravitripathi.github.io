---
title: Moving a Command Line Tool project to Swift Packages
date: 2022-03-12 18:30:00 +0530
tags: [swift, SPM, Command Line Tools]
---
![Swift Package](/assets/postAssets/swiftpackage.jpeg)

Creating an executable Swift Package is a great way to quickly build a command line tool for folks familiar with the language. [Antoine van der Lee](https://www.avanderlee.com) has written a [great article](https://www.avanderlee.com/swift/command-line-tool-package-manager/) on getting started with one.

I recently moved an internal [command line tool](https://www.gojek.io/blog/using-custom-lint-rules-to-improve-our-dls-coverage) which we use here at Gojek to a Swift Package from an Xcode Project. Moving away from an Xcode project presented some difficulties, requiring some workarounds which I would talk about in this post. But before jumping into that, let's address a common query.

### But why not stick to an Xcode Command Line Target?

Using SPM for building a command line tool instead of an Xcode Command Line target has some benefits:

- Since the Swift Toolchain is [available on Windows and Linux](https://www.swift.org/getting-started/), you can compile your Swift Package on these platforms without any Xcode-specific tools. Use `swift build` and `swift test` to build and test a swift package respectively.

- Adding test cases for Command Line Tool target in Xcode [has some caveats](https://developer.apple.com/forums/thread/52211), and often involves creating a separate dynamic framework for enclosing your core logic so that it can be unit tested independently. This is not a problem with Swift Packages, where a test target can be added for an executable.

A command line tool is usually a single compiled executable, and having an additional `.framework` carrying the core logic makes it difficult to distribute. It also makes further optimizations difficult. 

However, tools associated with Xcode are more mature when compared to SPM, which can make it difficult to migrate projects.

### Moving to SPM

This was really straightforward. In a new directory, I ran: 

```
swift package init --type executable <Your_Executable_Name>
```

This generates the `Sources` and `Tests` directory, a `README.md` and a `Package.swift` file. All I had to do was move the project files into `Sources` and the tests into `Tests` directory respectively. 

Opening the `Package.swift` launches Xcode. After adding the dependencies in the `Package.swift` file, I was good to go.


### Coverage Reports for Swift Packages

While Xcode displays code coverage percentage for both SPM-based and `.xcodeproj` projects within the IDE, many teams rely on tools like [xcov](https://github.com/fastlane-community/xcov) to generate test coverage reports, and even fail CI/CD pipelines if the total coverage drops below a certain limit. This is where I hit my first wall. 

Unlike Xcode Projects, no `.xccoverage` files are generated while running tests for a Swift Package. Instead, SPM generates a `.json` file containing the coverage data, which is not compatible with Xcov. I had no way to check the coverage percentage on a CI job.

Thankfully, I stumbled upon [swift-test-codecov](https://github.com/mattpolzin/swift-test-codecov), another command line swift package which can parse this json and generate a table containing coverage results. Coupling this with [Mint](https://github.com/yonaskolb/Mint) allowed me to parse the coverage report on CI:

<center>
<iframe
  src="https://carbon.now.sh/embed?bg=rgba%28171%2C+184%2C+195%2C+1%29&t=seti&wt=none&l=application%2Fx-sh&width=850&ds=true&dsyoff=20px&dsblur=68px&wc=true&wa=false&pv=56px&ph=56px&ln=false&fl=1&fm=Hack&fs=13px&lh=133%25&si=false&es=2x&wm=false&code=%253E%2520mint%2520run%2520swift-test-codecov%2520.%252F.build%252Fdebug%252Fcodecov%252FSonar.json%2520-p%2520table%2520--removetestfiles%250A%250A%25F0%259F%258C%25B1%2520Finding%2520latest%2520version%2520of%2520swift-test-codecov%250A%25F0%259F%258C%25B1%2520Running%2520swift-test-codecov%25200.10.2...%250A%250AOverall%2520Coverage%253A%252065.44%2525%250A%250AFile%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520Coverage%250A------------------------------------%2520--------%250A%253CREDACTED_NAME%253E.swift%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%2520%252097.37%2525%250A....."
  style="width: 900px; height: 373px; border:0; transform: scale(1); overflow:hidden;"
  sandbox="allow-scripts allow-same-origin">
</iframe>
</center>

### Exporting the final executable

This was the easy part. `swift build --configuration release` builds the project and generates the executable inside the `.build/release` directory. And that's it! Copy the executable to `/usr/local/bin` and you are good to go.

However, even on release configuration, the final executable size was well above 12 MB. Quite high for a command line utility. I began looking for ways to compress an executable

### UPX FTW 🚀

[upx](https://github.com/upx/upx) is powerful tool which can compress executables on Windows, macOS and Linux. This was perfect for my use case. Running upx brought down the final executable size from `13.7 MB` to `3.2 MB`, a 76% decrease!!

### Ship it 📦

For installing locally, a shell script can encapsulate all the steps needed for building and moving the final executable to `/usr/local/bin`:
<center>
<iframe
  src="https://carbon.now.sh/embed?bg=rgba%28171%2C+184%2C+195%2C+1%29&t=cobalt&wt=none&l=application%2Fx-sh&width=858&ds=true&dsyoff=20px&dsblur=68px&wc=true&wa=false&pv=56px&ph=56px&ln=false&fl=1&fm=Hack&fs=13px&lh=133%25&si=false&es=2x&wm=false&code=%2523%21%252Fbin%252Fsh%250A%250Aswift%2520build%2520--configuration%2520release%250Aupx%2520.build%252Frelease%252F%253CYOUR_EXECUTABLE%253E%250Acp%2520%2522.build%252Frelease%252F%253CYOUR_EXECUTABLE%253E%2522%2520%252Fusr%252Flocal%252Fbin"
  style="width: 560px; height: 271px; border:0; transform: scale(1); overflow:hidden;"
  sandbox="allow-scripts allow-same-origin">
</iframe>
</center>

For distributing your shiny new executable, you can use tools like `homebrew` or just provide a download link on your website. If you have a single Package in your repo, you use [Mint](https://github.com/yonaskolb/Mint) to distribute your tool right away, without any modifications.

----

SPM now makes it easy to manage and write tests for a command line tool written in Swift. And with tools like [swift-test-codecov](https://github.com/mattpolzin/swift-test-codecov), [upx](https://github.com/upx/upx) and [Mint](https://github.com/yonaskolb/Mint), you can generate coverage reports for your project, optimize the size of your executable, and provide an easy way to distribute it.