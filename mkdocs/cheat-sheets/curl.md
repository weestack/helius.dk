---
date: 2025-01-25
draft: false
tags:
  - curl
title: Curl cheatsheets
icon: simple/curl
---
# Curl

Curl can be used for a variety of tasks and is my go-to tool for testing web protocols, verifying mail credentials—heck, even testing FTP!

This cheat sheet contains some of the most common ways I use Curl.

## Acknowledgments
First of all, thank you to [Everything Curl](https://everything.curl.dev) for the great content and explanations—far better than what I’ll include in my cheat sheet!

## Checking redirects
Unlike your browser, Curl does not cache requests. 
This makes it a great tool for testing 301 or 302 
redirects in real time—allowing you to verify changes on the fly.

```bash
curl -I https://example.com
```


## Sending mails
As a senior DevOps engineer, I often receive messages like, 
"The SMTP login you provided isn't working [...]". Over the years, 
I’ve found that testing with Curl is incredibly easy, 
so I’ve made it a habit to send an email to the developer as a proof of concept. 
I then share the exact command I used and help them debug their code to identify the issue.

Create a mail file
`mail.txt`
```txt
From: Alexander Hoøgh <Alexander@helius.dk>
To: who ever <their@mail.com>
Subject: Proof of concept
Date: Fri, 1 Jan 2025 08:00:00

Dear developer, so sorry to hear about your issue,
lets look it over together and debug what went wrong ^_^

Best regards Alexander,
Have a fantastic day
```

`Command line:`
```bash
smtp://Username:Password@provider:port --mail-from alexander@provider.IDN --mail-rcpt their@mail.com --upload-file mail.txt
# Note that if your username or password contains @ or other characters then a uri ended string is supported on the protocol level
# Lets say the username is alexander@helius.dk and password is password123
smtp://alexander%40helius.dk:password123@myprovider:587 --mail-from alexander@helius.dk --mail-rcpt their@mail.com --upload-file mail.txt
```

