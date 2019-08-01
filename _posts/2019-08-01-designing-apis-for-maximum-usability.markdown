---
layout: post
title:  "Designing APIs for maximum usability"
description: "If API transactions are not your application's main communication, it's development tend to be Ad-hoc. What should be your concerns when designing an API that everyone will want to use?"
date:   2019-08-01 12:21:00 -0300
categories: Concepts
permalink: /:categories/:title
---
Design patterns and best practices aren't just a cosmetic or secondary concern when building APIs. Even if your main product doesn't revolve around your API you should develop it in a way people will not misuse or abuse it.

Let's get somethings straight, this article is about exploring concepts that are meant to **help** you make better APIs. There is no kind of bulletproof method to make a great application, but failing to apply concepts contained here may lead to:

 - Unpredictible user consumption;
 - Performance issues that are hard to isolate;
 - Impossible to debug situations;
 - Difficulties auditing or getting metrics that are meaningful.

## REST and GraphQL are there for a reason

Most of those issues shouldn't exist if you adopt widely known API design patterns such as [REST](https://medium.com/hashmapinc/rest-good-practices-for-api-design-881439796dc9) or [GraphQL](https://graphql.org/learn/best-practices/). Using them will make people know what to **expect** from your service and, in turn, your documentation will be easier to read and write.

Keep in mind that users actions are predictable most of the time. They will only try to do stuff that you didn't anticipate if:

 - Documentation doesn't cover all funcionalities;
 - Your API fails to deliver an acceptable interface with your application;
 - Exeptions returns errors for the user, but doesn't interrupt code execution;
 - Your responses are not clear about what actually happenned.

## SOLID design should permeate your whole application

That can't be stressed enough:

 > If you're building object oriented software, SOLID books should be your bible. Failing to apply SOLID design patterns on your application leads to software that is hard to maintain and/or scale.

On API calls your main focus should be on the "S" of the SOLID acronym. Make endpoints that have one and ONLY one responsability. It's better to have your users making calls to different endpoints to reach an objective than having a "does it all" endpoint. Numerous fast interactions are better than a big and slow one.

Sometimes your application may experience performance bottlenecks and, when that happens, those big API calls will generate timeouts and frustration. If you need to create an object then run some processing on it, make sure your users understand your application's workflow. Reducing users' interactions should not be your main concern.

## API is a product, think about your ROI

Besides having a comprehensible documentation and adopt a know design, APIs should implement some features to perform better and be easily consumable. such as:

 - Pagination, search and filtering on index actions;
 - Have a way to replicate actions an user could make on your UI;
 - Enforce quotas and throttling.

Pagination, search and filtering will make your users consume your resources more efficiently - avoiding giant payloads means less expenses with your application servers.

Replicating UI actions on your API will mean that you'll have less users making crazy feature requests. If you plan just enough to satisfy their needs your application will have way less maintenance costs.

Enforcing quotas and throttling will enable your API to reach maximum ROI through:

 1. API Monetization through quotas. Users that use large scale applications to consume your resources can cause your servers bills to go above the roof. It doesn't mean you'll always charge your consumers, but it is a possibility;
 2. Maintain a restraint over how much resources you will dedicate to your API. Throttling avoids abuses and protects your application from a possible DoS attack, even unintentional ones *(yes, it happens)*.

Having an usable API modernizes your relationship with your clients. While the concepts listed here are a great starting pointing to guide your API development, the team will always try to be as agile as possible when bringing your product live. The amount of energy and time spent to build a mediocre product is equivalent to building a good one, so remember that cutting corners can lead to extra efforts in the future.

You got nice experiences to share about API design? Discuss it within the comments below, I'd be happy to contribute or to know more about what you're doing :)
