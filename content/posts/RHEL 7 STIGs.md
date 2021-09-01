---
title: "RHEL 7 STIGs"
date: 2021-08-31T23:38:19-04:00
draft: true
toc: false
images:
tags:
  - linux
  - ansible
  - git
---

## Overview
In order to get familiar with what general linux hardening looks like, as well as practicing Ansible, I decided to put together a playbook that can be periodically ran against machines to verify continued STIG compliency. 

## Basics

I ran through the list of [STIG's](https://stigviewer.com/stig/red_hat_enterprise_linux_7/) one by one and automated them to the best of my ability. I've attempted to implement verification checks as well to ensure that if this script is run again in the future, it will be able to provide active proof of compliency.


