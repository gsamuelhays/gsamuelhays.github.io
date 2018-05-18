---
layout: post
title: AzureAD PCV with PingFederate
date: 2018-05-13 2:00 PM
category: blog 
tags: blog pingfederate azure aad 
---

Recently I've run into some use-cases to integrate PingFederate with Azure Active Directory (AAD) as a Credential Validator. Obviously this shouldn't be too hard because PingIdentity has a Password Credential Validator for that purpose[1]. But, sadly, the documentation was completely lacking[2]. Now, I don't think this is completely PingIdentity's fault as half of the integration is with 3rd party (Microsoft). But still. Have a look at the following screenshot describing how to configure the Azure AD side for yourself:

![PingIdentity's Instructions]({{ site.uri }}/assets/2018_05_17_PF_AAD_CONFIG.png)

Let's walk through getting this configured to see what the steps *actually* are as of May 17th, 2018. I will even stipulate that deploying the jar PCV is a given and you've already done that. Okay? So we're talking about the rest of the PingFederate's configuration as well as the Azure side.

We're going to start with configuring the AAD side so we can obtain the appropriate keys that will be necessary. These will be `Domain`, `Application ID`, and `Application Key`.

This is the right place for me to mention, right quick, that I am not an AAD wizard (but I'm pretty okay with AWS ;-)). So, efficiencies may be spotted or gained by working with someone else or ignoring this blog completely. Caveat lector.

Alright - the PingFederate doc makes no mention[3] of the need to create an App in Azure to handle this stuff. Perhaps this is supposed to be obvious - but it wasn't to me. So - you might think the following is the right place to go to accomplishing this:

![Apps in Azure]({{ site.uri }}/assets/2018_05_17_PF_AAD_CONFIG_02.png)

As you can see - the _obvious_ choice is the wrong choice. Instead, you want to follow that Microsoft link[4] highlighted in green. This page is where Azure Active Directory v2.0 apps are built (I guess[5]. Eh.) So, once we get to the v2 page, we can 'Add an app' that will help us get to where we're going!

![My applications screen]({{ site.uri }}/assets/2018_05_17_PF_AAD_CONFIG_03.png)

Let's give our application a name. I'll call it BetterDocumentationPlease. 

![BetterDocumentationPlease app]({{ site.uri }}/assets/2018_05_17_PF_AAD_CONFIG_04.png)

At this point we'll be greeted with a number of options. Let's run through the things we need to do here. 

1) Record the Application ID for later use<br>
2) Generate a new password and record that too

![AppID and Secret]({{ site.uri }}/assets/2018_05_17_PF_AAD_CONFIG_05.png)

The Platforms option is interesting - we tested with with with Web API and _nothing_ defined and both worked, but not defining anything caused an error (that seemed to do nothing) in a later step. So I am recommending leaving it blank.

Let's head down a little farther to the 'Microsoft Graph Permissions' section. This is one part of the configuration that was supremely frustrating about Ping documentation. It literally says (at the time of this writing):

>"Microsoft Graph > Delegated Permission<br>
>Sign in and read user profile<br>
>Read directory data<br>
>Windows Azure Active Directory > Delegated Permission<br>
>Sign in and read user profile"

That's it. Thats all it says. But guess what! Those permissions do not exist in the list... go ahead, check it out. Ridiculous, no? Well - I found a page[6] that does the translation for us. The gist is this: 

User.Read = Enable sign-in and read user profile<br>
Directory.Read.All = Read directory data

The following screenshot shows working and correct permissions:

![AAD Permissions]({{ site.uri }}/assets/2018_05_17_PF_AAD_CONFIG_06.png)

Here is what is interesting... in my testing, this was not enough to make things work on the Azure side. We had to administratively approve the app so there was not a necessary pop-up window during login. How does one do that? I would have bet quite a bit that there would be a UI element, but for the life of me I could not find one and a colleague ended up finding some documentation on how to construct the URL[7]. 

Long story short, it looks like this:<br>
`https://login.microsoftonline.com/common/adminconsent?client_id=<APPLICATION_ID>&state=12345&prompt=admin_consent`

Once the administrator has done this - it should be smooth sailing.

![Administrative Consent]({{ site.uri }}/assets/2018_05_17_PF_AAD_CONFIG_07.png)


Now - you might get an error after this. For example I got "AADSTS50011: No reply address is registered for the application." which is true. I didn't register a reply address. Shouldn't be a problem.

Now we're ready to configure the PingFederate side.

## PingFederate configuration

Note that I am using PingFederate 9.0.2

So we hit up 'Server Configuration' -> 'Password Credential Validators' and start the process of configuring our PCV.

![PCV Conf]({{ site.uri }}/assets/2018_05_17_PF_AAD_CONFIG_08.png)

On the next page I will fill out the Domain (which hopefully you know) and the Application ID and the Application Key (password) I told you to jot down earlier.

![PCV Settings]({{ site.uri }}/assets/2018_05_17_PF_AAD_CONFIG_09.png)

On the following page, you can extend the contract, but I won't do that here. After another "Next" or two in PingFederate, click "Done" and then "Save". 

At this point I should say - I strongly recommend using Policies, Policy Contracts, and Selectors in your PingFederate configuration but will not be building that out here for the sake of my time. You should be able to capture my meaning from the short-cut configuration I am doing from here on out.

Create an _Adapter_ that will be used for testing.

![PF Adapters]({{ site.uri }}/assets/2018_05_17_PF_AAD_CONFIG_10.png)

This part is mostly standard fair for an adapter creation - except you'll use the PCV you created earlier and you may want to extend the contract to include attributes available from the PCV. 

![PCV in adapter]({{ site.uri }}/assets/2018_05_17_PF_AAD_CONFIG_11.png)


Lastly, for testing purposes, you'll want to create a test connection that uses this adapter and PCV combination. I'll skip this part since I assume readers are PingFederate administrators - but once thats done you should see the following kinds of items in your log files (provided sufficient logging is enabled).

```
2018-05-17 11:52:49,312 tid:A2OCGujwxrlXnvwHkpkr-iWe4fo DEBUG [com.pingidentity.adapters.pcv.azure.AzurePasswordCredentialValidator] Searching for attribute mail in the returned attributes.
2018-05-17 11:52:49,312 tid:A2OCGujwxrlXnvwHkpkr-iWe4fo DEBUG [com.pingidentity.adapters.pcv.azure.AzurePasswordCredentialValidator] Found attribute mail in the returned attributes.
2018-05-17 11:52:49,312 tid:A2OCGujwxrlXnvwHkpkr-iWe4fo DEBUG [com.pingidentity.adapters.pcv.azure.AzurePasswordCredentialValidator] Searching for attribute displayName in the returned attributes.
2018-05-17 11:52:49,312 tid:A2OCGujwxrlXnvwHkpkr-iWe4fo DEBUG [com.pingidentity.adapters.pcv.azure.AzurePasswordCredentialValidator] Found attribute displayName in the returned attributes.
2018-05-17 11:52:49,312 tid:A2OCGujwxrlXnvwHkpkr-iWe4fo DEBUG [com.pingidentity.adapters.pcv.azure.AzurePasswordCredentialValidator] Searching for attribute surname in the returned attributes.
2018-05-17 11:52:49,312 tid:A2OCGujwxrlXnvwHkpkr-iWe4fo DEBUG [com.pingidentity.adapters.pcv.azure.AzurePasswordCredentialValidator] Searching for attribute givenName in the returned attributes.

[...] etc.
```

Ultimately it will build our a SAML assertion and send it on its way to whatever destination. 

So, in summary, getting the Azure AD connection was a huge pain but the adapter does work. My coworker Jason worked with me on this work and deserves much of the credit - particularly on the AAD side. 

If this has helped you out - I'd love the feedback at gsamuelhays at google's email service dot com.

Sam


### References:
[1] https://www.pingidentity.com/en/resources/downloads.html -> Azure AD PCV 1.1<br>
[2] https://docs.pingidentity.com/bundle/AzureADPCV11_sm_azureActiveDirectoryPCV/page/AzureADPCV_c_PCV.html<br>
[3] https://docs.pingidentity.com/bundle/AzureADPCV11_sm_azureActiveDirectoryPCV/page/AzureADPCV_c_configuration.html<br>
[4] https://go.microsoft.com/fwlink/?linkid=832645&clcid=0x9 -> (redirected me to https://apps.dev.microsoft.com/?referrer=https:%2f%2fazure.microsoft.com%2fdocumentation%2farticles#/appList)<br>
[5] https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-appmodel-v2-overview<br>
[6] https://msdn.microsoft.com/en-us/library/azure/ad/graph/howto/azure-ad-graph-api-permission-scopes<br>
[7] https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-v2-scopes<br>

