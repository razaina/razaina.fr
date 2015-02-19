---
layout: post
title: "Using OpenSSL as a static library within an Android native project"
description: ""
permalink: "/:title"
category: 
tags: []
---
{% include JB/setup %}

## Install OpenSSL

First off, I strictly followed the steps described here >> <a
href="http://wiki.openssl.org/index.php/Android#OpenSSL_Library" target="_blank">OpenSSL's
Wiki</a> << with the latest version of OpenSSL at this time: <a
href="https://www.openssl.org/source/openssl-1.0.2.tar.gz" target="_blank">1.0.2</a>.

Before to configure, compile, install, you need to properly set the cross-compilation
environment. The "setenv-android.sh" script is provided within the link above. 

Below are the different variables/piece of code I changed/added to fit my needs: 

{% highlight bash %}
_ANDROID_NDK="android-ndk-r10d"
_ANDROID_EABI="arm-linux-androideabi-4.9"
_ANDROID_ARCH=arch-arm
_ANDROID_API="android-19"
...
export ANDROID_SDK_ROOT="/home/razaina/android-sdks"
export ANDROID_NDK_ROOT="/home/razaina/android-ndk-r10d"
...
# In case the following loop fails while running the script
for tool in $ANDROID_TOOLS
do
    ...
done

#Just replace it like so
for tool in arm-linux-androideabi-gcc arm-linux-androideabi-ranlib arm-linux-androideabi-ld
do
    ...
done
{% endhighlight %}

Once OpenSSL compiled, it will be installed in the following directory:
/usr/local/ssl/android-19. 

## Integrate OpenSSL within your Android native project

You just need to properly configure your Android make file as follows:

{% highlight bash %}
LOCAL_PATH := $(call my-dir)

# Prepare the SSL static library
include $(CLEAR_VARS)
LOCAL_MODULE := libssl-prebuilt
LOCAL_SRC_FILES := /usr/local/ssl/android-19/lib/libssl.a 
LOCAL_EXPORT_C_INCLUDES := /usr/local/ssl/android-19/include
include $(PREBUILT_STATIC_LIBRARY)

# Prepare the CRYPTO static library
include $(CLEAR_VARS)
LOCAL_MODULE := libcrypto-prebuilt
LOCAL_SRC_FILES := /usr/local/ssl/android-19/lib/libcrypto.a
LOCAL_EXPORT_C_INCLUDES := /usr/local/ssl/android-19/include
include $(PREBUILT_STATIC_LIBRARY)

# Compile your executable
include $(CLEAR_VARS)
LOCAL_MODULE    := program 
LOCAL_SRC_FILES := program.c 
LOCAL_ARM_MODE := arm
LOCAL_CFLAGS := -g
LOCAL_C_INCLUDES := /usr/local/ssl/android-19/include
LOCAL_LDLIBS := -L/usr/local/ssl/android-19/lib -llog
LOCAL_STATIC_LIBRARIES := libssl-prebuilt libcrypto-prebuilt
include $(BUILD_EXECUTABLE)
{% endhighlight %}

<div id="disqus_thread"></div>
<script type="text/javascript">
/* * * CONFIGURATION VARIABLES: EDIT BEFORE PASTING INTO YOUR WEBPAGE * * */
var disqus_shortname = 'razaina'; // required: replace example with your forum shortname

/* * * DON'T EDIT BELOW THIS LINE * * */
(function() {
var dsq = document.createElement('script'); dsq.type = 'text/javascript'; dsq.async = true;
dsq.src = '//' + disqus_shortname + '.disqus.com/embed.js';
(document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
