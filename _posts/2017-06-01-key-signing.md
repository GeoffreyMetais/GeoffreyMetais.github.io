---
title: "Signing android releases with yubikey"
excerpt: "Use a security USB Key to embed your Play Store Keystore and sign your releases on the go"
categories:
 - code
tags:
 - vlc
 - android
---

At VideoLAN, we recently changed our signing procedure to leverage our security keys.
As explained on [Yubico website](https://developers.yubico.com/PIV/Guides/Android_code_signing.html), Android signing is quite easy.

{% include toc icon="gears" %}

With this method, I can now sign the VLC releases on any of my computers without duplicating the keystore file. And keystore password is replaced by my Yubikey PIN.

Here is the precise process we went through to get this done.

# Setup

### Install dependencies
I am considering a Debian based distribution with open JDK 8 installed for this post.
{: .notice--info}

First of all, we need to install the pkcs11 opensc lib and the zipalign tool
```
sudo apt-get install opensc-pkcs11 zipalign
```
`zipalign` can also be found in *android_sdk_path*/build-tools/*version*/
{: .notice--info}

### Prepare configuration file  
Then, we prepare the pkcs11 configuration. Let's create file `pkcs11_java.cfg` and fill it with:
```
name = OpenSC-PKCS11
description = SunPKCS11 via OpenSC
library = /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so
slotListIndex = 0
```
Yubico How-To advises `slotListIndex = 1`, but I had to set it to 0 to make it work with my Yubikey 4.
{: .notice--info}

Let's assume we save it in: `~/.pkcs11_java.cfg`

### Set up your own management key
If you did not set your management key, you have to do it now:
```
key=`dd if=/dev/random bs=1 count=24 2>/dev/null | hexdump -v -e '/1 "%02X"'`
echo $key
yubico-piv-tool -a set-mgm-key -n $key
```
### Change your PIN
Same for PIN setting, default one is *123456*
```
yubico-piv-tool -a change-pin -P 123456 -N <NEW PIN>
```
# Keystore import

Now it's time to import our keystore to the key PIV slot.
```
keytool -importkeystore -srckeystore mykeystore.keystore -destkeystore mykeystorey.p12 -srcstoretype jks -deststoretype pkcs12
yubico-piv-tool -s 9a -a import-key -a import-cert -i mykeystorey.p12 -K PKCS12 -k
```
 You will be asked to type in the keystore password, then the certificate management key.

 Starting from now, you won't have to type the keystore password anymore but your Yubikey PIN.
{: .notice--info}

 We can check that our key is ready to sign apps:
```
keytool -providerClass sun.security.pkcs11.SunPKCS11 -providerArg ~/.pkcs11_java.cfg -keystore NONE -storetype PKCS11 -list -J-Djava.security.debug=sunpkcs11
```
This is the Yubikey PIN you have to type-in now.  
And don't forget to touch it if you enabled the 'touch-to-sign' option.
{: .notice--warning}

# App signature

We now have to get an **unsigned** apk, so we must tell gradle to not apply any signing config for release builds
```
 buildTypes {
        release {
            signingConfig null
            //…
        }
    }
```

Finally we can sign an apk without our keystore, we just need the Yubikey to be plugged and fire up *jarsigner*
```
jarsigner -providerClass sun.security.pkcs11.SunPKCS11 -providerArg ~/.pkcs11_java.cfg \
  -keystore NONE -storetype PKCS11 -sigalg SHA1withRSA -digestalg SHA1 \
  app.apk "Certificate for PIV Authentication"
```

We can now verify the package is signed:
```
jarsigner -verify app.apk
```

The `-sigalg SHA1withRSA -digestalg SHA1` parameters are needed because we support old devices. If you don't support Android 4.2 and older you can rip it off.
{: .notice--info}


# Scripting

Here is the full bash script I use to sign all my apks at once:
``` bash
#! /bin/sh

echo "Please enter Yubikey PIN code "
stty -echo
trap 'stty echo' EXIT
read -p 'PIN: ' YUBI_PIN
stty echo
trap - EXIT

echo "\nSigning apks\n"
for i in `ls *.apk`;
do
jarsigner -providerClass sun.security.pkcs11.SunPKCS11 \
  -providerArg ~/.pkcs11_java.cfg -keystore NONE -storetype PKCS11 \
  -sigalg SHA1withRSA -digestalg SHA1 -storepass $YUBI_PIN \
  $i "Certificate for PIV Authentication"
zipalign 4 $i $i.tmp && mv -vf $i.tmp $i
done
unset YUBI_PIN
```

# Apksigner

Since build tools 24.0.3, Google released [apksigner](https://developer.android.com/studio/command-line/apksigner.html), a newer signature tool with convenient arguments like `--min-sdk-version` to get sure the application signature is correct.
But current release (25.0.3) doesn't handle pkcs11 protocol correctly.

builds tools 26.0.0 have just been released and they do not contain apksigner anymore
{: .notice--info}

We still can build apksigner from source  to get upstream version which is able to manage our Yubikey!

```
git clone https://android.googlesource.com/platform/tools/apksig
```
We need Bazel to build this project, [here are some instructions to install it](https://bazel.build/versions/master/docs/install.html)
Then go into apksig folder, create a Bazel workspace and let's build it.

```
touch WORKSPACE
export WORKSPACE=`pwd`
bazel build :apksigner
```

And here is how we can sign with it:
```
~/tmp/apksig/bazel-bin/apksigner sign --ks NONE --ks-pass "pass:$YUBI_PIN" \
--min-sdk-version 9 --provider-class sun.security.pkcs11.SunPKCS11 \
--provider-arg pkcs11_java.cfg --ks-type PKCS11 app.apk
```
With apksigner, we need to zipalign the apk **before** signing them.
{: .notice--warning}

We can also verify apk is well signed:
```
~/tmp/apksig/bazel-bin/apksigner verify app.apk
```
And `verify` accepts `--min-sdk-version` and `--max-sdk-version` to ensure your users won't get `103` once the release is out.