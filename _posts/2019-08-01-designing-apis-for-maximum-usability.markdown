---
layout: post
title:  "Designing APIs for maximum usability"
description: "If API transactions are not your application's main communication, it's development tend to be Ad-hoc. What should be your concerns when designing an API that everyone will want to use?"
date:   2019-08-01 12:21:00 -0300
categories: Concepts
permalink: /:categories/:title
---
Design patterns and best practices aren't just a cosmetic or secondary concern when building APIs. Even if your main product doesn't revolve around your API you should develop it in a way people will not misuse or abuse it.

Let's get something straight: this article is about exploring concepts that are meant to **help** you make better APIs. There is no kind of bulletproof method to make a great application, but failing to apply concepts contained here may lead to:

 - Unpredictible user consumption;
 - Performance issues that are hard to isolate;
 - Impossible to debug situations;
 - Difficulties auditing or getting metrics that are meaningful.

## REST and GraphQL are there for a reason

Most of those issues shouldn't exist if you adopt widely known API design patterns such as [REST](https://medium.com/hashmapinc/rest-good-practices-for-api-design-881439796dc9) or [GraphQL](https://graphql.org/learn/best-practices/). Using them will make people know what to **expect** from your service while also making documentation will be easier to read and write.

Keep in mind that people's actions are usually predictable. Actions you didn't anticipate should only happen if:

 - The documentation doesn't cover all funcionalities;
 - Your API fails to deliver an acceptable interface with your application;
 - Exeptions returns errors for the user, but doesn't interrupt code execution;
 - Your responses are not clear about what happenned.

## SOLID design should permeate your whole application

That can't be stressed enough:

 > If you're building object oriented software, SOLID books should be your bible. Failing to apply SOLID design patterns on your application leads to software that is hard to maintain and/or scale.

On API calls your main focus should be on the "S" of the SOLID acronym. Make endpoints that have one and ONLY one responsability. It's better to have your users making calls to different endpoints to reach an objective than having a "does it all" endpoint. Numerous fast interactions are better than a big and slow one.

Sometimes your application may experience performance bottlenecks. In those cases, those big API calls will generate timeouts and frustration. If you need to create an object then run some processing on it, make sure your users understand your application's workflow. Reducing users' interactions should not be your main concern.

## API is a product, think about your ROI

Besides having a comprehensible documentation and adopting known design patterns, APIs should have certain features to perform better and be easily consumable, such as:

 - Pagination, search and filtering on index actions;
 - A way to replicate actions an user could make on your UI;
 - Enforce quotas and throttling.

Pagination, search and filtering will make your users consume your resources more efficiently. Avoiding giant payloads means fewer expenses with your application servers.

Replicating UI actions on your API will mean that you won't have users making crazy feature requests. If you plan just enough to satisfy your users' needs, your application should have a much cheaper maintenance cost.

Enforcing quotas and throttling will allow you to:

 1. Monetize your API through quotas. Users that use large scale applications to consume your resources can cause your servers bills to go through the roof. It doesn't mean you'll always charge your consumers, but it's certainly a possibility;
 2. Practise restraint over how many resources you will dedicate to your API. Throttling avoids abuses and protects your application from a possible DoS attack, even unintentional ones *(yes, it happens)*.

Having an usable API modernizes your relationship with your clients. While the concepts listed here are a great starting pointing to guide API development, your team should always try to be as agile as possible when bringing your product live. The amount of energy and time spent to build a mediocre product is the same you'd spend building a good one. Cutting corners can lead to extra effort in the future.

Do you have some experiences about API design you'd like to share? Let's discuss it in the comments below! I'd be happy to contribute or to know more about what you're doing :)
