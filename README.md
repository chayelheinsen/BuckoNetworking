 [![Carthage compatible](https://img.shields.io/badge/Carthage-compatible-4BC51D.svg?style=flat)](https://github.com/Carthage/Carthage)

# iOS Protocol-Oriented Networking in Swift

After developing a number of applications, we noticed that everyone's networking code was different. Every time maintenance developers had to take over a project, they had to "learn" the individual nuances of each network layer. This lead to a lot of wasted time. To solve this, our iOS Engineers decided to use the same networking setup. Thus we looked into Protocol-Oriented Networking.


### Dependencies
------

* [Alamofire](https://github.com/Alamofire/Alamofire) - We ended up going with Alamofire instead of `URLSession` for a few reasons. Alamofire is asynchronous by nature, has session management, reduces boilerplate code, and is very easy to use.

* [PromiseKit](https://github.com/mxcl/PromiseKit) - We use Promises because they simplify asynchronous programming and separate successful and failed responses, allowing you to focus on each part in their own individual closures.

### Installation
------

- NOTE: If you are looking for the Swift 3 version, use BuckoNetworking version `1.1.3`.

#### Carthage

[Carthage](https://github.com/Carthage/Carthage) is a decentralized dependency manager that builds your dependencies and provides you with binary frameworks.

You can install Carthage with [Homebrew](http://brew.sh/) using the following command:

```bash
$ brew update
$ brew install carthage
```

To integrate BuckoNetworking into your Xcode project using Carthage, specify it in your `Cartfile`:

```ogdl
github "teepsllc/BuckoNetworking" ~> 2.1.0
```

1. Run `carthage update --platform iOS --no-use-binaries` to build the framework.
2. On your application targets’ “General” settings tab, in the “Linked Frameworks and Libraries” section, drag and drop `BuckoNetworking.framework` from the [Carthage/Build]() folder on disk. You will also need to drag `Alamofire.framework` and `PromiseKit.framework` into your project.
3. On your application targets’ “Build Phases” settings tab, click the “+” icon and choose “New Run Script Phase”. Create a Run Script in which you specify your shell (ex: `/bin/sh`), add the following contents to the script area below the shell:

   ```sh
   /usr/local/bin/carthage copy-frameworks
   ```

4. Add the paths to the frameworks you want to use under “Input Files”, e.g.:

   ```
   $(SRCROOT)/Carthage/Build/iOS/BuckoNetworking.framework
   $(SRCROOT)/Carthage/Build/iOS/Alamofire.framework
   $(SRCROOT)/Carthage/Build/iOS/PromiseKit.framework
   ```
   
   This script works around an [App Store submission bug](http://www.openradar.me/radar?id=6409498411401216) triggered by universal binaries and ensures that necessary bitcode-related files and dSYMs are copied when archiving.

5. Add the paths to the copied frameworks to the “Output Files”, e.g.:
  
   ```
   $(BUILT_PRODUCTS_DIR)/$(FRAMEWORKS_FOLDER_PATH)/BuckoNetworking.framework
   $(BUILT_PRODUCTS_DIR)/$(FRAMEWORKS_FOLDER_PATH)Alamofire.framework
   $(BUILT_PRODUCTS_DIR)/$(FRAMEWORKS_FOLDER_PATH)/PromiseKit.framework
   ```
  
    With output files specified alongside the input files, Xcode only needs to run the script when the input files have changed or the output files are missing. This means dirty builds will be faster when you haven't rebuilt frameworks with Carthage.


### Usage
------

To use BuckoNetworking, just import the module.

```swift
import BuckoNetworking
```

`BuckoNetworking` revolves around `Endpoint`s. There are a few ways you can use it. We use `services` to make all of our endpoints.

#### DecodableEndpoint

Swift 4 introduced the Codable protocol and the `DecodableEndpoint` in BuckoNetworking uses this to the max!

```swift
struct User: Decodable {
  var name: String
  var phoneNumber: String

  enum CodingKeys: String, CodingKey {
    case name
    case phoneNumber = "phone_number"
  }
}

struct UserService: DecodableEndpoint {
  typealias ResponseType = User
  var baseURL: String { return "https://example.com" }
  var path: String { return "/users" }
  var method: HTTPMethod { return .get }
  var body: Parameters { return Parameters() }
  var headers: HTTPHeaders { return HTTPHeaders() }
}

UserService().request().then { users in
  // Do something with users
  users.count
}.catch { error in

  if let json = error.json {
    // Use json
  } else {
    // Some other error occurred that doesn't include json
  }
}
```

If you don't want to use Promises, BuckoNetworking also provides normal closures:

```swift
UserService().request { (user, error) in
  guard let user = user else {
    // Do Error
    return
  }

  // Do something with user
}

```

Using an enum and `DecodableEndpoint` is possible, however, `DecodableEndpoint` will require that each case return the same type.
If you want each case to respond with a separate `Codable` type, you can use `Endpoint` and its `request(responseType:, completion:)` method.

```swift
enum UserService: Endpoint {
  case index

  var baseURL: String { return "https://example.com" }
  var path: String { return "/users" }
  var method: HTTPMethod { return .get }
  var body: Parameters { return Parameters() }
  var headers: HTTPHeaders { return HTTPHeaders() }
}

// Use your Endpoint
UserService.index.request(responseType: [User].self).then { users in
  // Do something with users
  users.count
}.catch { error in

  if let json = error.json {
    // Use json
  } else {
    // Some other error occurred that doesn't include json
  }
}

// Or without Promises
UserService.index.request(responseType: [User].self) { (users, error) in
  guard let users = users else {
    // Do Error
    return
  }

  // Do something with users
}

```

If you don't want to use `Codable`, you can instead use the `Endpoint` protocol, and provide your own object mapping.

#### Endpoint

#### class/struct

```swift
import BuckoNetworking

// Create an Endpoint
struct UserCreateService: Endpoint {
    var baseURL: String = "https://example.com/"
    var path: String = "users/"
    var method: HTTPMethod = .post
    var parameters: Parameters {
        var parameters = Parameters()
        parameters["first_name"] = "Bucko"
        return parameters
    }
    var headers: HttpHeaders = ["Authorization" : "Bearer SOME_TOKEN"]
}

// Use your Endpoint
Bucko.shared.request(UserCreateService()).then { response in
  // Response successful!
}.catch { error in
  //  Failure
}

// Or without Promises
Bucko.shared.request(UserCreateService()) { response in
  if response.result.isSuccess {
    // Response successful!
    // Convert `response.result.value!` to JSON
  } else {
    // Failure
  }
}

```

#### enum

```swift
import BuckoNetworking

// Create an Endpoint
enum UserService {
    case getUsers
    case getUser(id: Int)
    case createUser(firstName: String, lastName: String)
}

extension UserService: Endpoint {
    var baseURL: String { return "https://example.com/" }

    // Set up the paths
    var path: String {
        switch self {
        case .getUsers: return "users/"
        case .getUser(let id): return "users/\(id)/"
        case .createUser: return "users/"
        }
    }

    // Set up the methods
    var method: HTTPMethod {
        switch self {
        case .getUsers: return .get
        case .getUser: return .get
        case .createUser: return .post
        }
    }

    // Set up any headers you may have. You can also create an extension on `Endpoint` to set these globally.
    var headers: HTTPHeaders {
        return ["Authorization" : "Bearer SOME_TOKEN"]
    }

    // Lastly, we set the body. Here, the only route that requires parameters is create.
    var parameters: Parameters {
        var parameters: Parameters = Parameters()

        switch self {
        case .createUser(let firstName, let lastName):
            parameters["first_name"] = firstName
            parameters["last_name"] = lastName
        default:
            break
        }

        return parameters
    }
}

// Use your Endpoint
Bucko.shared.request(UserService.getUser(id: 1)).then { response in
  // Response successful!
}.catch { error in
  //  Failure
}

// Or without Promises
Bucko.shared.request(UserService.getUser(id: 1)) { response in
  if response.result.isSuccess {
    // Response successful!
    // Convert `response.result.value!` to JSON
  } else {
    // Failure
  }
}

```

### Blog
------

You can go to our [Blog](https://teeps.org/blog/2017/02/27/26-protocol-oriented-networking-in-swift) to read more.
