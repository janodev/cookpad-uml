# Universal links in Global-iOS

<img width="800" src="overview.png"/>

Features

- Explicit handling of application states (initial/launched)
- Receives universal links from Handoff, SiriKit, and Spotlight search
- Maps URLs to native screens. Falls back to pass the URL to iOS or Safari.
- Runs an authorization flow if needed before opening a URL
- Allows URL path localization for SEO purposes
- Handles [App deep linking with goo.gl](https://developers.google.com/search/blog/2015/05/app-deep-linking-with-googl) and [Firebase links](https://firebase.google.com/docs/dynamic-links)
- Logs URLs and UTM parameters

## Pseudocode

① **AppDelegate**

- Ignore the URL if the application is launching
- Call Firebase to log incoming URLs and user activities
- Call the Coordinator 
    - to process a possible OAuth URL
    - to start a flow based on the URL

② **TabCoordinator**

- Ask the AppLinkHandler to map the URL to a UniversalLink
    - If successful, ask the interactor to process the link
    - If failure, try to open the URL as instructed by AppResponderURLOptions

One exception: if the currently presented controller is not interruptible, the coordinator doesn’t try to open a native screen. *Interruptible* means being marked `ViewCoordinatorInteruptable`.

③ **TabInteractor**

The TabInteractor checks if the link requires authentication

- If it does, the interactor asks the tab coordinator to run the login flow and try processing the link again.
- If it doesn’t, tell the PendingExternalActionHandler to process the link.

<details><summary>Communication between coordinator/interactor/handler happens through delegation.</summary>

<img width="800" src="delegation.png"/>
</details>

④ **PendingExternalActionHandler**

Opening the link depends on the app’s state (initial/launched).

- If the app is active, call interactor.performLink(UniversalLink)
- If the app is not active, store the link as `lastAction`. When the application becomes active, the `lastAction` is called from the TabInteractor.viewDidAppear and didBecomeActiveNotification.

⑤ **UniversalLinkDetector** maps a URL to a `UniversaLink` that points to a native screen. Each universal link has a descriptor able to create it, and the detector tries all of them until finding one that works.

<details><summary>The descriptor of a link may have PathComponent objects, which are paths of a URL translated for SEO purposes. For instance, the https://cookpad.com/search is https://cookpad.com/buscar in Spanish.</summary>

<img width="800" src="pathcomponent.png"/>
</details>

## Other objects involved

- **AppContext** holds all the dependencies of the application. Each component defines its dependencies using composition of protocols. So even when they receive the full object, they only have access to what they need.

- **AppResponderURLOptions** represents the ways to open a URL. The `default` property is a set of native, Safari, and open url. The first option available wins.
    <ul>
      <li>Native: opening a native screen</li>
      <li>Safari: ask Safari to open the link</li>
      <li>Open URL: ask iOS to open the link</li>
      <li>Universal links only: open as native screen only, don’t fall back to other ways.</li>
    </ul>

- **State** represents the state of the app launching/launched. This prevents trying to handle a link while the application is still launching. When the app is launched, the state holds the app dependencies (AppContext) and root coordinator (TabCoordinator).

- **TabCoordinator**. A **coordinator** is an object that manages navigation and instantiation of controllers. Each coordinator can be considered a small application, where controllers don’t know each other directly. In Cookpad coordinators are view controllers, which load other view controllers nesting them full screen.

Why isn’t `AppResponderURLOptions` a enum? 

<details><summary>It could be, not a big difference. It is a struct now.</summary><img width="200" src="AppResponderURLOptions.png"/></details>

Modeling with structs vs enum
<ul>
  <li>Using a struct 
	  <ul>
		  <li>allows adding more cases without editing the code</li>
		  <li>allows having a member that combines several instances (instead of using an array)</li>
	  </ul>
  <li>Using an enum
    <ul>
      <li>requires minimal effort to add another case if the source is private</li>
      <li>causes compiler errors indicating where to handle new cases</li>
      <li>lets you define new cases without causing compiler errors by using `@unknown default`</li>
    </ul>
  </li>
</ul>

## Logging

<img width="800" src="logging.png"/>

“UTM” stands for [Urchin Tracking Module (UTM)](https://en.wikipedia.org/wiki/UTM_parameters) parameters. They are URL parameters that indicate sources of traffic to an application.

## How to add a new link

This figure is an example of adding a universal link to a search tab that doesn’t need authorization.

<img width="800" src="searchTab.png"/>

# Universal links
	
## What is a universal link?

**Universal links** are HTTP or HTTPS URLs that Apple's operating systems recognize as pointing to resources on the web or in your app. For instance, a URL http://cookpad.com/search/chicken can be loaded as HTML on the web, or as a native controller in a native app.

**To create a universal link:**

- Configure your app by adding an Associated Domain entitlement, plus the code to handle links in the app/scene delegate.
- Configure your website by adding an associated domain file.

Because the entitlement mentions your domain, and the file on your domain mentions your app, this creates a secure two-way association, allowing your app to perform particular tasks on behalf of your website.

The flow to load a universal link is as follows:

1. The user installs an application. If the app has an associated domain capability, iOS downloads the associated domain file for the domain.
2. The user loads a standard web link in an Apple app like mobile Safari, notes, iMessage. iOS checks if this domain is associated with an installed app.
    - If it isn’t, the URL is loaded in Safari.
    - If it is, the app is launched without ever loading the web page. It’s up to the app to recognize the URL and offer related content.

Universal links are supported in iOS, tvOS, and macOS.

## What is an associated domain?

An **associated domain** connects a domain name and an iOS application.

**To create an associated domain:**

- Configure your app with the associated domain capability.
- Host an AASA (apple-app-site-association) file in the webserver of your domain. This file should contain the bundle id of the app and the paths the app should open.

An associated domain is required to enable the following features:

- App Clips: an App Clip is a small part of an app that lets you do a task quickly. e.g., pay for food, rent a bike. See more.
- Handoff: continue within your apps and websites across multiple devices. Apps that support universal links automatically work with Handoff without requiring further configuration.
- Shared web credentials: sign in to apps using credentials saved in Safari with iCloud Keychain.
- Universal links

## How deep links arrive

The app delegate receives a universal link when the user

- types a URL in Safari
- opens Safari to invoke a Handoff passed from another device
- talks to Siri to trigger a SiriKit intent

<table>
<tr>
  <td>
    <img width="600" src="appdelegate.png"/>
  </td>
  <td>
    <p>
    // Asks the delegate to open a URL
    <a href="https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623112-application">application(_:open:options:)
    </a>
    </p>
    <p>
    // Asks the delegate to handle a *user activity* (handoff, SiriKit intent, Spotlight search)
    <a href="https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623072-application">application  (_:continue:restorationHandler:)</a>
    </p>
  </td>
</tr>
</table>
