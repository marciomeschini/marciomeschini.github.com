---
title: "Hackathon"
date: 2021-10-31T23:36:10Z
draft: true
---

# Intro

A while ago me and my team built a small `Status Bar` app to intercept deeplinks and to redirect them to both iOS Simulator and Android Emulator. Our mobile app already provides in-depth deeplink support, so the idea came natural: why use deeplink only for marketing?
We can adopt deeplinks also to simplify several internal processes such as:
- backend environment selection
- enter account credentials

We were lucky, we won. 🎉

&nbsp;

# Target platform

User can choose which platform to open the url with: either both platforms or one between iOS and Android.

We can use the following enum to model that and pass the values through an array:

{{< highlight swift >}}
enum Platform {
  case android
  case iOS
}
{{< /highlight >}}

However we soon discover something:

{{< highlight swift >}}
var platforms: [Platform]
platforms = [.android] 
platforms = [.iOS]

enabledPlatforms = [.android, .iOS, .iOS, .android] // 1. 🤔
enabledPlatforms = [] // 2. 😱
{{< /highlight >}}

To fix (1) we can use a `Set` in place of an array however in order to address empty set (2) we need something different. We leave this topic for next time as now we want to focus on one specific problem: how to open the link and how to provide a good feedback to the call-site.

Let's quickly address the **two impossible states** and introduce a third case to our enum: `both`. We closed the gate and client can not send bad requests anymore.

{{< highlight swift >}}
enum Platform {
  case android
  case both
  case iOS
}
{{< /highlight >}}

&nbsp;


# Platform

A platform represents an abstraction for both the iOS Simulator and the Android Emulator.

Behind the scenes we adopt two different tools: [simctl][simctl] and [adb][adb] respectively to interact with the two platforms but - for now - let's not worry about these details. Let's try instead to reduce the number of variables.

1. We know that a user may or may not have a given platform installed, if so we want to return an error.
2. Once we have a platform available we want to invoke `openURL` on that and eventually we want a result back. So also in that case we may want to return an error to the user.

Swift provides a standard type for this kind of requirements: the `Result` type.
We can add the following:

{{< highlight swift >}}
typealias DeviceOpen = (URL) -> Result<Void, Error> // 2 
let handler: () -> Result<DeviceOpen, Error> // 1
{{< /highlight >}}

&nbsp;

## The Controller

The `OpenURLController` object hides the connection to the two platforms and exposes a single API.

- the `URL` to open
- the `Platform` configuration, ie which devices to target

and returns back a `Result` type.

{{< highlight swift >}}
func open(_ url: URL, platform: Platform) -> Result<Void, Error>
{{< /highlight >}}

The `Success` value for now can be `Void` as we don't need to carry extra information back during this post.

Every time `open` function is called we want to get an handler for the required platform:
* something may happen between different executions
* avoid storing states especially if it involves a third party command line tool

We can break down the amount of possible combinations with some nice ascii art:



```
(URL, Platform) -> Result
1     3             2
2³ =  8 different paths


+----------+------+------+------+------+
| Platform |      |      |      |      |
+==========+======+======+======+======+
| Android  |   1  |   0  |      |      |
+----------+------+------+------+------+
| iOS      |   1  |   0  |      |      |
+----------+------+------+------+------+
| Both     |  11  |  00  |  10  |  01  |
+----------+------+------+------+------+
```


8 different paths sound like a hell of a job but `Result` type comes to the rescue. 
We are able to chain the 2 operations: get handler + open(url) with a `flatMap`. The rest is done through a couple of `switch`.

We don't have any special requirement in terms of error handling, we just want to pass them up to the caller.

The full code is below, I added some comments to track the 8 different paths we do support. 

{{< highlight swift >}}
struct GroupError: Error {
  let errors: [Error]
}

struct OpenURLController {
  typealias DeviceOpen = (URL) -> Result<Void, Error>
  let android: () -> Result<DeviceOpen, Error>
  let iOS: () -> Result<DeviceOpen, Error>

  func open(_ url: URL, platform: Platform) -> Result<Void, Error> {
    switch platform {
    case .android:
      return android().flatMap { $0(url) } // 1, 2
    case .both:
      switch (android().flatMap { $0(url) }, iOS().flatMap { $0(url) }) {
      case (.success, .success):
        return .success(()) // 3
      case let (.failure(iOS), .failure(android)):
        return .failure(GroupError(errors: [iOS, android])) // 4
      case let (.failure(error), .success), let (.success, .failure(error)):
        return .failure(error) // 5, 6
      }
    case .iOS:
      return iOS().flatMap { $0(url) } // 7, 8
    }
  }
}
{{< /highlight >}}

We can then take the code for a ride with some old school print oriented debugging.

{{< highlight swift >}}

OpenURLController( // 🌧
  android: { .failure(TestError.device) },                      // ❌
  iOS: { .success { _ in .failure(TestError.open) } }           // ✅ ❌
).open(url, platform: .both)

OpenURLController( // 🌧
 android: { .success { _ in .failure(TestError.open) } },      // ✅ ❌
 iOS: { .success { _ in .success(()) } }                       // ✅ ✅
).open(url, platform: .both)

OpenURLController( // ☀️
  android: { .success { _ in .failure(TestError.open) } },      // - -
  iOS: { .success { _ in .success(()) } }                       // ✅ ✅
).open(url, platform: .iOS)

..

{{< /highlight >}}


And that was all for today, hope you did enjoy reading :)


# References

* [Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/)
* [Pointfree: Algebraic Data Types](https://www.pointfree.co/collections/algebraic-data-types/algebraic-data-types)

[simctl]: <https://nshipster.com/simctl/>
[adb]: <https://developer.android.com/studio/command-line/adb>