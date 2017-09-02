---
layout: post
title: Takeaways from SRECon17 Europe
categories:
- blog
---

As every last three years in a row, I attended SRECon in Europe. I can literally say this year was totally broken comparing with former conferences. I think it's because I had much higher expectations from this conference. The first shot in 2014 was more than awesome, but year to year it's getting worse. Almost all talks from Google were like a summary of every chapter in [SRE book](https://landing.google.com/sre/book.html). We just skipped all the rest of the talks sourced by Google.

In general, the conference was nice and I still would recommend it for everyone even not for SREs.

To summarize what I found useful from this year:
* Managing SSH Access without SSH keys - this is the way like Facebook, Google, Uber are doing as well. But the best part was that they [opensourced](https://github.com/nsheridan/cashier) a software to authenticate users via Google OAuth before getting a certificate. They told they are issuing certificate for one week (while Google said they use 20hours);
* Openresty show, as expected, the talk was from Shopify. They integrated Openresty along with Zookeeper for auto-discovery, but dunno how they manage it (no real answer was given in my question). Another one interesting idea was they are using ECMP with some kind of custom throttling. Too fewer details to imagine what they are doing here, but it's quite interesting. Hence I had a couple more questions:

1. How are you dealing with ECMP members when throttling nodes?
2. If you take care of members when throttling then how do you drain the traffic clearly?
3. And how about resilient hashing within ECMP group?
4. What is the size of ECMP group?
5. etc..

* Capturing and Analyzing Millions of Queries without Any Overhead. This talk was presented by LinkedIn. They proved that slow query incurs ~30% overhead, performance schema incurs ~15%, what about near 0%? They created an agent which listens on 3306, sniffs the MySQL traffic using `libpcap`, parses the query, does anonymization and returns back to the original backend. Parsed data is sent to the centralized server to aggregate data. Overhead is ~3% if threads count is only over 128. Glad, that they are planning to opensource this soon;
* OK Log: Distributed and Coördination-Free Logging - One of the best talk in this conference this year. There wasn't anything fancy, but it was submitted and presented very well. The main point is that they implemented custom ingestors to absorb logs and eventually store them in Prometheus;
* [git-stacktrace](https://github.com/pinterest/git-stacktrace)
* BitTorrent protocol - LinkedIn is using this protocol for deploys, I heard someone else is using this as well in this conference.

A couple of few fun moments during this conference:

![Quake3-during-the-flight](/images/q3-during-the-flight.jpg "Deathmatching Quake3 during the flight")
![Hostinger-SRECon17-Europe](/images/hostinger-srecon17.jpg "Hostinger attending SRECon17 Europe")
![EuroBasket2017](/images/srecon17-eurobasket.jpg "Shopify vs. EuroBasket 2017")
![Twelve nines uptime](/images/srecon17-12nines.jpg "Twelve nines uptime")

#### Conclusion

Hope is not a strategy!
