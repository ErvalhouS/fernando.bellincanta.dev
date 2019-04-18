---
layout: post
title:  "Why distributing systems affects your whole organization?"
description: "Distributed systems is an architectural paradigm that makes any application more reliable, scalable and maintainable. The key to design a distributed system is to isolate behaviors." 
date:   2019-04-18 10:07:00 -0300
categories: Concepts
permalink: /:categories/:title
---
Take the **frontend**/**backend** separation for instance, one of the most basic service isolation there is. Developing systems like this enables your organization to have 2 specialized teams on each part of the application, making it easier to hire your personal by **limiting the skillset** necessary to be a part of either team.

But having two or more separate applications means that they will have to communicate somehow. This is where your teams have to make an effort to adopt as much **best practices** as possible. They should understand what to expect from each other through wireframes, mindmaps, test cases and documentation.

This should be done as much as possible, even when the project have time constraints, what usually leads to cutting the fat here. Neglecting documentation makes it harder for newcomers in your team to understand how your project works, or even how they should do their chores.

Application separation of this kind is seen more often on mobile development, since you have to implement API consumption on the clients' application. It becomes always inevitable not to have a backend application to feed the mobile app.

But, when talking about web development, one of the most used strategy is **clustering**, to maintain communication with your frontend as geographically close as possible. Clustering means multiple threads running your application, a machine with a bunch of virtual servers in it, or physical servers on the same location. All those strategies leads to an extra complexity, but it yields **superior performance** on each service as a return.

Clustering can be complicated if you are not used to deal with distributed applications, but it is way more controllable than having each of your clients installing an applications on mobile phones.

A well defined clustering strategy can even be replicable easily using a deployment tool like [Kubernetes](https://kubernetes.io/), [Ansible](https://www.ansible.com/) and/or [Docker](https://www.docker.com/), making horizontal scalability an easier thing to achieve.

The isolation of the applications can also lead to an **independent business relation**, allowing you to adopt them as separated products. Selling products with reduced scope is easier, so you can even make extra **profit** out of your clustering strategy.

Reducing the scope of applications can also enable your team to **open source** some services that are not tightly coupled to your business competitive advantage. In the end you will get better quality software that is maintainable, scalable, sellable and commutable, affecting your whole company in the process.

Do you have any suggestions or experiences on how to implement an effective clustering strategy? Leave your comment below!
