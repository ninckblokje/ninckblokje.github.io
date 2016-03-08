---
layout: post
title:  "Oracle JET packaged as Chrome Extension"
tags:
- chrome
- oraclejet
disqus: true
---

## Oracle JET
[Oracle JET](https://github.com/oracle/oraclejet) is a recently open sourced collection of frameworks and custom code by Oracle for rapidly creating (enterprise) Javascript websites and apps. It consists of the following frameworks:

- Hammer
- JQuery
- JQuery UI
- Knockout
- RequireJS

Oracle has also written its own components and added them to Oracle JET.

## Chrome app vs Chrome extension
I wanted to package Oracle JET as a Chrome packaged app, however I ran into some problems. So I decided to first start with a Chrome extension and later try again packaging Oracle JET as a packaged app. The permission model of a packaged app and an extension differ somewhat, which causes some problems with Oracle JET. More on Chrome permissions further on.

## Building the Chrome extension
It is actualy quite easy to package Oracle JET as a Chrome extension! The following is required:

- A **manifest.json** file describing the extension
- An image file
- CSS values for **body.height** and **body.width**

That's all! The project can be found in [GitHub](https://github.com/ninckblokje/oraclejet-chrome-extension).

I created a sample Oracle JET project with the basic template by following the instructions [here](http://www.oracle.com/webfolder/technetwork/jet/globalGetStarted.html). I also had to install **bower** and **grunt-cli**.

### manifest.json
I used the following **manifest.json** (a complete description can be found at the [developers guide](https://developer.chrome.com/extensions/manifest)).

    {
	  "manifest_version": 2,

	  "name": "Oracle JET extension",
	  "description": "Oracle JET packaged as a Chrome extension",
	  "version": "1.0",
	  "icons": { "256": "css/images/oraclejet-256.png" },

	  "browser_action": {
	    "default_icon": "css/images/oraclejet-256.png",
	    "default_popup": "index.html"
	  },
	  "permissions": [],
	  "content_security_policy": "script-src 'self' 'unsafe-eval'; object-src 'self'"
	}

This is a fairly basis manifest for a Chrome extension. I added a custom image file (**oraclejet-256.png**) and use it as an icon for the extension on the extension page by specifying the **icons** tag and I use it as an icon in the browser bar by specifying **browser_actions/default_icon**. No special permission are required, however the last line is very interesting:

    "content_security_policy": "script-src 'self' 'unsafe-eval'; object-src 'self'"

Omitting this line will cause an error while using Oracle JET in the extension:

![Broken extension]({{ site.github.url }}/assets/oraclejet-chrome-extension/permission_error_extension.png)

![Error message]({{ site.github.url }}/assets/oraclejet-chrome-extension/permission_error.png)

This is caused by Knockout, which uses Java script **eval**. Chrome blocks this for extensions and for packaged apps. However for extensions it is possible to enable the use of **eval** by adding **content_security_policy** to the manifest.

### CSS
To size the extension correctly the height and width of the body must be specified. This can be done in the CSS file **override.css** (located in the **css**) folder by adding the following CSS:

    /* Chrome extension */
	body {
	  height: 600px;
	  width: 358px;
	}

Oracle JET resizes itself automatically to the correct resolution and switches to a mobile friendly interface. This interfaces also works nicely for the extension!

### Result
After importing the Chrome extension the result looks like this:

![Extension]({{ site.github.url }}/assets/oraclejet-chrome-extension/result.png)

## Chrome & Optimus / multi monitor
I spend quite a lot of time figuring out why my extension was not rendering correctly. I finally figured it out, it was my multi monitor setup. By moving Chrome to the primary monitor my extension rendered correctly. I am not sure if this is because of the multi monitor setup or because of my NVidia Optimus graphic gards (GPU switching on a laptop).

![Rendering error]({{ site.github.url }}/assets/oraclejet-chrome-extension/optimus_rendering_extension.png)
