---
title:  Using Environment Object for Reactive Views
date: 2020-08-02 00:00:00 +0530
tags: [swiftui, combine]
---

Traditionally, writing networking code for iOS involved creating one or more _Networking Classses_. So, after you are done with creating your model objects, you might create a networking class which could look something like this:

```swift
class NetworkManager {
    static let shared = NetworkManager()
    
     func searchRepo(withName name: String, userName: String, completion: @escaping ([Repo]?)->(Void)) {
        ...
        let task = URLSession(configuration: URLSessionConfiguration.default).dataTask(with: requiredURL) { 								(data, response, error) in
            guard let httpResponse = response as? HTTPURLResponse,
                  (200...299).contains(httpResponse.statusCode), let data = data else {
                completion(nil)
                return
            }
            if let decodedResponse = try? JSONDecoder().decode([Repo].self, from: data) {
                completion(decodedResponse)
            }
        }
        task.resume()
    }
}
```

Or you might use a networking library like Alamofire for this purpose. Either way, the structure remains pretty much the same, a closure which provides you with the result of your call. But while working with SwiftUI, you might find this networking closure to be a bit clunky:

```swift
struct YourView: View {
    
    var viewModel: ViewModel
    @State var list: [Repo] = []
    
    var body: some View {
      List(list) { repo in
        //Your row
      }
    }.onAppear {
            viewModel.fetchRepos { (repoList) in
                self.list = repoList
            }
    }
}
```

You rely on settig a `@State` property to refresh your view after the closure returns. Also, if you have multiple views which utilize the same datasource, you might end up either making the same API call mutliple times, or building a complex mesh of initialzer where you pass the required object from one view to another. 

However, SwiftUI has tools which make it easier for you to share data across views, and make reactive bindings around them to ensure your views update when the data changes. Which brings us to one such tool, the `EnvironmentObject`



## EnvironmentObject and NetworkStore

First, lets build a `NetworkStore`



```swift
import Combine

class NetworkStore: ObservableObject {
    @Published private(set) var repos: [Repo] = []
    @Published private(set) var isLoadingRepos: Bool = false

    func fetch() {
        self.isLoadingRepos = true
        NetworkManager.shared.getRepoList { (repos) -> (Void) in
            if let repos = repos {
                DispatchQueue.main.async {
                    self.repos = repos
                    self.isLoadingRepos = false
                }
            }
        }
    }
}
```



Calling `fetch` gets the list of GitHub repos, and assigns it to the `repos` property. A boolean property `isLoadingRepos` keeps track of the status of the network call.



`NetworkStore` is an `ObservableObject` allowing it to be passed along as an `EnvironmentObject`. But what exactly is that? From the docs:

> An environment object invalidates the current view whenever the observable object changes. 



Perfect for our usecase! An environment object can be injected into a view via its parent, or even be made available app-wide.

- Injecting into a view: 

  ```swift
  YourView().environmentObject(NetworkStore())
  ```

- Injecting app-wide:

  ```swift
  //Note: The App Struct is a replacement for the AppDelegate, available from iOS 14+. You can also use the App/Scene Delegate for injecting an environment object.
  
  @main
  struct YourAppName: App {
      var body: some Scene {
          WindowGroup {
              ContentView()
                  .environmentObject(NetworkStore())
          }
      }
  }
  ```



You can then use it any number of your views:



```swift
struct MainView: View {
    @EnvironmentObject var netStore: NetworkStore
    var body: some View {
        ScrollView(.horizontal, showsIndicators: false) {
            HStack{
                if netStore.isLoadingRepos {
                    ProgressView()
                        .progressViewStyle(CircularProgressViewStyle())
                }
                ForEach(netStore.repos) { repo in
                    // add a view
                }
            }
        }.onAppear {
          netStore.fetch()
        }
    }
}
```



This frees you from making networking call in a specific view. You can call `netStore.fetch()` from any view, and every view which requires the repository list automatically gets updated, as long as it uses the NetworkStore environment object.

----

SwiftUI and Combine complement each other in many ways, and can be mixed and matched to build complex, reactive views with  reduced efforts. You can check out my [GitBrowser](https://github.com/ravitripathi/GitBrowser) app, a simple Github client which displays your followers and public repositories, built on pure SwiftUI (Requires Xcode 12 beta to run).