---
title: Debian RocketChat Deployment
date: 2024-2-23
description: This guide will help deploy RocketChat on Debian with Docker
categories: [Tutorials]
draft: false # Change to true to not render the post in on the website
---

<h3>**What's RocketChat?**</h3>

Rocket.Chat is a chat server you can run on your own machine or server that allows:

Real-time messaging (channels, private DMs)

File sharing

Video/audio calls (with add-ons)

Notifications

Admin control

Integration with other tools

Similar to Slack, Microsoft Teams and Discord

<h3>**Why should you use RocketChat?**</h3>

You should consider using Rocket.Chat if you want secure, customizable, team-based communication software that you fully control — without the vendor lock-in or data privacy concerns of tools like Slack, Discord, or Microsoft Teams.

For example it has:
1. Full Data Ownership and Privacy
2. Highly Customizable
3. Integration with your Tools
4. Multi-Platform Access
5. No Vendor Lock-in or Monthly Fees
6. Great for Learning and IT Practice


In order to deploy RocketChat on Debian with Docker you need to first update your machine
<pre>sudo apt-get update</pre>

Then install the Docker application
<pre>sudo apt-get install docker</pre>

Afterwards install the required packages for docker
<pre>sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release</pre>

Next add Docker's official GPG Key
<pre>curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg</pre>

You will need to setup a stable Docker Repository by running this command
<pre>echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \ https://download.docker.com/linux/debian $(lsb_release -cs) stable" | \ sudo tee /etc/apt/sources.list.d/docker.list > /dev/null</pre>

Then Update and install Docker:
<pre>sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
</pre>

Start and Enable the Docker Service:
<pre>sudo systemctl start docker
sudo systemctl enable docker
</pre>

Now that you have Docker installed and setup, you need Deploy RocketChat

First you are going to need to create a directory for RocketChat
<pre>mkdir rocketchat-deploy && cd rocketchat-deploy</pre>

Next create a YAML file
<pre>vim docker-compose.yml</pre>

Afterwards paste the following content into the YAML file:
<pre>
version: '3'

services:
  mongo:
    image: mongo:5.0
    container_name: mongo
    restart: unless-stopped
    command: mongod --replSet rs0 --oplogSize 128
    volumes:
      - ./data/db:/data/db

  mongo-init-replica:
    image: mongo:5.0
    container_name: mongo-init-replica
    depends_on:
      - mongo
    entrypoint: >
      bash -c "sleep 5 && 
      mongo --host mongo:27017 --eval '
        rs.initiate({_id: \"rs0\", members: [{_id: 0, host: \"mongo:27017\"}]})'"
    restart: "no"

  rocketchat:
    image: rocketchat/rocket.chat:latest
    container_name: rocketchat
    restart: unless-stopped
    depends_on:
      - mongo
      - mongo-init-replica
    environment:
      - MONGO_URL=mongodb://mongo:27017/rocketchat?replicaSet=rs0
      - MONGO_OPLOG_URL=mongodb://mongo:27017/local?replicaSet=rs0
      - ROOT_URL=http://localhost:3000
      - PORT=3000
    ports:
      - 3000:3000

</pre>

Now Install Docker Compose
<pre>sudo apt update
sudo apt install docker-compose-plugin -y
</pre>

Now Start the containers
<pre>sudo docker compose up -d</pre>

Wait a couple of minutes for RocketChat to Initialize

And there you go, that's how you deploy RocketChat using Docker on a Debian based machine.









