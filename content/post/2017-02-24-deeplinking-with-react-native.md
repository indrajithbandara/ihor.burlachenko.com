+++
date = "2017-02-24"
title = "Deep Linking with React Native"
categories = ["React Native", "Howto"]
+++

Recently I added deep linking integration to one of my projects and I want to share my experience with you. It is a complete guide which covers all the steps and explains how to add deep links both on iOS and Android. I'll use [React Native Router Flux](https://github.com/aksonov/react-native-router-flux). It is a very nice navigation library and at the moment of writing, I couldn't find anything which I would like more. In case you are using something else, this tutorial still will be useful for you as it doesn't rely on the navigation library but describes how to integrate deep links to your application.

### What is deep linking?

Deep links allow us to link to specific screens in a mobile application rather than simply launching the app. [fb://profile/4](fb://profile/4) is an example of a deep link. If you have Facebook installed on your phone try opening that link. It will launch the Facebook application and show you Mark Zuckerberg's profile.

### Example application

I prepared a example application which has two screens. On the first one we are offered to enter our name and when we submit it we are taken to the second screen where we can see a greeting message. Our goal will be to allow deep links with parametrized names to the greeting screen. You can check the sources [here](https://github.com/ihor/ReactNativeDeepLinkingExample).
<p align="center">
    <img src="/img/deeplinking/example-app.gif" alt="Sample Application" title="Example application" height=500/>
</p>

### Deep linking with React Native

In order to process incoming links in our React application, we need to use [Linking API](https://facebook.github.io/react-native/docs/linking.html) from React Native. According to the official docs, to handle initial URL which was used to open our application we can use the following code:
```jsx
Linking
    .getInitialURL()
    .then(url => handleOpenURL({ url }))
    .catch(console.error);
```
To handle incoming URLs when the app is running in background we have to attach a listener which will be called when such event occurs:
```jsx
Linking.addEventListener('url', handleOpenURL);
```

```handleOpenURL``` has to navigate us to the corresponding scene based on URL which it will receive with the event. I'll use [crossroads](https://millermedeiros.github.io/crossroads.js/) to parse the route. This routing library does too much for such simple application, but it might become very useful when your application gets bigger:

```jsx
let scheme = 'exampleapp';
handleOpenURL(event) {
    if (event.url && event.url.indexOf(scheme + '://') === 0) {
        crossroads.parse(event.url.slice(scheme.length + 3));
    }
}
```

We can put this code anywhere we want. For our example project let's create a new [LinkedRouter](https://github.com/ihor/ReactNativeDeepLinkingExample/blob/master/app/components/LinkedRouter.js) component and listen for incoming links at ```componentDidMount```:
```jsx
import React from 'react';
import { Linking } from 'react-native';
import { Router } from 'react-native-router-flux';

class LinkedRouter extends React.Component {
    constructor(props) {
        super(props);

        this.handleOpenURL = this.handleOpenURL.bind(this);
    }

    componentDidMount() {
        Linking
            .getInitialURL()
            .then(url => this.handleOpenURL({ url }))
            .catch(console.error);

        Linking.addEventListener('url', this.handleOpenURL);
    }

    componentWillUnmount() {
        Linking.removeEventListener('url', this.handleOpenURL);
    }

    handleOpenURL(event) {
        if (event.url && event.url.indexOf(this.props.scheme + '://') === 0) {
            crossroads.parse(event.url.slice(this.props.scheme.length + 3));
        }
    }

    render() {
        return <Router { ...this.props }/>;
    }
}

LinkedRouter.propTypes = {
    scheme: React.PropTypes.string.isRequired
};
```

The only thing which is left is mapping incoming URLs to the corresponding scenes. Nice place to add this code is our [router configuration](https://github.com/ihor/ReactNativeDeepLinkingExample/blob/master/app/router.js). Here we assign ```Greeting``` screen to the ```greetings/{name}``` route and pass the ```name``` parameter as the prop:

```jsx
import React from 'react';
import { Scene, Actions } from 'react-native-router-flux';
import crossroads from 'crossroads';

import LinkedRouter from './components/LinkedRouter';
import HomeScreen from './components/HomeScreen';
import GreetingScreen from './components/GreetingScreen';

const scenes = Actions.create(
    <Scene key="root">
        <Scene key="home" title="Home" component={HomeScreen} initial={true}/>
        <Scene key="greeting" title="Greeting" component={GreetingScreen}/>
    </Scene>
);

// Mapping incoming URLs to scenes
crossroads.addRoute('greetings/{name}', name => Actions.greeting({ name }));

export default <LinkedRouter scenes={scenes} scheme="exampleapp"/>;
```

At this point, our React application is ready to handle incoming links. There is a little configuration work left to get it working on iOS and Android.

### iOS deep linking

On iOS, we'll need to link ```RCTLinking``` library which comes with React Native to our project. For this we'll have to do the following steps:

1. Open project ```*.xcodeproj``` with XCode.
2. Drag ```RCTLinking.xcodeproj``` from ```node_modules/react-native/Libraries/LinkingIOS``` to the project ```Libraries```
<p align="center">
    <img src="/img/deeplinking/ios-step-2.png" alt="Add RCTLinking to the project Libraries" title="Add RCTLinking to the project Libraries"/>
</p>
3. Click on your main project file (the one that represents the .xcodeproj) select ```Build Phases``` and drag the static library from the ```RTCLinking``` ```Products``` folder to ```Link Binary With Libraries```
<p align="center">
    <img src="/img/deeplinking/ios-step-3.png" alt="Link RCTLinking with binaries" title="Link RCTLinking with binaries"/>
</p>
4. Click on your main project file again, select ```Build Settings```, search for ```Header Search Paths``` and put ```$(SRCROOT)/../node_modules/react-native/Libraries``` there
<p align="center">
    <img src="/img/deeplinking/ios-step-4.png" alt="Add header search path" title="Add header search path"/>
</p>
5. Click on your main project file one more time, select ```Info``` and add a URL type at the bottom. We'll put ```exampleapp``` there.
<p align="center">
    <img src="/img/deeplinking/ios-step-5.png" alt="Added URL scheme" title="Added URL scheme"/>
</p>

If you want to listen to incoming app links during your app's execution you'll need to add the following lines to the *AppDelegate.m file:
```objectivec
 #import "React/RCTLinkingManager.h"

 - (BOOL)application:(UIApplication *)application openURL:(NSURL *)url
   sourceApplication:(NSString *)sourceApplication annotation:(id)annotation
 {
   return [RCTLinkingManager application:application openURL:url
                       sourceApplication:sourceApplication annotation:annotation];
 }

 // Only if your app is using [Universal Links](https://developer.apple.com/library/prerelease/ios/documentation/General/Conceptual/AppSearch/UniversalLinks.html).
 - (BOOL)application:(UIApplication *)application continueUserActivity:(NSUserActivity *)userActivity
  restorationHandler:(void (^)(NSArray * _Nullable))restorationHandler
 {
  return [RCTLinkingManager application:application
                   continueUserActivity:userActivity
                     restorationHandler:restorationHandler];
 }
```
Now you can open Safari and navigate to ```exampleapp://greetings/World```. You'll be taken to the Greeting screen:
<p align="center">
    <img src="/img/deeplinking/deeplinking-on-ios.gif" alt="Deeplinking on iOS" title="Deeplinking on iOS" height=500/>
</p>

### Android deep linking

In order to allow deep linking to the content in Android, we need to add intent filters to respond to action requests from other applications. Intent filters are specified in your android manifest located in your React Native project at ```android/app/src/main/AndroidManifest.xml```. Here is the modified manifest with the intent filter added to the main activity:

```xml
<activity
    android:name=".MainActivity"
    android:label="@string/app_name"
    android:configChanges="keyboard|keyboardHidden|orientation|screenSize">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>

    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="exampleapp"
            android:host="greetings"
            android:pathPrefix="/" />
    </intent-filter>
</activity>
```

After updating the manifest file you can launch the application on Android Virtual Device and execute the following command in a terminal to test the deep link:

```bash
adb shell am start -a android.intent.action.VIEW -d "exampleapp://greetings/World" com.reactnativedeeplinkingexample
```
The same as with iOS application you should see the "Hello World" message.

### Summary

That's pretty much it. As with many other things with React/React Native it was very easy to add deep links to our example app. I used React Native Router Flux navigation library but as you saw it was all about implementing the ```handleOpenURL``` function and it shouldn't be a problem to add deep linking to your application following steps from this tutorial.

### Links

- [Sample Application sources](https://github.com/ihor/ReactNativeDeepLinkingExample)
- [React Native Linking docummentation](https://facebook.github.io/react-native/docs/linking.html)
- [React Native Router Flux navigation library](https://github.com/aksonov/react-native-router-flux)
- [Crossroads routing library](https://millermedeiros.github.io/crossroads.js/)