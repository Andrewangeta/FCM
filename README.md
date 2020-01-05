[![Mihael Isaev](https://user-images.githubusercontent.com/1272610/42512735-738605f4-8466-11e8-80ef-86394e852875.png)](http://mihaelisaev.com)

<p align="center">
    <a href="LICENSE">
        <img src="https://img.shields.io/badge/license-MIT-brightgreen.svg" alt="MIT License">
    </a>
    <a href="https://swift.org">
        <img src="https://img.shields.io/badge/swift-4.1-brightgreen.svg" alt="Swift 4.1">
    </a>
    <a href="https://twitter.com/VaporRussia">
        <img src="https://img.shields.io/badge/twitter-VaporRussia-5AA9E7.svg" alt="Twitter">
    </a>
</p>

<br>


# Intro 👏

It's a swift lib that gives ability to send push notifications through Firebase Cloud Messaging.

Built for Vapor3 and depends on `JWT` Vapor lib.

Note: the project is in active development state and it may cause huge syntax changes before v1.0.0

If you have great ideas of how to improve this package write me (@iMike) in [Vapor's discord chat](http://vapor.team) or just send pull request.

Hope it'll be useful for someone :)

### Install through Swift Package Manager ❤️

Edit your `Package.swift`

```swift
//add this repo to dependencies
.package(url: "https://github.com/MihaelIsaev/FCM.git", from: "1.1.0")
//and don't forget about targets
//"FCM"
```

### How it works ?

First of all you should configure FCM in `configure.swift`

```swift
import FCM

/// Called before your application initializes.
public func configure(_ config: inout Config, _ env: inout Environment, _ services: inout Services) throws {
//here you should initialize FCM
}
```

#### There are two ways

##### 1. Using environment variables 👍
```swift
let fcm = FCM()
services.register(fcm, as: FCM.self)
```
and don't forget to pass the following environment variables
```swift
fcmServiceAccountKeyPath // /tmp/serviceAccountKey.json
```
OR
```swift
fcmEmail // firebase-adminsdk-0w4ba@example-3ab5c.iam.gserviceaccount.com
fcmKeyPath // /tmp/fcm.pem
fcmProjectId // example-3ab5c
```

##### 2. Manually 🤖
```swift
let fcm = FCM(pathToServiceAccountKey: "/tmp/serviceAccountKey.json")
services.register(fcm, as: FCM.self)
```
OR
```swift
let fcm = FCM(email: "firebase-adminsdk-0w4ba@example-3ab5c.iam.gserviceaccount.com",
              projectId: "example-3ab5c",
              pathToKey: "/tmp/fcm.pem")
services.register(fcm, as: FCM.self)
```
OR
```swift
let fcm = FCM(email: "firebase-adminsdk-0w4ba@example-3ab5c.iam.gserviceaccount.com",
              projectId: "example-3ab5c",
              key: "<YOUR PRIVATE KEY>")
services.register(fcm, as: FCM.self)
```

> ⚠️ **TIP:** `serviceAccountKey.json` you could get from [Firebase Console](https://console.firebase.google.com)
>
> 🔑 Just go to Settings -> Service Accounts tab and press **Create Private Key** button in e.g. NodeJS tab

#### OPTIONAL: Set default configurations, e.g. to enable notification sound
Add the following code to your `configure.swift`
```swift
fcm.apnsDefaultConfig = FCMApnsConfig(headers: [:],
                                      aps: FCMApnsApsObject(sound: "default"))
fcm.androidDefaultConfig = FCMAndroidConfig(ttl: "86400s",
                                            restricted_package_name: "com.example.myapp",
                                            notification: FCMAndroidNotification(sound: "default"))
fcm.webpushDefaultConfig = FCMWebpushConfig(headers: [:],
                                            data: [:],
                                            notification: [:])
```
#### Let's send first push notification! 🚀

Then you could send push notifications using token, topic or condition.

Here's an example route handler with push notification sending using token

```swift
router.get("testfcm") { req -> Future<String> in
  let fcm = try req.make(FCM.self)
  let token = "<YOUR FIREBASE DEVICE TOKEN>"
  let notification = FCMNotification(title: "Vapor is awesome!", body: "Swift one love! ❤️")
  let message = FCMMessage(token: token, notification: notification)
  return try fcm.sendMessage(req.client(), message: message)
}
```

`fcm.sendMessage` returns message name like `projects/example-3ab5c/messages/1531222329648135`

`FCMMessage` struct is absolutely the same as `Message` struct in Firebase docs https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages
So you could take a look on its source code to build proper message.

## Bonus 🍾

In my Vapor projects I'm using this extension for sending push notifications

```swift
import Vapor
import Fluent
import FCM

protocol Firebaseable: Model {
    var firebaseToken: String? { get set }
}

extension Firebaseable {
    func sendPush(title: String, message: String, on req: Container) throws -> Future<Void> {
        guard let token = firebaseToken else {
            return req.eventLoop.newSucceededFuture(result: ())
        }
        return try Self.sendPush(title: title, message: message, token: token, on: req)
    }

    static func sendPush(title: String, message: String, token: String, on container: Container) throws -> Future<Void> {
        let fcm = try container.make(FCM.self)
        let message = FCMMessage(token: token, notification: FCMNotification(title: title, body: message))
        return try fcm.sendMessage(container.make(Client.self), message: message).transform(to: ())
    }
}

extension Array where Element: Firebaseable {
    func sendPush(title: String, message: String, on container: Container) throws -> Future<Void> {
        return try map { try $0.sendPush(title: title, message: message, on: container) }.flatten(on: container)
    }
}
```
Optionally you can handle `sendMessage` error through defining `catchFlatMap` after it, e.g. for removing broken tokens or anything else
```swift
return try fcm.sendMessage(container.make(Client.self), message: message).transform(to: ()).catchFlatMap { error in
    guard let googleError = error as? GoogleError, let fcmError = googleError.fcmError else {
        return container.eventLoop.newSucceededFuture(result: ())
    }
    switch fcmError.errorCode {
        case .unregistered: // drop token only if unregistered
            return container.requestPooledConnection(to: .psql).flatMap { conn in
                return Self.query(on: conn).filter(\.firebaseToken == token).first().flatMap { model in
                    defer { try? container.releasePooledConnection(conn, to: .psql) }
                    guard var model = model else { return container.eventLoop.newSucceededFuture(result: ()) }
                    model.firebaseToken = nil
                    return model.save(on: conn).transform(to: ())
                }.always {
                    try? container.releasePooledConnection(conn, to: .psql)
                }
            }
        default:
            return container.eventLoop.newSucceededFuture(result: ())
    }
}
```

> Special thanks to @grahamburgsma for `GoogleError` and `FCMError` #10

Then e.g. I'm conforming my `Token` model to `Firebaseable`

```swift
final class Token: Content {
    var id: UUID?
    var token: String
    var userId: User.ID
    var firebaseToken: String?
}
extension Token: Firebaseable {}
```

So then you'll be able to send pushes by querying tokens like this
```swift
Token.query(on: req)
    .join(\User.id, to: \Token.userId)
    .filter(\User.email == "benny@gmail.com")
    .all().map { tokens in
    try tokens.sendPush(title: "Test push", message: "Hello world!", on: req)
}
```
