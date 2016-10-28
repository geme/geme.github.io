---
layout: post
title:  "iOS ViewController Url Navigation"
date:   2016-10-27 12:00:00 +0100
categories: swift
description: "To navigate between ViewControllers can be a bit tricky and should be right when the projects starts. The problem is that there a to many ways you can handle navigation and there is no real best practice. So I started experimenting what could be a clean way. What I came up with is a web-like navigation with urls..."
author: "@gerritmenzel"
---

To navigate between ViewControllers can be a bit tricky and should be right when the projects starts. The problem is that there are to many ways you can handle navigation and there is no real best practice. So I started experimenting what could be a clean approach. What I came up with is a web-like navigation with urls.

You can get the full source code from my [github page](https://github.com/geme/url-navigation), here I just want to point out the critical parts and how to use it.

One idea of mine was it would be greate if I could define in the URL if we want to pop to the rootviewcontroller and then navigate to our desired viewcontroller or just push the viewcontroller onto the stack.
A great way would be the file scheme syntax:

push onto stack
`app://foobar`

pop to root and then push to stack
`app:///foorbar`

one reserved word would be `animated` which state could be `true`, `1`, `false`, `0`. As you can guess it defines if we want to transition with animation or not. 

The main idea behind this is using the `application(_ app: UIApplication, open url: URL, options: [UIApplicationOpenURLOptionsKey : Any] = [:]) -> Bool` method from `UIApplication`. What I was trying to do is parse the url match it to a ViewController and create a ViewModel from the query items.

This delegate starts the lookup by injecting the url and the table where we can find the naming to viewcontroller mapping.

The LookupTable is a `Protocol` where you have to define a method `viewController(for: String) -> UIViewController`. No real magic here. The next question is how do we inject values into the `ViewController` and its `views`. 

`app://foobar?title=foo`

so I created a `ViewModel` which implements the `Protocol` `UrlInitable` which says that the object has to implement a `init?(url: URL)` method. Here I parse the url query items and set my properties accordingly.

The `ViewController` now needs to implement the `Protocol` `ViewModelBindable`, which says we have a property `viewModel` of any type. What this `Protocol` and its default implementation does for me is supply a `bind(url: URL)` and a `bind(viewModel: T)` methods (`T` has to be a `UrlInitable` in this case). The parsing will do the rest.

Here is how it goes:
If the url is opened, it looks if there is a match between "foobar" and a ViewController in the LookupTable. Then it checks if the ViewController is a ViewModelBindable and we have query parameters. If so we will init the ViewController and bind the the viewModel with the url. This `bind` method will now initialize the ViewModel with the `url` and then call the `bind(viewModel: T)` which simply sets the propertie. 

Now we can simply use the `didSet` function of the propertie to assigne the `viewModel` to our views. 

Using this, what do we need to do to set up a new ViewController?

- create ViewController
-- implement `Protocol` `ViewModelBindable`
-- implement a property viewModel

- create a ViewModel
-- implement `Protocol` `UrlInitable`
-- implement `init?(url: URL)`

- add the string name and the viewcontroller to the Lookup Table

now you should be able to call `UIApplication.shared.open(URL(string: "app://viewControllerName?title=foobar")!, options: [:], completionHandler: nil)`

as a bonus the app is now completly deeplinkable.
open the same url in safari and the app should handle it in the same way.

If you dont want deeplinking for all or specific section of your app you can check the scheme `UIApplicationOpenURLOptionsKey` in `application(_ app: UIApplication, open url: URL, options: [UIApplicationOpenURLOptionsKey : Any] = [:]) -> Bool` and disable it if the opening app is not `app://`

If you dont want to set a viewModel in your ViewController just use the `Protocol` `UrlBindable`, this requires you to implement `bind(url: URL)` yourself and our done.

For more information See my [project on github](https://github.com/geme/url-navigation)

