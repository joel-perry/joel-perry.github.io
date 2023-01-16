---
layout: post
title:  ApolloCombine
date:   2022-09-01 18:02:40 -0600
tags:   [combine apollo graphql]
---
ApolloCombine is a collection of Combine publishers for the [Apollo iOS client](https://github.com/apollographql/apollo-ios). The collection is available via Swift Package Manager and CocoaPods on [GitHub](https://github.com/joel-perry/ApolloCombine). The repo contains usage instructions.

In late 2019, I made the decision to use the Combine framework in one of our production apps. I started by incrementally replacing some of the flow-control code in the UI, then began to look for other places to use Combine. The one place I really wanted to attack was the network layer, which uses the Apollo iOS client to (ultimately) communicate with our REST API. This package is the result of that work.

Of the five main operations in the Apollo client, I currently use three extensively in production: `fetch`, `perform`, and `upload`. The other two operations (`watch` and `subscribe`) have been tested, but have not received the same attention.