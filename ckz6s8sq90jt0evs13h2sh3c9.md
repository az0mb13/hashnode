## Hacking a Chrome Extension for Fun and Profit

## Introduction

Google chrome extensions are a bundle of multiple JavaScript, HTML, and CSS files, much like a web app but inside your browser and interacting with the pages or providing functionalities to enhance your browsing experience.

In this post, I'll be talking about an extension called Raindrop, which is one of those neat Bookmark managers for Google chrome, and believe me, it works really well in increasing productivity and keeping a track of things in an organized way. I'll be explaining how I exploited Raindrop's Google Chrome extension to use their premium services for free.

* * *

## Background

Raindrop is one of the most famous Bookmark manager apps that's being used by more than 100,000+ users according to its Chrome Extension page here -

<figure class="kg-card kg-image-card"><img src="https://imgur.com/u9qRYz4.png" class="kg-image" alt="chrome page" loading="lazy"></figure>

It also has one of the nicest web interfaces at [https://app.raindrop.io/](https://app.raindrop.io/). On top of all that, it's completely open sourced at [https://github.com/raindropio](https://github.com/raindropio) which attracted me towards using it.   
So how this extension works is you add it to your Google chrome, and then bookmark any of your pages using this extension. This will be saved in your browser's extension as well as on Raindrop's Web app because the same API is being used to interact with the application.

Raindrop features two plans, one is free and the other one is paid. The paid plan comes with some extra features which you can find on their website. One of the most useful features in their paid plan is creating nested collections for bookmarks. This is what I'll be bypassing today.

* * *

## Finding a needle in the Haystack

First of all, this is an unlisted Extension and it can't be searched in Google Chrome extensions anymore. You can still find its other extension in the Chrome store which is _not vulnerable_. Let me rephrase that to obfuscated.

I'll start with installing the extension normally from the Chrome store.  
When this extension is installed for the first time, a screen like this will be shown in a new tab after signing up and logging in

<figure class="kg-card kg-image-card"><img src="https://imgur.com/207vuvq.png" class="kg-image" alt="dash" loading="lazy"></figure>

And when we try to create a Nested Collection from the sidebar, in the free account, an error like this will be shown which I'll be bypassing soon.

<figure class="kg-card kg-image-card"><img src="https://imgur.com/bRkoZbs.png" class="kg-image" alt="error" loading="lazy"></figure>

I already checked my Burp to see if Pro validations are being done on any API endpoint but couldn't find any which probably means that it's happening somewhere in the source code on the client-side.

Let's fire up the Chrome debugger to look deeper into the source to see where the actual validation is happening.  
Going to the Sources tab, I can see a partial directory structure of the extension. From the looks of it, it seems the application logic is inside `app.js`.

<center><figure class="kg-card kg-image-card"><img src="https://imgur.com/kWCWBUS.png" class="kg-image" alt="app.js" loading="lazy" style="width:50%"></figure></center>

Now since I'm looking for the function which validates if a user is Pro or Free, I'll be searching for the word `Pro` inside the JS file. This gave around 932 matches which was too much to manually go through so I turned on "Match Case" which reduced the potential matches to 233 and also gave me what I was looking for. The name of the function which looks like it is the one validating if a user is Pro.

<figure class="kg-card kg-image-card"><img src="https://imgur.com/E3whIQn.png" class="kg-image" alt="search" loading="lazy"></figure>

So the name of the function is `isPro()`. Now onto the function definition which quickly brought me to this little part where the function is defined.

<center><figure class="kg-card kg-image-card"><img src="https://imgur.com/sJF31eH.png" class="kg-image" alt="ispro" loading="lazy" style="width:50%"></figure></center>

From the definition, it is obvious that the function is checking if the user is logged in and if the user is a pro user, this will return true, otherwise false. Now, all we need to do to bypass the Pro validation is to make this function return true. Let's verify this by using Chrome's Debugger.  
I'll set a break-point on the function and invoke a Pro feature (Nested collection) to see if the application flow stops on my break-point. To create a break-point, just click on the line number in the JS file and it'll be shown on the right side as shown below.

<figure class="kg-card kg-image-card"><img src="https://imgur.com/CCQnX8m.png" class="kg-image" alt="breakpoint1" loading="lazy"></figure>

Now I'll invoke the Pro feature from the sidebar. It can be seen from the below screenshot that the execution halted on this exact function.

<figure class="kg-card kg-image-card"><img src="https://imgur.com/xab5rRn.png" class="kg-image" alt="breakpoint2" loading="lazy"></figure>

I'll step into the next function call to see the return value and it can be seen in the Scope section that false was returned since I'm not a pro user.

<figure class="kg-card kg-image-card"><img src="https://imgur.com/oayc6m2.png" class="kg-image" alt="breakpoint3" loading="lazy"></figure>

This confirms that it is the same function that's validating if a user is on a Pro or a Free plan.  
Now onto the next part.

* * *

## Taking things apart

Here I'll modify the extension, bypass Chrome's Extension Content verification and Load our Unpacked extension back into the browser.

On Linux, the path where Chrome extensions are stored is usually in 
- `/home/<user>/.config/google-chrome/Default/Extensions/<extension_id>`

The extension ID can be obtained from the Extensions page in Chrome 
- `chrome://extensions/`

I'll copy that folder from the path to any other location and will be making changes in this one. Now Google Chrome does not allow modified extensions to be directly installed and will give an error saying that `This extension may have been corrupted`.

To do this I need to modify some of the files inside the source. Here's what's needed to be done.

- Disable or delete the original extension in Chrome. 
- Delete `_metadata`.
- Edit `manifest.json` and remove _`key`_ and _`update_url`_ sections.
- Change the `name` and the _`short_name`_ fields to something else which will be easily identifiable in Chrome so as to not get confused with the disabled original extension if it's not already deleted. 

Now I just have to find the `isPro()` function inside `app.js` and make it always return true as shown below.

<figure class="kg-card kg-image-card"><img src="https://imgur.com/KHfVdjE.png" class="kg-image" alt="return" loading="lazy"></figure>

To proceed further, I need to save all my changes including the ones in the manifest file and the app.js file.

Heading over to `chrome://extensions/` I enabled Developer mode and loaded the unpacked extension folder from the top-left option.

Now when I create a Nested collection it'll happily allow me to do so since the validation function will always return true. This also bypasses other Pro features for the extension and the application.

<figure class="kg-card kg-image-card"><img src="https://imgur.com/GOWIWMP.png" class="kg-image" alt="pro" loading="lazy"></figure>

These changes will also be synced with the server which I think is another bug that automatically creates Pro resources due to a missing server-side validation on the endpoint.

<figure class="kg-card kg-image-card"><img src="https://imgur.com/Sp0eyGt.png" class="kg-image" alt="server" loading="lazy"></figure>