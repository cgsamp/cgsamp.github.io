---
title: "TCP Adventures"
date: 2024-5-29
---


# TCP Routing

I have been working with a customer that is seeing odd behavior with an app that uses tcp routing on Tanzu Cloud Foundry. This isn't very common, but some older or specialized apps have to use it. This one is both.

Since the app was recently deployed on an upgraded CF and started exhibiting odd symptoms, I got involved. It has been an interesting challenge.

## Problem

The application would see requests every ~5 seconds that consisted of a string of null bytes. Not a null string, but a string of about ten bytes with the value 0x00 (Hex zero). Sometimes it would be in logs as "\n0000" repeated.

This was a special challenge because the application sometimes receives null bytes as a recoverable error, so has logic to find them and replace them, usually with spaces (0x20).

## Solving

I was shared a test class that worked locally. I deployed it to CF. There was some configuration to get through. I stumbled on updating my load balancer settings. Details can be found here:

https://github.com/cgsamp/tcp-server-test
