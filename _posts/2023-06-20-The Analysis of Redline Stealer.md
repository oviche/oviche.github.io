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
  - The malware is distributed as a cheating module for the GTA game on some websites and delivered when it gets downloaded by gamers.

  - The below screenshot shows a text file that exists in the same folder with the infection first stage executable, trying to give the user confidence that it's a non-malicious cheating module by showing instructions and features.

  ![img]({{ '/assets/images/Redline/redline-2.png' | relative_url }}){: .center-image }*(**Snippet for the instructions text file**)*
