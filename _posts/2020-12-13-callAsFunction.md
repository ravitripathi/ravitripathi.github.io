---
title: TILSwift: `callAsFunction`
date: 2020-12-13 20:32:00 +0530
tags: [til-swift]
---

Decided to start a series of (regular?) blog posts on interesting features in the Swift language. First up, `callAsFunction`.

Introduced in [Swift 5.2](https://github.com/apple/swift-evolution/blob/master/proposals/0253-callable.md), `callAsFunction` provides a way to call values as functions. An instance of a `class` or `struct` can be called as a function if it contains a `callAsFunction` method.

Let's dig into this by first starting off with a simple class, `JsonUtility`, which can be initialized with a raw JSON `String`. It contains a `lookup(key:)` function which can recursively check for the value for a given key in the JSON:

```swift
enum JsonUtilityError: Error {
    case initError(String)
}

class JsonUtility {
    private let generatedDict: [String: Any]

    init(withString string: String) throws {
        guard let data = string.data(using: .utf8) else {
            throw JsonUtilityError.initError("Can't extract data from string")
        }
        let obj = try JSONSerialization.jsonObject(with: data, options: [])
        guard let dict = obj as? [String: Any] else {
            throw JsonUtilityError.initError("JSON invalid")
        }
        generatedDict = dict
    }

    func lookup(key: String) -> Any? {
        lookup(key: key, inDict: generatedDict)
    }

    private func lookup(key: String, inDict dictionary: [String: Any]) -> Any? {
        var output: Any?
        for dictKey in dictionary.keys {
            if dictKey == key {
                output = dictionary[dictKey]
            } else if let childDict = dictionary[dictKey] as? [String: Any] {
                output = lookup(key: key, inDict: childDict)
            }
        }
        return output
    }
```

As you can see, when initialized with a string, this class converts the provided string into a `[String: Any]` dictionary and assigns it to `generatedDict`. A `lookup(key:)` function searches the value for this key in this `generatedDict` recursively. This is what it would look like in action:

```swift
var str = """
{
    "id": 234896773,
    "name": "Beaver",
    "owner": {
      "login": "ravitripathi",
      "id": 13906959,
      "url": "https://api.github.com/users/ravitripathi",
      "html_url": "https://github.com/ravitripathi",
      "followers_url": "https://api.github.com/users/ravitripathi/followers",
      "starred_url": "https://api.github.com/users/ravitripathi/starred{/owner}{/repo}",
      "repos_url": "https://api.github.com/users/ravitripathi/repos",
      "events_url": "https://api.github.com/users/ravitripathi/events{/privacy}",
      "received_events_url": "https://api.github.com/users/ravitripathi/received_events",
      "type": "User",
      "site_admin": false
    }
}
"""

if let jsonUtil = try? JsonUtility(withString: str) {
    let htmlUrlString = jsonUtil.lookup(key: "html_url") //  "https://github.com/ravitripathi"
}
```

But what if we could remove the `.lookup` call to obtain the urlString? `callAsFunction` allows you to use the `jsonUtil` instance as a function call, allowing you to make your call syntax more concise.

```swift
class JsonUtility {
    private let generatedDict: [String: Any]
    init(withString string: String) throws { ... }

    private func lookup(key: String, inDict dictionary: [String: Any]) -> Any? { ... }

    func callAsFunction(findValueForKey key: String) -> Any? {
        return lookup(key: key, inDict: generatedDict)
    }
}

//Usage
if let jsonUtil = try? JsonUtility(withString: str) {
    let htmlUrlString = jsonUtil(findValueForKey: "html_url") //  "https://github.com/ravitripathi"
}
```

Just like with normal functions, it is also possible to overload a `callAsFunction` method:

```swift
func callAsFunction(findValueForKey key: String) -> Any? { .. }
func callAsFunction(findKeyForValue value: String) -> String? { .. }

jsonUtil(findValueForKey: "html_url") //  "https://github.com/ravitripathi"
jsonUtil(findKeyForValue: "https://api.github.com/users/ravitripathi/repos") // "repo_url"
```
----

### Where should I use this?

Often, you might have many nominal types, that have a "primary method" that performs their main use. For example, a calculator class which mostly calls a specific function.
```swift
calculator.calculating(query).
```

Since a primary function like this one would be called frequently, `callAsFunction` simplifies the expression used for it.