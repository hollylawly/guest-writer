---
layout: post
title: "Cross-platform in SwiftUI App with Sign in with Apple"
description: ""
date: "2020-07-16 08:30"
author:
  name: "Lila Pustovoyt"
  url: ""
  mail: ""
  avatar: "https://twitter.com//profile_image?size=original"
related:
- 
---

[Apple](https://www.apple.com/) released SwiftUI last year at WWDC 19 as the new way to build native apps for iOS, macOS, tvOS and watchOS. This release was targeted towards building cross platform user-interfaces.

This year at WWDC 20, Apple announced SwiftUI 2.0. SwiftUI now plays a bigger role in native app development for Apple platforms. Unlike the previous version, it can be used on its own to write fully fledged apps, and it also powers new experiences like App Clips and Widgets.

In this blog we’ll build an app and look at how we can use SwiftUI to build great native applications that use Sign in with Apple to provide authentication.

## What is SwiftUI?

Apple ships a variety of devices from smart-watches all the way to TV. Every device has its unique challenges in terms of input capabilities, form factors and use cases. While Apple offers great ways to build applications and abstract these differences, developing an application for all of Apple’s platforms requires developing separate apps with completely different frameworks.

This fragmentation presented a massive barrier to entry when writing even simple native applications, so for many use cases, Apple’s native development frameworks fell behind alternatives like React Native. SwiftUI is Apple’s solution to this problem: as a brand new framework from Apple, it offers a single paradigm to write apps for all of Apple’s platforms.

SwiftUI interfaces can be built in a declarative format similar to React or Angular. It also shuns the MVC pattern as the de-facto design pattern for apps written for all the Apple platforms in favor of modern patterns like MVVM. And with this year’s release of Swift UI 2.0, it offers a cross-platform template that runs out of the box on all platforms.

SwiftUI is very similar to other modern like Angular, Vue and React, if you are familiar with any of them you might find SwiftUI fairly quick to start with.

## What We’ll be building

We’ll be building a To-do list app that runs on macOS, iOS and tvOS. The app will allow users to create and update to-do items in a list, and all changes will be stored on the server side. Our app is inspired by [https://todomvc.com](https://todomvc.com). 

IMAGE HERE

Our application follows the architecture presented by [Alexey’s blog on clean architecture with SwiftUI](https://nalexn.github.io/clean-architecture-swiftui) and uses Sign in with Apple for authenticating users. The full source code for the app is available at [https://github.com/oboecat/todo-swiftui](https://github.com/oboecat/todo-swiftui).

## Setup: A Hello World in SwiftUI

To get started with SwiftUI, we’ll need the following tools pre-installed:

- XCode 12.0 Beta+
- Cocoapods (for Auth0 and SDWebImageSwiftUI)
- Latest versions of macOS, iOS and tvOS to run our project on real hardware

Once XCode beta is installed, creating a new app using SwiftUI project is fairly straightforward. We can do this and test that everything is set up correctly by building a simple Hello World app.

The default template works out of the box on iOS and macOS, to add tvOS we have to add another target in XCode to the project. These are the steps to set up our cross-platform project:

- Click on File > New Project
- Click on multi-platform 
- Select Blank app
- Ensure “View” and “App” are set to SwiftUI
- XCode has created macOS and iOS targets for us, let’s add a new target for tvOS
  - Remove the tvOSApp and ContentView files inside the target’s directory, we won’t need those
  - Make sure all the files in the Shared directory are associated with the tvOS target

The same procedure can be followed to add other targets like App Clips and watchOS. When we build and run the app for each platform, we should see a “Hello, world!” message:

IMAGE HERE

How does this work? Each platform’s directory only contains a few configuration files. The actual code for our app is in the Shared directory which all our targets have in common. At the moment, this consists of `TodoSampleApp.swift` and `ContentView.swift`, but even as we add more views, most of our code will remain shared between the platforms. This is one of the great features of SwiftUI that makes cross-platform development much more straightforward.

### Todo Server

Our application will talk to an API in order to add, update and save the to-do list. We wrote a server in Node.js that handles all of this logic and maintains our to-do list in memory. The full source code is available at https://github.com/oboecat/todo-nodejs-server.

### Anatomy of a SwiftUI App

The starting point of our app is `TodoSampleApp.swift`. It contains a struct implementing the App protocol:  

```swift
@main
struct MyApp: App {
   var body: some Scene {
       WindowGroup {
           ContentView()
       }
   }
}
```

The `@main` annotation marks a function or struct as the entry-point for a Swift application. You can read more about this in [Apple’s documentation](https://docs.swift.org/swift-book/ReferenceManual/Attributes.html#ID626).

The App protocol in SwiftUI includes a default implementation for the main function and handles all the work required to make the app work on all the platforms. This is the root of our application and as such, all app-level environment settings and variables should be passed here.

## Structure of our Todo App

Our app follows the “Model, Views and Interactors” pattern described by Alexey Naumov in his Clean Architecture for SwiftUI article [https://nalexn.github.io/clean-architecture-swiftui](https://nalexn.github.io/clean-architecture-swiftui). Data is stored in Models, which are rendered by the Views, which in turn mutate the data using Interactors. This architectural pattern is popular in React-like frameworks.

IMAGE HERE

### Models

We will store our app’s state in an application-wide store which will function as the single source of truth for the app. For now, the state simply consists of a list to-dos, which will be fetched from the server, maintained locally by our business logic and rendered by our presentation layer.

```swift
import Foundation
class AppStore: ObservableObject{
	@Published var todos: [Todo]
}
```

Our `AppStore` class implements `ObservableObject`, which can automatically update Views when any data is changed. This makes it really easy to have your Views simply take the Model as the single source of truth and render it.

Our `AppStore` model is passed as an [`EnvironmentObject`](https://developer.apple.com/documentation/swiftui/environmentobject) to the views. This allows the views to subscribe to changes and render appropriately. This is entirely managed by SwiftUI.

### Interactors

Interactors are the core business logic of our app, they facilitate user interactions and update the state to reflect the outcome of these interactions. We use protocols to describe the interactions our user can perform. For example, TodoInteractions defines all the interactions associated with managing Todo items:

```swift
protocol TodoInteractions {
    func getTodos() {}
    func updateTodo(_ todo: Todo)
    func addNewTodo(_ todo: Todo)
    func deleteTodo(_ todoId: String)
}
```

Our `WebTodoInteractor` implements this protocol by communicating with the server using `APIClient`, a helper that abstracts the networking code:

```swift
class WebTodoInteractor: TodoInteractions {
    private var api: APIClient
    private unowned var store: AppStore
    private var cancelBag = Set<AnyCancellable>()
    
    init(store: AppStore, apiClient: APIClient) {
        self.store = store
        self.api = apiClient
    }

    func getTodos() {
        api.getTodos()
            .receive(on: RunLoop.main)
            .assign(to: \.todos, on: store)
            .store(in: &cancelBag)
    }
    
    func updateTodo(_ todo: Todo) {
        api.updateTodo(todo)
            .receive(on: RunLoop.main)
            .sink { todo in
                if let index = self.store.todos.firstIndex(where: { 
$0.id == todo.id 
   }) {
                    self.store.todos[index] = todo
                }
            }
            .store(in: &cancelBag)
    }
    
    func addNewTodo(_ todo: Todo) {
        api.addNewTodo(todo)
            .receive(on: RunLoop.main)
            .sink { todo in
                self.store.todos.append(todo)
            }
            .store(in: &cancelBag)
    }    
}
```

For previewing and testing, we can create a mock implementation of the protocol to use instead of `WebTodoInteractor`.

A helper class called `AppEnvironment` handles the creation of the AppStore and the Interactors, which are then injected into the environment from `TodoSampleApp`:

```swift
@main
struct NewTodoSampleApp: App {
    @StateObject var appEnvironment = AppEnvironment()
    
    var body: some Scene {
        WindowGroup {
            TodoAppView()
                .environmentObject(appEnvironment.store)
                .environment(\.todosInteractor, appEnvironment.todoInteractor)
                .environment(\.userInteractor, appEnvironment.userInteractor)
        }
    }
}
```

### Views 

Views are the basic building blocks of the user interface. SwiftUI comes with many built-in views fulfilling a variety of functions: from views that display text and images to presentational components like stacks and grids, and even more complicated views with associated behaviors, such as `TextField` and `SignInWithAppleButton`. These can be composed to create custom views that make up a full user interface. The community has created a great [cheat sheet](https://github.com/SimpleBoilerplates/SwiftUI-Cheat-Sheet) for commonly used views.

To see how we can define our own custom views, let’s take a look at the TodoListTitle view, which displays the title of our app in a large font:

IMAGE HERE

```swift
struct TodoListTitle: View {
    var body: some View {
        Text("Todo Sample")
            .font(.system(.largeTitle, design: .rounded))
            .foregroundColor(.accentColor)
    }
}
```

Here, we added modifiers to a Text view to make it large, give it a rounded design and set its color to the accent color of our app, which is blue.

Now wherever we need to display the title in our app, we can refer to our TodoListTitle view.

SwiftUI offers [several methods](https://www.vadimbulavin.com/passing-data-between-swiftui-views/) for passing data to a view. Our TodoList subscribes to the AppStore and observes the todos, it also accepts a property that allows us to filter only pending todos. As mentioned earlier, the `@EnvironmentObject` property wrapper allows the view to respond to changes to the `AppStore`.

IMAGE HERE

```swift
struct TodoList: View {
    @EnvironmentObject var store: AppStore
    var isShowingCompleted: Bool
    
    var body: some View {
        List {
            ForEach(store.todos.filter { self.isShowingCompleted || !$0.completed }) { todo in
                TodoItem(todo: todo)
            }
            NewTodoItem()
        }
        .animation(.default)
    }
}
```

Other views in our app have more responsibilities than just displaying content, such as handling user interaction. This often requires managing a local view state. The TodoItem view inside the TodoList is a good example of this:

IMAGE HERE

```swift
struct TodoItem: View {
    @Environment(\.todosInteractor) var interactor: TodoInteractions
    @State var bodyDraft: String = ""
    @State var isMarkedCompleted = false
    var todo: Todo
    
    var body: some View {
        #if os(tvOS)
        return todoItem.onPlayPauseCommand(perform: self.toggleCompleted)
        #else
        return todoItem
        #endif
    }
    
    var todoItem: some View {
        HStack {
            #if os(tvOS)
            TodoStatus(isCompleted: isMarkedCompleted)
            #else
            Button(action: self.toggleCompleted) {
                TodoStatus(isCompleted: self.isMarkedCompleted)
            }
            #endif

            TextField("To-do", text: $bodyDraft, onCommit: {
                var updatedTodo = self.todo
                updatedTodo.body = self.bodyDraft
                self.interactor.updateTodo(updatedTodo)
            })
            .textFieldStyle(PlainTextFieldStyle())

            Spacer()
        }
        .onAppear {
	     // Initialize the local state
            self.isMarkedCompleted = self.todo.completed
            self.bodyDraft = self.todo.body
        }
    }
    
    private func toggleCompleted() {
        var updatedTodo = self.todo
        updatedTodo.completed.toggle()
        self.isMarkedCompleted = updatedTodo.completed
        self.interactor.updateTodo(updatedTodo)
    }
}
```

TodoItem displays a single Todo task. The user can either modify the text of the Todo or toggle the completion status. However, since the server is the source of truth for the AppStore, we don’t want TodoItem to modify it directly. But we also want our UI to update as soon as the user makes a change, rather than waiting for the server to respond.

To achieve this, we define two properties, `bodyDraft` and `isMarkedCompleted`, that are copies of the `body` and `completed` properties of the Todo sent by the server, but that the user can modify directly. @State marks these properties as state variables owned by our view. Whenever the user makes a change, the view uses the TodoInteractor to send the updated value to the server.

This view also shows how you can make adjustments to accommodate for different user experiences on different platforms. For example, on tvOS the play/pause button on the remote toggles the completion status. SwiftUI encourages this [adaptive interface building](https://developer.apple.com/videos/play/wwdc2019/240).

Our application uses `WebTodoInteractor` that works with our server to manage todos, and stores them in AppStore. Our TodoScreen and its descendants render this information and use the interactor to submit modifications.

IMAGE HERE

## Adding Authentication with Sign in With Apple

Now that we can create and update todos, let’s add authentication using Sign in with Apple.

### Sign in with Apple ID in SwiftUI

In WWDC 20, Apple released a `SignInWithAppleButton` for SwiftUI.

```swift
 SignInWithAppleButton(
        .signIn,
        onRequest: { request in
            // Modify requested permissions
            request.requestedScopes = [.fullName, .email]
        },
        onCompletion: { result in
            switch result {
               case .success(let authResults):
                    // Handle success
               case .failure(let error):
                    // Handle failure
            }
        }
 )

```
This button makes it easy to add Sign in with Apple to a SwiftUI app, since the entire authentication flow is managed by Apple and the only thing we need to do is handle the result.

When the user successfully authenticates, the completion closure is called with a set of credentials from Apple, which include information about the user and an authorization code. 

However, Apple does not provide us with a token to call our API, so we can’t use these credentials as-is in our service. Instead, we should exchange the Apple-provided authorization code for an access token which can be used to call our API. Thankfully, Auth0 makes this really easy.

### Auth0 Setup

The Auth0 setup is fairly straightforward, since most of the tasks we need to perform are supported out of the box. To set up our Auth0 tenant, we need to complete these steps in the Auth0 dashboard:

- Configure the Sign in with Apple social connection
- Create a Todo API (https://todo.example.org)
- Create a new application for each of the platforms on which our app runs
- Enable native Sign in with Apple for all applications

IMAGE HERE

### Adjustments to the Todo Server 
There are minimal changes to our API Server when adding Auth0. Since our server is using Express and Node.js, we simply followed the steps outlined in [quickstart](https://auth0.com/docs/quickstart/webapp/nodejs/01-login). This ensures that all requests must have an access token issued by Auth0 to be fulfilled. We use the `sub` claim in the access token that represents the `user_id` to associate a list of to-dos with its owner.

### Changes to the Todo Client

Now that the user can sign in, we can add two new properties to the AppStore: the user information and an `authStatus` property to keep track of the authentication status.

```swift
class AppStore: ObservableObject {
    @Published var authStatus: AuthStatus = .loading
    @Published var user: UserInfo?
    @Published var todos: [Todo]
    
    init(user: UserInfo?, todos: [Todo]) {
        self.authStatus = .loading
        self.user = user
        self.todos = todos
    }
    
    convenience init() {
        self.init(user: nil, todos: [])
    }
    
    enum AuthStatus {
        case loading
        case notSignedIn(error: Error?)
        case signedIn
    }
}
```

Our app requires users to sign in before using the app. This policy is enforced by a new `RoutingView`, which renders the appropriate screen depending on the authentication status:

```swift
struct Router: View {
    @EnvironmentObject var store: AppStore
    
    var body: some View {
        Group {
            switch store.authStatus {
            case .loading:
                LoadingScreen()
            case .notSignedIn:
                LoginScreen()
            case .signedIn:
                TodoListScreen()
            }
        }
    }
}

```

[Video of the sign in process](https://youtu.be/7sL24uVC3Oc)

In the business layer, the `UserInteractions` protocol defines the actions associated with the user, such as login and logout. The login method accepts the user’s authorization credentials provided by Sign in with Apple.

```swift
protocol UserInteractions {
    func login(_ appleIdCredential: ASAuthorizationAppleIDCredential)
    func logout()
    func checkSession()
}
```

`Auth0UserInteractor` implements `UserInteractions` by wrapping the `Auth0.swift` library based on [Auth0’s Swift - Sign in with Apple Quickstart](https://auth0.com/docs/quickstart/native/ios-swift-siwa). We also expanded our `APIClient` utility to use `CredentialsManager` from `Auth0.swift`. `CredentialsManager` is used by our `APIClient` to add the latest auth0 access token to the HTTP request.

Now we can use the `SignInWithAppleButton` to call the login method in our interactor:

```swift
struct LoginButton: View {
    @Environment(\.userInteractor) var userInteractor: UserInteractions
    
    var body: some View {
        SignInWithAppleButton(
            .signIn,
            onRequest: { request in
                request.requestedScopes = [.fullName, .email]
            },
            onCompletion: { result in
                switch result {
                case .success(let authResults):
                    let appleIDCredential = authResults.credential as! ASAuthorizationAppleIDCredential
                    self.userInteractor.loginApple(appleIDCredential)
                case .failure(let error):
                    print("Authorization failed: " + error.localizedDescription)
                }
            }
        ).frame(width: 200, height: 44, alignment: .center)
    }
}
```

### Sign in with Apple on Simulators

At the time of writing, Sign in with Apple does not work on any of the Simulators. If you to Sign in with Apple, the system will keep waiting for Apple to respond. This becomes an obstacle in running and testing the application on simulators.

As a workaround, we created a test user with a username and password on Auth0. This can be done easily by clicking the “Create User” button in Dashboard > Users & Roles > Users.

We then added username and password login via Resource Owner Password Grant, since we want to allow this only for our simulator we use `#if targetEnvironment(simulator)` to abstract the login mechanism being used to authenticate the user.

```swift
#if targetEnvironment(simulator)
struct LoginButton: View {
    @Environment(\.userInteractor) var userInteractor: UserInteractions
    
    var body: some View {
        Button(action: {
            self.userInteractor.testLogin()
        }) {
            Text("Sign In")
        }
    }
}
#endif
```

## Conclusion

Our app can now create and manage todos for multiple users, to users that login using Sign in with Apple via Auth0. The Interactor pattern allows us to keep the business logic completely independent. 

IMAGE HERE

SwiftUI is a great new way to start writing applications on iOS, it offers a declarative UI, modern architectures and offers exciting experiences like App-Clips and Widgets on iOS 14+. As we move closer to iOS 14 we will probably see more changes in SwiftUI and some of its APIs.
