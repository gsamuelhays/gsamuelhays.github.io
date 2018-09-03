---
layout: post
title: Controlling STS with PingFederate by Application
date: 2018-09-01 8:30 PM
category: blog 
tags: blog pingfederate ws-trust sts office365 azure
---
### What you'll get from this post
If you're using PingFederate and Office 365, this post will show you how to selectively authenticate users who are hitting the STS endpoint based on the application they are requesting and arbitrary attributes available in Active Directory.  In other words you could make decisions like:

1. Only let users in the 'STS-Powershell' security group connect to the Microsoft Online powershell endpoint
2. Only let users in the 'STS-IMAP' group connect via IMAP to collect email

etc. 

I will not cover the Azure/Office 365 configuration here. Instead I assume you have this configured and are relatively familiar with those items. Our starting place is a fully-configured PingFederate environment that is using WS-Federation or SAML and WS-Trust. 

### Overview of the problem

It is not my experience that many companies or service providers are using WS-Trust/STS these days except for Microsoft. When using Office 365 and Azure, there can be significant STS usage across various services including ActiveSync, SMTP, IMAP, MAPI, and several more. I do not have any kind of fundamental problem with their protocol selection - but there are security implications which should be considered. To that end - The problem I have has to do with the lack of options I have for step-up authentication when these protocols are employed (due to the nature of the protocol, not something Microsoft or PingIdentity has done wrong).

When using SAML or WS-Federation for passive-client authentication, it is relatively straightforward to inject a step-up in the auth path. However, when when a user is connecting via STS (as opposed to Modern-Auth), it is very difficult to enforce that second factor. Additionally, I have seen an uptick with phishing campaigns such that if and when a user's credentians are compromised, I see STS connections almost immediately across a range of applications. Why? Because without a second factor - compromised creds are all the attacker might need.

Now - this blog post is not how to inject 2-factor. As I said, that is a hard problem and with Microsoft encouraging folks to head towards modern auth, I'm not sure its worth the time to try and solve. Instead, I've been working with my teammates to figure out a strong transition plan to accomplish the following:

1. Move all users to modern auth without disabling globally 
2. Limit STS for specific applications based on Active Directory group membership
3. Increase the logging such that it is easy to see the activity in a meaningful way

### Item 1) Moving users to Modern Auth

The game plan is to re-configure all of the iDevices and Androids to use Modern Auth. The problem is that even if a user gets a new iPhone, they usually restore their configuration which includes their mail configuration.  Most users (again, in my experience) have had their mail configured and saved to the cloud for a long time and consequently - its configured using traditional ActiveSync over STS. How do we easily control this based on attributes stored in our Active Directory?


### Item 2) Limit specific applications/users from STS via Active Directory?

The next problem is that there are some integrations with 3rd party vendors that need access to endpoints such as EWS or ActiveSync or IMAP and those integrations only support STS. This puts us in a position that we cannot disable the protocols entirely but rather need to make decisions as to whether or not a user will be authorized based on things like userPrincipalName or group membership. At the end of the day, the ideal would be all human targets cannot use STS for authentication but some limited number of service accounts can.

### Item 3) Increased logging

The company I work for uses Splunk as its SIEM. I want to snag STS connections and record the username (userPrincipalName), the application that is being authenticated, the true IP address (for analysis of impossible-travel), and some other fields to be calculated at processing time. These data should go into Splunk for easy review and potentially alerting.

## Data Collection

The first thing we need to do is collect some data about what STS connections might be being hit. You can set this up by following [my post](http://sams.site/blog/2018/05/27/Limit-Powershell-from-O365-Powershell.html) and only using the 7 lines of code (yes, this post is similar to that one - except I realized how to do some newer, cooler stuff).

Once we have that logger enabled, we can run a query in splunk like the following:

```
index=federation STS-Logger
| stats count by X_MS_Client_Application
```

This will give you results like the following (note I ran this over 90 days in my environment).

![Apps hitting STS]({{ site.uri }}/assets/2018_09_01-STS_Apps.png)


With this in place, we now have data regarding usage of our STS endpoint and can start understanding the who, what, where, when, and why's of those connections perhaps put some controls in place. Perhaps PowerShell-LiveID should be limited to a subset of users. Perhaps only from specific IP ranges. You might be thinking "Hey, things like PowerShell support modern auth!" and that's true. Maybe STS can be disabled in the cloud - but at the time of this writing, my colleagues are not aware of an easy way to do that. Plus, you know, its nice to have options when solutioning.


### Active Directory Set-Up

My team decided we wanted to create Active Directory groups and populate them with users so that easy analysis can be done and simply removing a user from a group would cut off their STS access. This means that once they're out of the group, when Microsoft sends the token request to your STS, we simply deny the request and the user cannot authenticate - effectively removing their ability to use the service. The use cases are numerous but our first goal is to roll-back classic STS auth for ActiveSync and enforce (gently) modern auth throughout the company. Secondary things like finer control to IMAP and PowerShell are just icing on the cake.

First thing to do is create some groups that will be used to control access.  We've created the following groups and those will be used in the OGNL code later in this article.

![AD Groups for STS Controll]({{ site.uri }}/assets/2018_09_01-STS-AD-Groups.png)

(NOTE: You will need to get the full Distinguished Name when we use them in PingFederate)

<span style='color: red'>**NOTE: PRIOR TO IMPLEMENTING ANY CODE IN PINGFEDERATE, MAKE SURE THE GROUPS ARE POPULATED WITH USERS BECAUSE PING WILL DENY USERS NOT IN THE GROUP ONCE THE CODE IS IN. YOU HAVE BEEN WARNED.**</span>


### PingFederate Code-Prep

I spent a goodly amount of time thinking about whether I wanted one big OGNL script to handle all of the logic or a cascading set of smaller scripts for each of the protocols. On the one hand, one script would likely be less resource intensive since I would only instantiate objects one time for processing. On the other hand, OGNL is an abomination of a 'language' straight from the pits of hell and long code is hard to debug, explain to others, and maintain. So I decided to go with smaller chunks of code. This will assume you want to do likewise. 

So - let's look at the code:

{% highlight text linenos %}
#log = @org.apache.commons.logging.LogFactory@getLog("com.pingidentity.ognl.logger"), 
#objReq = #this.get("context.HttpRequest").getObjectValue(),
#msapp = #objReq.getHeader("X-MS-Client-Application"),
#group = "CN=SEC-STSAllow-PowerShell,OU=SEC-GROUPS,DC=DOMAIN,DC=COM",
#result = msapp == "Microsoft.Exchange.Powershell" ? (
    #xff = #objReq.getHeader("X-MS-Forwarded-Client-IP") != null ? #objReq.getHeader("X-MS-Forwarded-Client-IP") : "NONE_PROVIDED", 
    #memberof = #this.get("ds.AD.memberOf") == null ? "" : #this.get("ds.AD.memberOf").toString(),
    #upn = #this.get("ds.AD.userPrincipalName"),
    #match = #memberof.indexOf(#group),
    #match = #match >= 0 ? true : false,
    #log.debug("[STS-Logger] msapp = " + #msapp + ", upn = " + #upn + ", group = " + group + ", member = " + #match + ", X-MS-Forwarded-Client-IP = " + #xff),
    #match
  
) : true
{% endhighlight %}


There is a bit we'll need to do in PingFederate before we can implement this code, but let's go ahead and explain it and you can get your own code ready, if you're playing at home. ;-)

Line numbers and their explanations below:
1. Instantiate an object so we can log to the PingFederate Server log (remember, my group has that logging enabled and parsed by Splunk).
2. Obtain the HttpRequest object (which contains all the headers passed to us by Microsoft) so we can get the Application trying to use the STS as well as IP address information.
3. Get the X-MS-Client-Application header that Microsoft passes in and assign it to a variable called `msapp`.
4. Create a variable to hold our group name. This part could be skipped and you could just do the code in-line, but I like it for my documentation. Either way - make sure this is the full distinguished name of the group corresponding to the app to be controlled.
5. This one is a little weird. We're using the ternary operator to see if the app is PowerShell... if it is, we continue to the indented block. If it isn't we assign the value true (line 
14) - which is what will tell PingFederate this this is OK to Issue a response to.
6. We check to see if the X-MS-Forwarded-Client-IP is present. If it is - we assign it to the `xff` variable otherwise we assign 'NONE_PROVIDED'.
7. We configure PingFederate (shown later) to lookup group membership during the token processor phase of the STS flow. This line assigns the results of that LDAP query to the variable `memberof`. This will be used to determine if a user is a member of a group and if they should be allowed to authenticate.
8. Just like line 7, we are extracting userPrincipalName from LDAP and making it available here in the form of the variable `upn`.
9. We check to see if our `group` distinguised name exists in the string that represents all the groups to which the user under evaluation is a member.
10. If we did find a match on line 9, then we assign the value of `true` to the variable `match`, otherwise `false`.
11. Print some useful debug information in the server log.  We should be picking this up in something like Splunk or some other SIEM.
12. Here we echo the `match` variable that is true if the user was a member, and false otherwise. Since this is the last line in the block, this value is assigned to the `result` variable on line 5. If this variable is `false`, PingFederate will deny the STS request.

(There will be a test.  Kidding. kidding.)



### PingFederate Server Configuration

Obviously everybody's PingFederate configuration can be pretty different- but if this isn't exactly the same, it should be close enough to get you going. Now, as I mentioned in my previous article - some of these configuration items are _deep_ in the system. PingFed is great for SSO and I am a big fan, but one of the three problems I have with that company is the user interface for PingFed.  So, let's get started.

First, we have to navigate to the STS section of the Office 365 SP connection. 

![Navigating to STS 1]({{ site.uri }}/assets/2018_05_27-PF-PS-OGNL-1.png)

![Navigating to STS 2]({{ site.uri }}/assets/2018_05_27-PF-PS-OGNL-2.png)

![Navigating to STS 3]({{ site.uri }}/assets/2018_05_27-PF-PS-OGNL-3.png)

![Navigating to STS 4]({{ site.uri }}/assets/2018_05_27-PF-PS-OGNL-4.png)

![Navigating to STS 5]({{ site.uri }}/assets/2018_05_27-PF-PS-OGNL-5.png)

Now that we're in the IdP Token Processor Mapping, we need to quickly have a look at the Attribute Sources & User Lookup. This is because the OGNL we'll be putting it requires some additional attributes to be pulled from LDAP and made available to us.

![LDAP Attributes]({{ site.uri }}/assets/2018_09_01-LDAP-attrs.png)

Click the Data Store tab and take note of your `Attribute Source Id`. This is used in the OGNL expressions above. Those lines that read like: `#this.get("ds.AD.userPrincipalName")` aren't magic. That 'AD' is referencing this ID and so your code will need to also match.

![Attribute Source Id]({{ site.uri }}/assets/2018_09_01-Attr-Source-Id.png)

Now we'll bounce over to the 'LDAP Directory Search' section. We'll want to add `memberOf` and `userPrincipalName` as attributes that are pulled. _**When you've added these, click 'DONE' for the next set of changes. If you click 'Save' the attributes you added will likely be removed because they are not part of an attribute contract. This is bad behavior on PingFederate's part (in my opinion). So - DONE, not SAVE.**_

![New Attributes in LDAP]({{ site.uri }}/assets/2018_09_01-New-Attrs.png)

Once you've hit Done (and NOT 'SAVE'), you should be back at the IdP Token Processor Mapping section of the configuration.  Now we want to navigate to the Issuance Criteria where we'll add our prepared code. Also, once we're at this screen we'll want to click "Show Advanced Criteria".

![Issuance Criteria]({{ site.uri }}/assets/2018_09_01-Issuance-Criteria.png)


With the 'Advanced Criteria' show - you'll get some new fields that don't tell you very much:

![Advanced Criteria]({{ site.uri }}/assets/2018_09_01-Advanced-Criteria.png)

These are the fields where we'll plug in our prepared OGNL. For me, I have a new expression for each of the protocols I want to control via AD Group Membership. As I mentioned earlier - this might add a _little_ overhead to the system, but I think the manageability is much stronger. So- you know, I do what I want. :)

Lets look at a screenshot from my configuration. This has TWO OGNL expressions shown (of the 13 or so actually implemented).  I have underlined the section where my `Attribute Source Id` was used in code (see above). 

![Production OGNL]({{ site.uri }}/assets/2018_09_01-Production-OGNL.png)

Once this is done - PingFederate will start checking to see if a user is a member of a specific group during STS authentication. It will drop the connection if they are not a member. In Splunk, we can easily see the value added here:

 
![Splunk Data]({{ site.uri }}/assets/2018_09_01-Splunk-STS.png)

What is plainly visible here is we can see who is connecting, from which IP address, the group we are checking and whether or not they are a member.  We could search for situations where people are blocked by running a search like this:

```
index=federation STS-Logger msapp=* member=false
| table msapp, upn, group, member, X_MS_Forwarded_Client_IP
```

### Final Thoughts

So, with this all in place, you should have a handy set of new tools to help protect your environment. If you find that that isn't true, you'll at least have greater visibility into what is happening, especially since by default STS information is not logged. 

This has been very handy for my team to start putting in stronger controls and best practices. If you have questions, feel free to reach out at gsamuelhays at google’s email service dot com.

Good luck and stay safe!

Sam
