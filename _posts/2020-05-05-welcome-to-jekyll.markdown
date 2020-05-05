---
layout: post
title:  "Arogya Setu: An app that gets everything wrong"
date:   2020-05-05 15:01:04 +0530
categories: [analysis, vulnerability, security, aarogya-setu]
permalink: /aarogya-setu/
---

These are my observations from reverse engineering Aarogya Setu's APK. It must be highlughted though, that a) I am not a trained or experienced hacker, but a developer who looked at it purely from a security-conscious developer's perspective. A trained hacker will most definitely be able to point out several other vulnerabilities that I must have missed.  

# Permissions

Let’s start with the list of permissions this app requires., because generally, that gives a pretty good sense of what the app could possibly have access to on a user’s phone.

  1. **Bluetooth + Bluetooth Admin** – these are understandable because it claims to use BT to identify people. Still, access to BT Admin is a little disconcerting because that means the app can switch on your BT without your manual intervention. A more responsible use would have been to just check and alert the user if their BT was off and leave it upto them to switch it on.
  2. **Location / GPS** – this is also not a shocker. The app does claim to access your GPS data. What’s problematic, however, is the fact that in addition to COARSE_LOCATION and FINE_LOCATION, it also has access to something called BACKGROUND_LOCATION, which means it can get access to your device’s GPS data at all times, even when the app is not running.
  3. **Network State** – this lets the app know details about the network your phone is connected to.
  4. **Wake Lock** – with this, the app can keep your phone is active state (prevent it from going into sleep mode) for as long as it requires.
  5. **Receive Boot Completed** – the app gets a message everytime your phone switches on (finishes booting). Typically this is useful for apps such as email, messenger clients etc because such apps have a heavy reliance on background processing. Without it, everytime you restarted your phone, your whatsapp or gmail apps (for example) won’t get new emails/messages till you tapped on them at least once. Why does an app like AS need this, I don’t know.
  6. **SMS Retriever** – this is not a declared permission because the new android APIs don’t require you to declare this in manifest explicitly, but the app also has a SMSRetriever routine that lets it grab SMS messages from the user’s phone. This is a very common use-case with a lot of apps that rely in SMS OTPs, but the way this particular app implements it is problematic. More details below.

# Observations

Next, let’s look a couple of problems in the app code itself.

A major problem from a security perspective is that the app stores user tokens in shared preferences in their raw form. On Android, shared preferences (as the name suggests) are shared amongst all apps installed on a phone. What that means is that it will only take one malicious app (out of what? Around 100 that most people have on their phones) to access a user’s token. It would have been different if the token was encryped, or was not stored in a location exclusively available to the app. A related problem is that the same lax attitude can be seen when handling api keys and authorization header values.

![Screenshot 1](/assets/img/as_1.png)

***
<br/>

![Screenshot 2](/assets/img/as_2.png)

> **Note:** the keys have been hidden in the screenshots just because one doesn’t know where this document will end up going. The app code, however, made no such considerations.  
  
***
<br/>

The SMS retriever routine looks shady as hell. No matter what the regex pattern match returned, I did not see a block that discarded the SMS (as in, did not send it for further processing).

![Screenshot 3](/assets/img/as_3.png)

> SMS Retriever... multiple if blocks... segregated conditions, but the message goes ahead in the end, no matter what.

***
<br/>

Another major problem is the way it scans for nearby bluetooth devices. There is no check for whether the other device has this app installed or not, if there is a device nearby that has it’s bluetooth turned on (which is not a rare scenario given how many of us use bluetooth headsets, speakers, fitness trackers etc), it will grab that device’s information and store in the database, along with its mac address.

![Screenshot 4](/assets/img/as_4.png)

> The data that goes into the local database. There is another issue with db security as well. More about that below. 

***
<br/>

The local database’s identity hash is also hard coded into the source code in plain view. Again, all it will take is one malicious app to capture all that information they are collecting about the user’s and every other phone in their vicinity to grab the data right from the source itself. This could have been properly encrypted at the very least.

![Screenshot 5](/assets/img/as_5.png)
> Here. I have hidden the actual hash value. The app does not.

***
<br/>

Next, it’s been designed to keep tabs on, and collect as much information as possible, at ALL TIMES. They have a background service, a foreground service, and a job scheduler (worker) all running the same code that collects information about the user’s GPS coordinates and bluetooth details of every phone in their vicinity (whether the other phone has this app or not). This is not only excessive surveillance, but a bad coding practise as well. Why would you not show any consideration towards the user’s phone’s data or battery life? This doesn’t stop. No matter what.

***
<br/>

Yet another problem is the fact that there is decompiled bit code in there that looks at every other service running on your phone. Why would you need to look at such a sweeping search result? Why not just look at a specific service, like the sdk expects you to?

![Screenshot 6](/assets/img/as_6.png)

***
<br/>

The next major problem is how the app goes through your contacts list. Given how a contact lookup returns not just the number, but other, sensitive details (like how many times have you contacted them, which group have you added them to, their photo, display pic etc), teh app’s sweeping lookup parameters are problematic. 

![Screenshot 7](/assets/img/as_7.png)

***
<br/>

Last, but not the least, even the storage mechanism outside the app is not very secure, but that was expected given how easily the authorization headers and api keys were accessible in the app itself. This whole exercise putting a lot of unconsented for data of users and non-users at risk.

***
<br/>

```
Important Notes/Disclaimers:

1. I am not a hacker, so I might have missed out on other things that would  
seem more important to a trained hacker. A couple other things like the fact  
that the app can not just read, but send messages, as well as trigger  
calls looked iffy to me, but I wasn’t sure if that is in response to a user  
action or not, so have let that be for the scope of this document.

2. There are much more serious problems with this app, not just from a  
technology, but social, human rights, and other viewpoints as well.  
The issues highlighted here are purely from a tech perspective,  
mostly because that’s the only perspective that I could make sense of.
```