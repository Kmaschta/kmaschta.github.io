---
layout: post
author: Kévin Maschtaler
title: "Site Reliability Engineering: Google's Secret Sauce For High Availability And Happy Ops"
excerpt: "Developers have a lot to learn from the book \"Site Reliability Engineering - How Google Runs Production Systems\", starting with time management and blameless retrospectives."
date: 2018-05-24 20:00:00
image: /blog/content/sre-large.png
canonical: https://marmelab.com/blog/2018/05/24/site-reliability-engineering.html
tags:
- devops
- book-review
- performance
---

> *Hope is not a strategy*<br />
> — Traditional SRE saying

I've read the book *Site Reliability Engineering - How Google Runs Production Systems*. I learned a lot, and I took away many good practices to apply to our own services. Here is the gist, and what I've learned from it. I'll focus on what web developers can learn from this SRE thing, without entering in the complexity of the Google's infrastructure.

One quick disclaimer before diving in: at Marmelab, we don't run our customers' services in production. Our expertise is eliminating uncertainties through agile iterations, so we usually delegate hosting to a partner company. That said, our job can't be disconnected from the production. We are responsible for the quality of the delivered software, and that includes making software that reports quality of service problems, and an architecture that make it resilient and performant. So even though it's not our job per se, the operation of web services interests me a lot.

[![Site Reliability Engineering - How Google Runs Production Systems](/blog/content/sre-book.png)](https://landing.google.com/sre/book.html)

## What Is Site Reliability Engineering (SRE)?

First and foremost, let's define this emerging profession invented by Ben Treynor Sloss, Senior VP of Engineering at Google. From his own words:

> Fundamentally, SRE is what happens when you ask software engineers to design an operations function.

A typical SRE team is formed of 6 to 8 engineers, in order to keep a balanced on-call rotation. Half of it is composed of traditional software engineers (SWE in Google's parlance), and the other half of engineers who are almost qualified to be SWE, but who have skills and interests related to operational fields: Unix internals, networking, and so on.

They occupy a central position in the company's workflow. They are in relation with software engineers, release managers, data center engineers, product owners, accounting service, and upper management.

They are fully **responsible for the availability, latency, performance, efficiency, charge management, monitoring, emergency response, and capacity planning of a given application**.

In order to face these huge responsibilities and hard scaling challenges, they carefully manage their time. They do it in an unusual way: **they must devote less than 50% of their time to operational tasks, toil, and emergency response**. Most of their time should be spent writing software and tools that automate their own job, or making sure that their application heals itself.


**DevOps vs. SRE**

DevOps is a movement initiated in 2008-2009 by [Patrick Debois](https://twitter.com/patrickdebois) to promote agility in the deployment process, and to reduce the gap between developers and ops.

SRE is rather an engineering field, a way to organize engineers in order to manage reliability as a whole. As explained [page 7 of the SRE book](https://landing.google.com/sre/book/chapters/introduction.html#devops-or-sre-8OS8HmcX):

> *One could equivalently view SRE as a specific implementation of DevOps with some idiosyncratic extensions.*

![Balancing Is Not That Easy](https://media.giphy.com/media/zjF9aoAIrrjCE/giphy.gif)

## Balancing Availability and Velocity

I expected this book to teach me Google's secret path to the "Always Available System™". I was wrong, but I learned a far more valuable lesson. Google engineers do not focus on 100% availability, because it's unrealistic and unsustainable.

Instead, according to the business requirements and expectations, each application or production system gets a Service Level Objective (SLO). For a typical Google app, the SLO is set around 99.99% ("four nines" availability), or 99.999% ("five nines" availability) of successful requests rate over all requests.

Here is [how the number of nines translates to downtime per year](https://en.wikipedia.org/wiki/High_availability#Percentage_calculation):

* 99% ("two nines"): 3.65 days
* 99.9% ("three nines"): 8.77 hours
* 99.99% ("four nines"): 52.60 minutes
* 99.999% ("five nines"): 5.26 minutes

At marmelab, most of our projects have a 99.9% uptime (three nines, not that bad!), even though we focus far more on velocity than availability. One important takeway from the SRE book is that, if we ever wanted to reach the next service level grade (four nines), we'd have to increase our efforts on availability **ten times**.

The difference between 100% and "x nines" availability" is important because it provides a measurable room for maneuver. For example, if the unavailability is measured in terms of unplanned downtime and the SLO is set to 99.9%, over a year it is considered acceptable to have about eight hours of downtime.

This relatively thin margin of acceptable and unplanned downtime is a way to measure the risk induced by the velocity. If the availability decreases, and gets closer to the SLO, the SRE team should slow down the feature delivery, and focus on stabilizing the app. On the contrary, if the availability is far above the SLO, then the team has some margin, and can go ahead and push new features to prod.

Of course, the Service Level Objective can change over time. It should be discussed among all the stakeholders of the project in regard to all the circumstances. It is a valuable metric to have.

![Automate ALL The Things](https://cdn-images-1.medium.com/max/1600/1*TKt92huSBbSnbRNuAVTx_A.jpeg)

## Eliminating Toil

Since SRE shouldn't devote more than 50% of their time to operational tasks, they regularly have to automate their work as their application scales. They should focus on reducing **toil**, defined as:

> a manual, repetitive and automatable task that is mandatory to the proper functioning of the app and that grows with it.

Automating toil can be a non-negligible investment, but it comes with several advantages. Of course, it saves engineer's time. But it also leads engineers to focus on a task at the right time (not when the service is broken), in a less error-prone way. Moreover, it reduces context switching, allows everyone to take care of serious issues and focus on what really matters.

As developers, we also have to take care of toil every day - it just takes another form. **Manual tests** are the most common toil, that's why we write automated tests. The second most common toil is **deployment and release management** in general. Continuous Delivery is a luxury that I definitely recommend! Other toils are more subtle: **updating dependencies** that don't have breaking changes, **benchmarking performance**, **auditing security vulnerabilities**, etc.

**Eliminating toil increases the ability to scale and makes the application nicer to play with.** It's up to you to find it and automate it! The best way is to include this routine as a habit on a daily basis, just like any other development practice.

![Shame nun of Game of Thrones](https://pbs.twimg.com/media/CVkAep3UkAAXUXT.jpg:large)

## Blameless Culture of Retrospective

SRE has a culture of continuous improvement (what agile methodologies translate as the *retrospective*) in the form of **postmortems**. Every time an issue or an accident is big enough or is user-facing, the on-call engineer writes a postmortem, another engineer reviews it and publishes it to the whole team.

Postmortems are extremely useful to keep track of incidents, plan further actions, and avoid repeating mistakes. But SRE teams go beyond that: they write postmortems **blamelessly**.

Each postmortem focuses on facts and root causes analysis. It assumes that everyone had good intentions, and did the right actions with the pieces of information they had. All finger pointing and shaming are explicitly banned. Moreover, the management doesn't punish mistakes. Instead, they take postmortems as an opportunity to improve the robustness of their systems.

It's a hard thing to do, but removing blame and feels from retrospectives (and in this case postmortems) gives the confidence to escalate the real issues. Moreover, most system failures are due to a human mistake.

It was refreshing to read that from a book by Google engineers. It's a good example of how agile principles can be put in practice, beyond the development process.

## Conclusion

This book is full of insights and I wrote about only a very small portion of them. Among other technical tips like "divide and conquer" or "simplicity by design", it also teaches how Google scales human interactions among its employees.

I warmly recommend *Site Reliability Engineering* to anyone who is interested in production scaling and DevOps, of course, especially the *Practices* part, which is full of practical stuff. 

The whole book is a must for all engineering managers. The fourth part, *Management*, gives valuable insights on how to deal with interrupts, handle or recover from operational overload, and so on.

But I also recommend reading the *Principles* part to every product owner and agile guru, to get a better understanding of how ops teams work and collaborate.

The book is modular and easily accessible. You can read just a chapter of it without having to understand how the Google infrastructure works.

The SRE book is [free to read online](https://landing.google.com/sre/book.html), or you can find it on [Amazon](http://amzn.eu/8N8FRhV)!

Site Reliability Engineering is a new and very interesting field, and you might expect to read more about it on this very blog.

Further reading about SRE:

- [So you want to be an SRE?](https://hackernoon.com/so-you-want-to-be-an-sre-34e832357a8c) by Krishelle Hardson-Hurley
- [Google SRE resources](https://landing.google.com/sre/resources.html)

Image credit: [Google's Finland Data Center](https://www.utilityclick.com/google-energy/)
