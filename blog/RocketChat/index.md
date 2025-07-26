---
title: Debian RocketChat Deployment
date: 2024-2-23
description: This guide will help deploy RocketChat on Debian with Docker
categories: [Tutorials]
draft: false # Change to true to not render the post in on the website
---

In order to deploy RocketChat on Debian with Docker you need to first update your machine
<pre>'''sudo apt-get update'''</pre>

Then install the Docker application
<pre>'''sudo apt-get install docker'''</pre>

Afterwards install the required packages for docker
<pre>'''sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release'''</pre>

Next add Docker's official GPG Key
<pre>'''curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg'''</pre>

