---
layout: post
title:  "AppDelegate and Single Responsibility Principle"
date:   2016-04-28 13:00:00 +0100
categories: swift
description: "As our Project grew, the AppDelegate got messier and messier. Now the file got 1000+ lines, uncountable delegates and helper methods and no one has any clue what is going on in here. This class is responsible for so many things that it seems there is no way to conform to the Single Responsibilty Principle. On top of that we want to convert the AppDelegate from Obj-C to Swift. We had to come up with a solution ..."
author: "@gerritmenzel"
---

As our Project grew, the AppDelegate got messier and messier. Now the file got 1000+ lines, uncountable delegates and helper methods and no one has any clue what is going on in here. We needed to refactor it. Initially the project was setup as an Obj-C project but every new bit we write is in Swift. It is unlikely that we will refactor the AppDelegate all at once so we had to come up with a solution. First I had to identify what we are doing in the AppDelegate:

- Lots and lots of header files
- constant keys
- properties
- Network monitoring (AFNetworking)
- Tracking (GA, GTM, Adjust)
- Crash reporting (HockeyApp)
- Handoff (AppleWatch)
- Local notifications (Beacons)
- Remote Notifications
- Window initialisation
- Deep linking

Most of this is initialised in `didFinishLaunchingWithOptions`. The AppDelegate is responisibile for a lot of stuff. Come to think about it, how can we conform to the [Single Responsibility Principle](https://realm.io/news/donn-felker-solid-part-1/) within the AppDelegate if the AppDelegate is the entry point for our App. 
My Solution is to break things down to AppDelegateComponents. First we create a class (keeping the backwards compatibility to Obj-C) which conforms to the same Protocol as AppDelegate does.

``` swift
// HockeyappAppDelegateComponent.swift
// Swift 2.1
class HockeyappAppDelegateComponent: NSObject, UIApplicationDelegate {
		
}
```

there we can only implement the delegate methods we really need for our HockeyApp implementation

``` swift
func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject : AnyObject]?) -> Bool {
	BITHockeyManager.configure(self)
	return true
}
```

For the configuration of the BITHockeyManager I wrote a extension and I only need to supply a BITHockeyManagerDelegate. 
In the same class I can now handle the delegate calls from BITHockeyMangerDelegate.

``` swift
extension HockeyappAppDelegateComponent: BITHockeyManagerDelegate {
    
	func userIDForHockeyManager(hockeyManager: BITHockeyManager!, componentManager: BITHockeyBaseManager!) -> String! {
	    ...
	}

	func userNameForHockeyManager(hockeyManager: BITHockeyManager!, componentManager: BITHockeyBaseManager!) -> String! {
	    ...
	}
}
```

All code resposibile for my HockeyApp implementation now lives in this one class. The beauty of this is, I can refactor one module at a time. If there is some spare time, or I need to adjust some code in AppDelegate I can now refactor it. 
The still missing piece is the call from the AppDelegate. First I created one property to store all my components: `@property (nonatomic) NSArray<UIApplicationDelegate> *components;`
Then I lazy initialied it with all my components. Now for every delegate method that I need in my components I have to go through my array an call it with its parameters.

``` objective-c
// AppDelegate.m
@implementation HMTAppDelegate

-(NSArray<UIApplicationDelegate> *)components {
    if (!_components) {
        _components = (NSArray<UIApplicationDelegate>*)@[[HockeyappAppDelegateComponent new],
                                                         [AdjustAppDelegateComponent new],
                                                         [GoogleTagManagerAppDelegateComponent new],
                                                         [GoogleAnalyticsAppDelegateComponent new],
                                                         [HandoffAppDelegateComponent new]];
    }
    return _components;
}

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
	[self.components enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        if ([obj respondsToSelector:@selector(application:didFinishLaunchingWithOptions:)]) {
                [obj application:application didFinishLaunchingWithOptions:launchOptions];
        }
    }];

    return true
}

...

@end
```

Looking at this code I thought, this is pretty ugly but thats objective-c ... I guess ... 
A few days later it already annoyed me. Enumarating in all delegates over components could be outsourced. I wanted only one line to handle all that.
So I created a SequenceType extension:

``` swift
// Swift 2.1
extension SequenceType where Generator.Element == UIApplicationDelegate {
	func application(application: UIApplication, didFinishLaunchingWithOptions: [NSObject : AnyObject]?) -> Bool {
        for delegate in self {
            delegate.application?(application, didFinishLaunchingWithOptions: didFinishLaunchingWithOptions)
        }
        return true
    }

    func applicationDidEnterBackground(application: UIApplication) {
        for delegate in self {
            delegate.applicationDidEnterBackground?(application)
        }
    }

    ...
}
```
Unfortunately Objective-C cannot handle generics so I had to build an Obj-C wrapper. I marked it, so if the AppDelegate will someday be in swift I can delete the wrappers

``` swift
extension NSArray {
    func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject : AnyObject]?) -> Bool {
        if let array = self as? [UIApplicationDelegate] {
            return array.application(application, didFinishLaunchingWithOptions: launchOptions)
        }
        return true
    }
}
```

But now in AppDelegate I can call just

``` objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
	return [self.components application:application didFinishLaunchingWithOptions:launchOptions];
}
```

I only showcased the didFinishLaunchingWithOptions but i repeated the steps for all the Delegate methods my components need. Just to give you an idea here are some of my components

``` swift
import Foundation
import AdjustSdk

class AdjustAppDelegateComponent: NSObject, UIApplicationDelegate {
    func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject : AnyObject]?) -> Bool {
        Adjust.configure()
        Adjust.trackAppOpened()
        return true
    }
    
    func applicationWillEnterForeground(application: UIApplication) {
        Adjust.trackAppOpened()
    }
}
```

I even can declare properties in my components and they only live in this context and not my whole AppDelegate

``` swift
import Foundation

protocol TAGManagerDataSource {
    var tagManager: TAGManager? { get set }
    var container: TAGContainer? { get set }
}

class GoogleTagManagerAppDelegateComponent: NSObject, TAGManagerDataSource {
    var tagManager: TAGManager?
    var container: TAGContainer?
}

extension GoogleTagManagerAppDelegateComponent: UIApplicationDelegate {
    func application(application: UIApplication, didFinishLaunchingWithOptions launchOptions: [NSObject : AnyObject]?) -> Bool {
        tagManager = TAGManager.instance()
        
        if let tagManager = tagManager {
            tagManager.logger.setLogLevel(kTAGLoggerLogLevelNone)
            
            TAGContainerOpener.openContainerWithId("GTM-XXXXXX", tagManager: tagManager, openType: kTAGOpenTypePreferFresh, timeout: nil, notifier: self)
        }
        return true
    }
}

extension GoogleTagManagerAppDelegateComponent: TAGContainerOpenerNotifier {
    func containerAvailable(container: TAGContainer!) {
        dispatch_async(dispatch_get_main_queue()) { () -> Void in
            self.container = container
        }
    }
}
```

Refactoring this way we were able to minimize our AppDelegate and conform to the Single Responsibility Principle.

