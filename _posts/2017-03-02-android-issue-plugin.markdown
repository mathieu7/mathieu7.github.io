---
layout: post
title:  "Android Issue Tracker Plugin"
date:   2017-03-02 23:18:51
categories: android intellij plugin
disqus: true
---

I've spent the last few months developing a plugin for Android Studio in my spare time. It is still in its infancy, but with some more improvements I think people could use it even more effectively.

One of my motivations for creating the plugin were to solve a lingering problem that has come up several times during my development on Android. When I working at OkCupid, a common problem was breaking changes from one version of Android to another that would require a sizeable amount of time to dig into. These types of problems would manifest during development, testing, or just by happenstance. If a particular component or dependency didn't work, it would take a long time to search forums for device manufacturers, AOSP issues, user communities - anything that would shed some light on why something wasn't working as expected.

Here's a real life example from a job interview I was part of months ago. I was interviewing for Tumblr, and had to complete a take home assignment that made use of SwipeRefreshLayout, a component that is part of Android Support Libraries (many thanks to the team that works on this to get backward compatibility to older versions that are still seeing support). I had 24 hours to build this take home project, and wanted to use SwipeRefreshLayout for its intended functionality and also as a quasi-progress bar that would appear when the app was fetching its data through the Tumblr API. 

So, in order to display the SwipeRereshLayout as my loading indicator, I wrote something to the effect of ```mSwipeContainer.setRefreshing(true);``` in my Activity's ```onCreate()``` method. However, in testing, the expected behavior didn't happen. The widget wouldn't render. I tried a few things, and eventually settled on a workaround posted [here](https://code.google.com/p/android/issues/detail?id=77712&can=1&q=SwipeRefreshLayout%20setRefreshing&colspec=ID%20Status%20Priority%20Owner%20Summary%20Stars%20Reporter%20Opened).

The issue has since been fixed, but it made for an interesting discussion to have while explaining my code during my interview.

I could have just created another progress bar to display while the app was loading for the first time, but that wasn't a good enough deal breaker (plus it kept the feel of the app consistent with Material Design). Anyway, this is probably one of the simplest examples, but one of many I had come across over the years. 

So, another motivation for this plugin was, why search manually for these issues? Android Studio already works well with most packaged Android tools, so why not tie it to Android's issue tracker? 

Another motivation was to simply learn the codebase for Android Studio, Intellij IDE and give back to the ecosystem that has improved so much over the years. I like digging into how software works, so this was a great time to put all these motiviations together! I've learned a tremendous amount, and there is much more to go.

### How the plugin works

The plugin makes requests to Android Issue Tracker and scrapes relevant AOSP issues from the site. It currently only works with "Assigned" issues (that is, issues that have been accepted and assigned to specific developers at Google), but I will make a user setting to fetch whatever issue statuses they would prefer. 

Once downloaded, the issues are cached locally and indexed for easy searching. On any string in your current open code, you can search for issues and it will search the index and provide you the relevant open and assigned issues.

![Screenshot 1](/assets/pluginscreenshot1.png)

Use Alt+F9 or right-click and "Find Issues" on the current token under your cursor.

![Screenshot 2](/assets/pluginscreenshot2.png)

A tool panel will appear containing issues containing that token, if any.

![Screenshot 3](/assets/pluginscreenshot3.png)

You can quickly view the current thread for the issue locally, in a browser, and get a quick link added to your clipboard.

My plans for the future include better sorting and filtering issues, allowing for finding workarounds, starring issues (so Google can notice many devs are awaiting a fix!), and gradle support (run this issue check as a gradle task for instance). 

Download the plugin from the supporting github site and let me know where I can improve even more [here](https://mathieu7.github.io/android-issue-plugin)



