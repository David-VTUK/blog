---
title: "Intel Skylake/Kaby Lake processors: broken hyper-threading"
date: 2017-06-26
slug: "intel-skylakekaby-lake-processors-broken-hyper-threading"
categories: 
  - "storage"
  - "virtualisation"
---

# Overview

Source : [https://lists.debian.org/debian-devel/2017/06/msg00308.html](https://lists.debian.org/debian-devel/2017/06/msg00308.html)

It appears some Intel Xeon CPU's are susceptible to a recently discovered Hyper Threading bug. However, these are limited to E3 v5/v6 based Xeon systems which are found mostly in entry level servers with **single socket** implementations. > Dual socket systems currently leverage E5 based Xeons which don't appear to be affected.

Currently, the easiest way to mitigate against this bug is to simply disable hyper-threading. The bug also appears to be OS agnostic.

# **Just Servers?**

The focus around social media has predominately been around run of the mill servers; ones you typically purchase from the likes of Dell, HP, etc. However, there could be many bespoke devices that leverage susceptible processors, such as NAS/SAN heads. It is unlikely that in the event you find such a device HT can simply be disabled, but it should be something to be aware of.

[List of Intel processors code-named "Skylake"](http://ark.intel.com/products/codename/37572/Skylake) [List of Intel processors code-named "Kaby Lake"](http://ark.intel.com/products/codename/82879/Kaby-Lake)
