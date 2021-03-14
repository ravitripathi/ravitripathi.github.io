---
title: Haptic Feedback on UI Components
date: 2020-08-02 00:00:00 +0530
tags: [swift, haptics, iOS]
---

Beginning with the iPhone 7, numerous Apple devices ship with a vibration motor dubbed the _"Taptic Engine"_, which can generate vibrations with great efficiency. A number of built-in UIKit components utilize it for adding haptic feedbacks on their interactions. iOS provides some APIs which allow you to use haptics for adding delightful feedbacks to your own animations. Let's take a look at them and some interactions which could be a good place to use them.

## `UIFeedbackGenerator`

Quoting [the docs](https://developer.apple.com/documentation/uikit/uifeedbackgenerator), `UIFeedbackGenerator` is _the abstract superclass for all feedback generators._ You can use one of the concrete subclasses of `UIFeedbackGenerator` to generate a vibration feedback:

- `UINotificationFeedbackGenerator`
- `UIImpactFeedbackGenerator`
- `UISelectionFeedbackGenerator`

### `UINotificationFeedbackGenerator`

As the name suggests, this class is used to give user feedback when an notification is displayed. It provides vibration feedbacks for 3 scenarios, `.success`, `.warning` and `.error`.

![Error Animation](/assets/postAssets/errorKeyFrame.gif)

Animations like these are ofen used to draw a user's attention to the screen in scenarios where a provided input may be incorrect. A vibration feedback similar to one provided when an incorrect password is entered on the lockscreen can be added by simply calling:

```swift
let generator = UINotificationFeedbackGenerator()
generator.notificationOccurred(.error)
```

### `UIImpactFeedbackGenerator`

When building views which collide with each other or the screen bounds, `UIImpactFeedbackGenerator` allows you to add vibration feedback analogous to the perceived weight of the component. 

![Bounce Animation](/assets/postAssets/bounceanimation.gif)

A large view like this one can be timed to generate a strong vibration feedback every time it touches the screen's bottom bound. Again, the call for generating an impact feedback is very straightforward:

```swift
let impactGenerator = UIImpactFeedbackGenerator(style: .heavy)
impactGenerator.impactOccurred()
```

### `UISelectionFeedbackGenerator`

Selection feedback is perhaps the most frequently encountered haptic feedback on iOS. Switching between menus and toggling views which indicate state are perfect places for generatinng a selection feedback. 

```swift
 let selectionGenerator = UISelectionFeedbackGenerator()
 selectionGenerator.selectionChanged()
 ```
 <video width="320" height="240" autoplay loop muted playsinline>
  <source src="/assets/postAssets/iospicker.mov" type="video/mov" />
</video>