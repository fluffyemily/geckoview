---
layout: default
title: Configuring GeckoView for Automation
summary: How to set environment variables, Gecko arguments, and Gecko preferences for automation and debugging.
tags: [GeckoView,Gecko,mozilla,android,WebView,mobile,debugging,automation,config,configuration,environment,variables,arguments,preferences]
nav_exclude: true
exclude: true
---
## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

# Configuring GeckoView

GeckoView and the underlying Gecko engine have many, many options, switches, and toggles "under the hood".  Automation (and to a lesser extent, debugging) can require configuring the Gecko engine to allow (or disallow) specific actions or features.

Some such actions and features are controlled by the [`GeckoRuntimeSettings`](../javadoc/mozilla-central/org/mozilla/geckoview/GeckoRuntimeSettings.html) instance you configure in your consuming project.  For example, remote debugging web content via the Firefox Developer Tools is configured by [`GeckoRuntimeSettings.Builder#remoteDebuggingEnabled`](../javadoc/mozilla-central/org/mozilla/geckoview/GeckoRuntimeSettings.Builder.html#remoteDebuggingEnabled-boolean-)

Not all actions and features have GeckoView API interfaces.  Generally, actions and features that do not have GeckoView API interfaces are not intended for broad usage.  Configuration for these types of things is controlled by:
- environment variables in GeckoView's runtime environment
- command line arguments to the Gecko process
- internal Gecko preferences

Automation-specific configuration is generally in this category.

## Reading configuration from a file

When GeckoView is embedded into a debugabble application (i.e., when your manifest includes `android:debuggable="true"`), by default GeckoView reads configuration from a file named `/data/local/tmp/$PACKAGE-geckoview-config.yaml`.  For example, if your Android package name is `com.yourcompany.yourapp`, GeckoView will read configuration from
```
/data/local/tmp/com.yourcompany.yourapp-geckoview-config.yaml
```

### Configuration file format

The configuration file format is [YAML](https://yaml.org).  The following keys are recognized:
- `env` is a map from string environment variable name to string value to set in GeckoView's runtime environment
- `args` is a list of string command line arguments to pass to the Gecko process
- `prefs` is a map from string Gecko preference name to boolean, string, or integer value to pass to set in the Gecko profile

```yaml
# Contents of /data/local/tmp/com.yourcompany.yourapp-geckoview-config.yaml

env:
  MOZ_LOG: nsHttp:5

args:
  - --marionette
  - --profile
  - "/path/to/gecko-profile"

prefs:
  foo.bar.boolean: true
  foo.bar.string: "string"
  foo.bar.int: 500
```

### Verifying configuration from a file

When configuration from a file is read, GeckoView logs to `adb logcat`, like:

```
           GeckoRuntime  I  Adding debug configuration from: /data/local/tmp/org.mozilla.geckoview_example-geckoview-config.yaml
       GeckoDebugConfig  D  Adding environment variables from debug config: {MOZ_LOG=nsHttp:5}
       GeckoDebugConfig  D  Adding arguments from debug config: [--marionette]
       GeckoDebugConfig  D  Adding prefs from debug config: {foo.bar.baz=true}
```

When a confiugration file is found but cannot be parsed, an error is logged and the file is ignored entirely.  When a configuration file is not found, nothing is logged.

### Controlling configuration from a file

#### Enabling reading configuration from a file for release builds

By default, GeckoView provides a secure web rendering engine.  Custom configuration can compromise security in many ways: by storing sensitive data in insecure locations on the device, by trusting websites with incorrect security configurations, or by not validating HTTP Public Key Pinning configurations.

**You should only allow such configuration if your end-user opts-in to the configuration!**

But some applications (for example, web browsers) may want to allow configuration for automation even when the application is not debuggable (i.e., for release builds that have `android:debuggable="false"`).  In such cases, you can use [`GeckoRuntimeSettings.Builder#configFilePath`](../javadoc/mozilla-central/org/mozilla/geckoview/GeckoRuntimeSettings.Builder.html#configFilePath-java.lang.String-) to force GeckoView to read configuration from the given file path, like:

```java
new GeckoRuntimeSettings.Builder()
    .configFilePath("/your/app/specific/location")
    .build();
```

#### Disabling reading configuration from a file entirely

To force GeckoView to never read configuration from a file, even when the embedding application is debuggable, invoke [`GeckoRuntimeSettings.Builder#configFilePath`](../javadoc/mozilla-central/org/mozilla/geckoview/GeckoRuntimeSettings.Builder.html#configFilePath-java.lang.String-)` with an empty path, like:

```java
new GeckoRuntimeSettings.Builder()
    .configFilePath("")
    .build();
```

The empty path is recognized and no file I/O is performed.
