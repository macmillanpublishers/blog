---
layout: post
title: Building My First Slackbot to Query Channel History
---

Here's the full code:

```javascript
var SlackClient = require('slack-client');

var autoReconnect = true; // Automatically reconnect after an error response from Slack.

var token = "YOUR TOKEN HERE";

var team = "YOUR TEAM HERE";
 
var slackClient = new SlackClient(token, autoReconnect);
 
var bot; // Track bot user .. for detecting messages by yourself

var XMLHttpRequest = require("xmlhttprequest").XMLHttpRequest;

var getChannelHistory = function() {
    this.get = function(family, value, callback) {
    var xhr = new XMLHttpRequest();
    var url = "https://" + team + ".slack.com/api/" + family + ".history?token=" + token + "&channel=" + value;
    xhr.onreadystatechange = function() { 
            if (xhr.readyState == 4 && xhr.status == 200)
                callback(xhr.responseText);
        }
        xhr.open("GET", url, true); 
        xhr.send();
    }
}
 
slackClient.on('loggedIn', function(user, team) {
    bot = user;
    console.log("Logged in as " + user.name
        + " of " + team.name + ", but not yet connected");
});
 
slackClient.on('open', function() {

    var botChannels = Object.keys(slackClient.channels)
        .map(function (k) { return slackClient.channels[k]; })
        .filter(function (c) { return c.is_member; })
        .map(function (c) { return c.name; });

    var botGroups = Object.keys(slackClient.groups)
        .map(function (k) { return slackClient.groups[k]; })
        .filter(function (g) { return g.is_open && !g.is_archived; })
        .map(function (g) { return g.name; });
 
    console.log('Connected as ' + slackClient.self.name + ' of ' + slackClient.team.name);
 
    if (botChannels.length > 0) {
        console.log('You are in these public channels: ' + botChannels.join(', '));
    } else {
        console.log('You are not in any public channels.');
    }
 
    if (botGroups.length > 0) {
       console.log('You are in these private channels: ' + botGroups.join(', '));
    } else {
        console.log('You are not in any private channels.');
    }
});
 
slackClient.on('message', function(message) {
    // Ignore bot's own messages
    if (message.user == bot.id) return;
 
    // get the current channel, for posterity
    var channel = slackClient.getChannelGroupOrDMByID(message.channel);

    // only respond to messages directed at this bot
    if (message.type === 'message' && message.text.length >= 0 && message.text.indexOf(slackClient.self.id) > -1) { 
        searchString = message.text.split(" ").pop();
        console.log("Attempting to query channel: " + searchString);
        
        //retrieve the requested channel object
        var myChannel = slackClient.getChannelGroupOrDMByName(searchString);

        //check to verify we've returned a valid channel
        if (typeof myChannel != "undefined") {            
            // get the channel ID
            var myChannelID = myChannel['id'];
            // determine whether this is a group or a channel
            if (myChannel.getType() == 'Group') {
                family = "groups";
            } else {
                family = "channels";
            };
            // get the message history for the channel
            history = new getChannelHistory();
            history.get(family, myChannelID, function(response) {
                json = JSON.parse(response);
                mymessages = json['messages'];
                var unresolved = [];
                // filter for messages that don't have reactions
                for (var i = 0; mymessages.length > i; i++) {
                    var msgStatus = mymessages[i]['reactions'];
                    // add unresolved messages to our output array
                    if (typeof msgStatus == "undefined") {
                        unresolved.push(mymessages[i].text);
                    }
                };
                var myCount = unresolved.length;
                var list = unresolved.join("\n");
                // return output array
                channel.send("Your Unresolved Items: \n" + list);
                // log success to console
                console.log("Query successful! Returned " + myCount + " items")
            });
        // how to respond to bad requests
        } else {
            // taskbot is a fancy lady and speaks with correct grammar
            channel.send("I'm sorry, I don't understand. I need you to end your sentence with a valid channel name of which I am a member.");
            console.log("Query FAILED.");
            console.log("Did your sentence end with a valid channel name?");
            console.log("Have you invited Taskbot to join the requested channel?");
        }
    }; 
});
 
slackClient.login();
```