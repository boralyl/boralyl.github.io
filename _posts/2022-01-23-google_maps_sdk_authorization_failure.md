---
title: "Google Maps SDK for Android: Authorization Failure"
excerpt: >
  A quick post documenting my tiny mistake that took me way too much time to diagnose.
categories:
  - programming
tags:
  - android
classes: wide
---

I've recently been working on rewriting an Android app in [Kotlin](https://kotlinlang.org/)
using the new [Jetpack Compose UI framework](https://developer.android.com/jetpack/compose).
One of the screens in the app utilizes Google Maps.

I followed the [documentation](https://developers.google.com/maps/documentation/android-sdk/start)
carefully and launched the app. The map loaded as a gray tile and in the logs I received
the following error:

```
2022-01-16 17:27:13.868 17817-18850/com.foobar.android E/Google Maps Android API: Authorization failure.  Please see https://developers.google.com/maps/documentation/android-api/start for how to correctly set up the map.
2022-01-16 17:27:13.872 17817-18850/com.foobar.android E/Google Maps Android API: In the Google Developer Console (https://console.developers.google.com)
    Ensure that the "Google Maps Android API v2" is enabled.
    Ensure that the following Android Key exists:
    API Key: "<my-api-key-recacted>"
    Android Application (<cert_fingerprint>;<package_name>): <my-cert-fingerprint-redacted>;com.foobar.android
```

After trying many combinations of verifying the api key was valid and trying no restrictions
and restrictions recommended by the documentation, it still wouldn't load. I scoured the
first few pages of search results in Google and nothing worked. I finally decided to
reach out to technical support for the Google Maps Platform as a last ditch effort.

I provided a self-contained sample demonstrating the problem, but to my surprise they
were not able to reproduce the issue with my sample code. In their response they asked
about my `local.properties` file that contained the api key since that sensitive file
was not provided in the sample code. In their response they mentioned:

> We have not seen how you added the API key in the local.properties file, but we would like you to note that you don't need to include quotation marks in the API key.

I went and double checked my `local.properties` and saw I had wrapped the api
key in quotes:

```
GOOGLE_MAPS_DEV_API_KEY="<my-api-key-redacted>"
```

I went ahead and removed the quotes and re-ran the app. Sure enough the map loaded with
no issue. So hopefully this helps someone else out there that makes this simple mistake,
as the error logged by the SDK didn't give any hint that the quotes were the problem.

_TLDR_: Don't wrap your values in your `local.properties` in quotes.

**UPDATE 2022-02-15:** It looks like I'm not the only one who has run into [this problem](https://github.com/google/secrets-gradle-plugin/issues/46).
Quotes will now automatically be removed in [version 2.0.1](https://github.com/google/secrets-gradle-plugin/releases/tag/v2.0.1).
