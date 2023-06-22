---
layout: post
title: "The Analysis of RedLine Stealer"
date: 2023-06-20
tags: .Net Malware 
description: The post shows the details of the analysis of the RedLine Stealer that acts as a GTA cheating module. 
---

# Intro

- In this post, I will cover the details of my analysis for info-stealer malware called **RedLine stealer** which is currently one of the trends. Regarding the case covered in this post, the malware masquerades as a cheating module for the GTA game.


# The kill-chain

- The following diagram shows the flow of attack from the initial access to communication with the C2 server.

   ![img]({{ '/assets/images/Redline/redline-1.png' | relative_url }}){: .center-image }*(**The kill-chain Diagram**)*

## Delivery stage

