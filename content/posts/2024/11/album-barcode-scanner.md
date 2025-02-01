---
title: From Vinyl to Chromecast with Home Assistant And ESPHome
subtitle: ""
date: 2024-11-02
slug: 1c219a1
draft: false
author: Andrew
description:
keywords:
comments: true
weight: 0
tags:

- esphome
- homeassistant
- homeautomation
categories:
- open-source
- python
- esphome
- homeautomation
summary:
hideMeta: false
hideSummary: false
showtoc: false
tocopen: false
ShowPostNavLinks: false
ShowBreadCrumbs: false
ShowWordCount: false
hideFooter: false
---

I used ESPHome and Home Assistant and an [open-source Python API](https://github.com/andrewthetechie/home-barcode-api) to build a way to scan my [vinyl albums' barcodes](https://www.discogs.com/user/andrewthetechie/collection) and have them play using spotify over my
home's Chromecasts.

<!--more-->

## ESPHome Hardware

This project started with an idea: I wonder if I can make esphome read barcodes. Turns out, the answer is an emphatic yes. There [are](https://www.pcbway.com/project/shareproject/Barcode_scanner_made_for_ESPHome_bb9bba9a.html) a few [projects](https://community.home-assistant.io/t/gm67-bar-code-reader-module/394182) out there already, but they had one drawback - the [UART interface barcode scanners](https://www.amazon.com/Coolwell-Recognition-Supports-Accuracy-Scanning/dp/B0C3M3H7DV) they were using meant that you had to bring the item up to the scanner to scan. I wanted a handheld wireless scanner.

There are lots of handheld barcode scanners for sale, I bought an [inexpensive one from Aliexpress](https://www.aliexpress.us/item/3256806029682141.html?spm=a2g0o.order_list.order_list_main.137.77b11802IR5u9Q&gatewayAdapt=glo2usa) for around $15. It operates as a [USB HID Device](https://en.wikipedia.org/wiki/USB_human_interface_device_class) - a USB Keyboard. When its USB dongle is plugged into a computer, scanning a barcode will type it into a text field. This particular scanner has a USB recharable battery and a speaker/led to indicate scanning success on the handheld unit.

![Wireless Barcode Scanner](/images/2024/11/album-barcode-scanner/barcode-scanner.png)

My first attempt to make that readable by an ESP was using a [CH9350 HID module](https://www.aliexpress.us/item/3256806406901461.html?spm=a2g0o.order_list.order_list_main.142.77b11802IR5u9Q&gatewayAdapt=glo2usa). This module takes a USB device in and then outputs HID codes over UART. I was able to get this working with an ESP32, but ran into difficulty trying to convert the HID codes into ASCII characters.  [Jann](https://github.com/joetrs) created an [example library](https://github.com/joetrs/ESP32_CH9350_KEY) that I tried to adapt, but I wasn't able to get it working inside of ESPHome. The CH9350 has some interesting capabilities, I hope I can get it working with ESPHome in the future and share a component for others to use.

Thankfully for this project, Hobbytronics had me covered with their [USB Host Controller Board](https://www.hobbytronics.co.uk/product/host-board). For £18, I had to write no code at all. Their pre-loaded firmware did exactly what I want - output the ASCII characters that my HID barcode scanner was "typing". I connected this to an ESP32's UART pins and stuck both of them in a small project box with plenty of hot glue to hold everything in place.

![Completed ESP Barcode Scanner](/images/2024/11/album-barcode-scanner/completed-esp-scanner.png)

The ESPHome device config is intentionally bare bones. A text sensor is setup using a [component](https://github.com/ssieb/esphome_components) that listens for a string coming in via serial. After 10s, the text sensor is reset to "blank" to allow retriggering of the same album. The rest of the config is just ESPHome boilerplate.

{{< yaml_collapsible file="files/2024/11/album-barcode-scanner/esphome.yaml" expand_link_text="Expand ESPHome Device Config" download_link_text="Download ESPHome Config" >}}

## Barcode Lookup API

With a way to scan barcodes, I needed a way to look them up and get a Spotify ID. I use [Discogs](https://www.discogs.com/) to organize my vinyl collection, and it offers an [API](https://www.discogs.com/developers). Combining that with the [Spotify API](https://developer.spotify.com/documentation/web-api/reference/search), I had
access to the data I needed.

After researching [options](https://hacs-pyscript.readthedocs.io/en/latest/) to run [python scripts](https://www.home-assistant.io/integrations/python_script/) in Home Assistant, I decided it would be worth the extra effort to run this code as an external service and then query it in Home Assistant with a [RESTful Command](https://www.home-assistant.io/integrations/rest_command/). I have some other plans for the API in the future and it gave me a good excuse to play with some FastAPI skills.

[![andrewthetechie/home-barcode-api - GitHub](https://gh-card.dev/repos/andrewthetechie/home-barcode-api.svg)](https://github.com/andrewthetechie/home-barcode-api)

The Discogs and Spotify APIs have fair limits for APIs, but to try to limit the amount of outbound calls, the Barcode API uses a DB as a cache. I deployed the API into my homelab k8s cluster and mounted a 500mb PVC to the pod for SQLite. If you'd like to
deploy it for yourself, there is a [Docker image](https://github.com/andrewthetechie/home-barcode-api/pkgs/container/home-barcode-api) and it takes its config via env vars. Maybe I'll even write a proper README someday

I had to do some very basic data cleaning because the Discogs API returns odd results for artists. For example, Tool is returned as "Tool (2)", which the Spotify API doesn't like. I suspect there will be more bugs as I use this system with more than the 20 albums I grabbed out of the my collection.

## Home Assistant

After adding my barcode scanner to Home Assistant via the ESPHome integration, I needed an automation to trigger when a barcode was scanned.

{{< yaml_collapsible file="files/2024/11/album-barcode-scanner/homeassistant.yaml" expand_link_text="Expand HomeAssistant Automation" download_link_text="Download HA Automation" >}}

The automation looks for the text sensor's state to change and then submits that to the API via a RESTful command. If the API returns a 200, it uses TTS to speak the artist and album name and then starts playing using Spotcast. If the automation gets a 404, it uses TTS to read back the error message (which usually results in a cheerful robot reading out a 12 digit number UPC-A.)

## Parts list and final costs

* [ESP32 WROOM-32 Development Board](https://www.aliexpress.us/item/3256806290562326.html?spm=a2g0o.order_list.order_list_main.11.1db5180221ceGR&gatewayAdapt=glo2usa) ~$4
* [Hobbytronics USB Host Controller Board v2.4](https://www.hobbytronics.co.uk/product/host-board) ~$23
* [USB Wireless Barcode Scanner](https://www.aliexpress.us/item/3256806029682141.html?spm=a2g0o.order_list.order_list_main.137.77b11802IR5u9Q&gatewayAdapt=glo2usa) - $15
* Project Box, USB C cable, hookup wires, hot glue, solder ~$4

Total ~$46

## What's Next?

This project was a proof of concept to reliably reading barcodes with ESPHome. In the future, I want to setup a scanner in my kitchen for use with a grocery ERP to track our groceries and household goods.

The music player is really a toy. It'll be a neat thing to show off to visitors that gets used occasionally. My total cost wasn't much for a fun demo and for learning more about ESPhome, Barcodes, HID devices, and Home assistant. I'll eventually replace the turntable.
