---
layout: post
title: Building My First Slackbot to Query Channel History
---

In this post, I will:

* Give a little bit of background
* Run through the logic of what exactly this app will do, with some notes about discoveries I made along the way
* Walk through some required initial setup (set up the Slack integration, install a couple of node libraries)
* Provide a couple of links to resources I reference
* Give the full final code, with inline commenting to walk through what is happening

## Background

We started using Slack in the fall of 2014, and at the time we weren’t totally sure how it was going to fit in to our workflow. Somehow between then and now, slack became my task management tool. I wanted a simple task management interface, where I could add tasks quickly from anywhere, and see at a glance whether a task was completed. Slack already had this functionality built-in: messaging, and emoji reactions. So I created a private channel for logging my tasks, and give them a green checkmark as I complete them, or a skull to kill the task. I'm usually pretty good about tagging tasks as either dead or complete, but every once in a while one slips through the cracks, and gets buried in my channel history. The one thing Slack is missing is a way to round up all those unresolved tasks and list them for me. This seemed like a great reason to build my first bot.

I don't have a ton of experience writing node apps. I've written a few things here and there, but so far I haven't written enough for it to become something I can just rattle off the top of my head. But I figured, what the hey! Let's do this with node.

I started this off the way I start most of these kind of projects: by going to Google to see if anyone has ever done anything like this before. I found two blog posts that became my jumping off points (and I even ended up borrowing some of each of their code in the final product):

* http://blog.somewhatabstract.com/2015/03/02/writing-a-simple-slack-bot-with-node-slack-client/
* http://nordicapis.com/building-an-intelligent-bot-using-the-slack-api/

Both projects had started with the slack-client API wrapper, so I went ahead and used that as my base as well.

## The App Logic

Let's start by listing out what we'll be doing in our bot code:

<ol>
<li><p>Ignore the bot's own messages</p></li>
<li><p>Get the current channel, so we know where to post messages to later</p>

<p>FYI, your bot needs to be a member of any channels that you want her to be active in. Invite her the same way you'd invite any other user.</p></li>

<li><p>Only respond to messages directed at this bot</p>

<p>You need to decide how you're going to talk to your bot. Do you want her to respond to all messages posted to a channel, or just messages directed at her? And do you want her to receive input?</p>

<p>For TaskBot, I chose to require a structured input message. When users want to call TaskBot, they must use the following format: “@taskbot: channel-name” (e.g., “@taskbot: nellie-notes”). This way, we can split up the message and feed TaskBot the name of the channel she should analyze; she'll use this to find the channel object, using functions that are part of the slack-client library. (Though the reality is that TaskBot is a bit smarter than we’re giving her credit for: as long as her name is referenced in the message, and the very last item in the message is a channel name, she’ll know what to do.)</p></li>

<li><p>Take the message that was sent to the bot and get the channel name</p></li>

<li><p>Retrieve the requested channel object using functions from the slack-client library</p>

<p>I couldn't find robust documentation of all the functions available in the slack-client library, but you can see what most of your options are by looking at the library code. For example, the function we'll be using can be found here: https://github.com/slackhq/node-slack-client/blob/master/src/client.coffee#L293. See line TK in the full code below.</p></li>

<li><p>Check to verify if we've returned a valid channel</p></li>

<li><p>Get the channel ID from the channel object</p></li>

<p>channels.history and groups.history (see the following note) require a channel ID. First you have to get the channel object, and then get the ID.</p></li>

<li><p>Determine whether this is a public or private channel (i.e., group) by looking at the data in the channel object</p>

<p>Public channels and private channels need different api calls. Although this bot is just for me right now in my private channels, eventually I'll implement a similar collaborative channel for my team to track which issues and support requests we’re each working on, so the bot needs to be able to handle both. Fortunately the calls are pretty similar: public channels need the prefix “channels”, and private channels need the prefix “groups”. The specific function calls are the same for both, so all we need to worry about is swapping out the prefix as appropriate.
</p></li>

<li><p>Get the message history for the channel using a custom API call</p>

<p>I couldn't get the slack-client history-fetching functions to work for me, so I decided to write my own API call function instead. It turns out that writing a JavaScript function to make an api call for a channel or group is actually pretty simple. You can use the standard http request format, and follow the format listed in the slack API docs (e.g., https://api.slack.com/methods/channels.history). The format for our history API call will be as follows: https://YOURTEAM.slack.com/api/channels.history?token=YOURTOKEN&amp;channel=CHANNELID (see line TK in the code below).</p>

<p>Our http request is going to run asynchronously, so remember that asynchronous functions just kind of run whenever it's convenient! This means that once we get our result from the API call, we need to put all the code that is going to work on the returned data inside our function.</p></li>

<li><p>Parse the returned messages to make them readable</p></li>
<li><p>Filter for messages that don't have reactions</p></li>
<li><p>Combine our array of unresolved messages into one line so the bot can send just one message to slack</p>

<p>Instead of printing each unresolved task as it's own message to Slack, I want the bot to just post a single list of my unresolved tasks. We can use new lines (\n) in Slack messages to create line breaks within a single message.</p></li>

<li><p>Print out our final, single message to slack</p></li>
<li><p>Print to the console that the query was successful</p></li>
<li><p>Finally, if the message to the bot was formatted wrong, print out some help text instead of doing all that other stuff</p></li>
</ol>

## Setup

In order to test and ultimately run Taskbot, you need to set up the integration for your slack team. Don't be intimidated: it's very easy and there's no real risk (you're not using up precious resources or anything).

Once the integration is set up, you'll also need to invite your bot to some channels. She needs to be a member of any channels you want her to participate in (or, in this case, analyze).

Additionally, we'll be using two libraries that need to be installed via npm:

npm install xmlhttprequest
npm install slack-client --save

## A Few Notes Up Front

* The channels.history API call, in its default form, maxes out at 100 messages. I'm pretty sure there's a way to get more pages of results after that, but I didn't bother trying to add that functionality. If there are more than 100 tasks between me and an unresolved task, I have bigger problems.

* This bot lives and runs locally via node, and needs to be started up when you want to use it. The next step would be to put it on a real web server (like Heroku) so everyone can use it whenever they want.

## Helpful Links

* The slack-client library: https://www.npmjs.com/package/slack-client
* The Slack API docs: https://api.slack.com/methods

## Final Result

And here's the full code, with comments inline to walk you through what's happening:

```javascript
// We're using the slack-client library
var SlackClient = require('slack-client');

// Automatically reconnect after an error response from Slack.
var autoReconnect = true;

// Put your bot API token here
var token = "YOUR TOKEN HERE";

// Put your slack team name here
// We'll use this when piecing together our API call
var team = "YOUR TEAM HERE";

var slackClient = new SlackClient(token, autoReconnect);
 
// Track bot user, for detecting the bot's own messages
var bot;

// We'll define our own custom API call to get channel history
// See NOTE TKTK
var XMLHttpRequest = require("xmlhttprequest").XMLHttpRequest;

var getChannelHistory = function() {
    this.get = function(family, value, callback) {
    var xhr = new XMLHttpRequest();
    // This builds the actual structure of the API call using our provided variables
    var url = "https://" + team + ".slack.com/api/" + family + ".history?token=" + token + "&channel=" + value;
    xhr.onreadystatechange = function() { 
            if (xhr.readyState == 4 && xhr.status == 200)
                callback(xhr.responseText);
        }
        xhr.open("GET", url, true); 
        xhr.send();
    }
}

// Tell us when the app is running
slackClient.on('loggedIn', function(user, team) {
    bot = user;
    console.log("Logged in as " + user.name
        + " of " + team.name + ", but not yet connected");
});

slackClient.on('open', function() {

    // Find out which public channels the bot is a member of
    var botChannels = Object.keys(slackClient.channels)
        .map(function (k) { return slackClient.channels[k]; })
        .filter(function (c) { return c.is_member; })
        .map(function (c) { return c.name; });

    // Find out which private channels the bot is a member of
    var botGroups = Object.keys(slackClient.groups)
        .map(function (k) { return slackClient.groups[k]; })
        .filter(function (g) { return g.is_open && !g.is_archived; })
        .map(function (g) { return g.name; });
 
    // Tell us when the bot is connected
    console.log('Connected as ' + slackClient.self.name + ' of ' + slackClient.team.name);
 
    // Print our list of channels and groups that we found above in the console
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

// What to do when a message is posted to one of the bot's channels
// This is the real meat of our app
slackClient.on('message', function(message) {
    // Ignore the bot's own messages
    if (message.user == bot.id) return;
 
    // Get the current channel, so we know where to post messages to later
    var channel = slackClient.getChannelGroupOrDMByID(message.channel);

    // We're wrapping our functionality in an IF statement
    // to only respond to messages directed at this bot
    if (message.type === 'message' && message.text.length >= 0 && message.text.indexOf(slackClient.self.id) > -1) { 
        // Take the message that was sent to the bot
        // And split off just the relevant bit
        // See NOTE TKTK
        searchString = message.text.split(" ").pop();

        // Print out the name of the channel the bot is analyzing
        console.log("Attempting to query channel: " + searchString);
        
        // Retrieve the requested channel object
        // using functions from the slack-client library
        var myChannel = slackClient.getChannelGroupOrDMByName(searchString);

        // Check to verify if we've returned a valid channel
        if (typeof myChannel != "undefined") {            
            // Get the channel ID from the channel object
            var myChannelID = myChannel['id'];

            // Determine whether this is a public or private channel (i.e., group)
            // by looking at the data in the channel object
            if (myChannel.getType() == 'Group') {
                family = "groups";
            } else {
                family = "channels";
            };

            // Get the message history for the channel
            // using our custom API call
            history = new getChannelHistory();
            history.get(family, myChannelID, function(response) {
                // Now that we have our messages, let's parse them to make them readable
                json = JSON.parse(response);

                // The history object had a few children, so we're just pulling out the messages child
                mymessages = json['messages'];

                // Create an array to hold our filtered list of messages
                var unresolved = [];

                // Filter for messages that don't have reactions
                for (var i = 0; mymessages.length > i; i++) {
                    // For each message, get the reactions (if any)
                    var msgStatus = mymessages[i]['reactions'];

                    // If the message has no reactions...
                    if (typeof msgStatus == "undefined") {
                        // ...add the message to our array
                        unresolved.push(mymessages[i].text);
                    }
                };

                // Once we've finished looping through all the messages
                // count how many didn't have reactions, for logging
                var myCount = unresolved.length;

                // Combine our array of unresolved messages into one line
                // so the bot can send just one message to slack
                var list = unresolved.join("\n");

                // Print out our final, single message to slack
                channel.send("Your Unresolved Items: \n" + list);

                // Print to the console that the query was successful,
                // and how many unresolved messages were found
                console.log("Query successful! Returned " + myCount + " items")
            });

        // If the message to the bot is formatted wrong, then do this:
        } else {
            // Tell the user how to format their request correctly
            channel.send("I'm sorry, I don't understand. I need you to end your sentence with a valid channel name of which I am a member.");

            // Also print to the console that the query failed, and give some helpful pointers
            console.log("Query FAILED.");
            console.log("Did your sentence end with a valid channel name?");
            console.log("Have you invited Taskbot to join the requested channel?");
        }
    }; 
});

slackClient.login();
```