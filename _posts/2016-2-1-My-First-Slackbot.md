---
layout: post
title: Building a Slack Bot with Node.js to Query Channel History
---

Posted by [Nellie McKesson](http://macmillanpublishers.github.io/blog/about/)

## Background

Shortly after we started using Slack on our team in 2014, it also became my task management tool. I hate having my notes spread over multiple systems, and Slack meets all my main task-management-system requirements: 

* Easy to access (the Slack app is already open on my Desktop all day)
* Add tasks quickly from anywhere (messaging)
* See at a glance whether a task was completed (emoji reactions)

I use a private channel for logging my tasks, and give them a green checkmark as I complete them, or a skull to kill the task. I'm usually pretty good about tagging tasks as either dead or complete, but every once in a while a task slips through the cracks, and gets buried in my channel history. Slack doesn't have a way to round up and list my "unresolved" (i.e. un-marked) "tasks" (i.e., messages), so I decided to build a bot to do it.

I don't have a ton of experience writing node apps. I've written a few things here and there, but so far I haven't written enough for it to become something I can just rattle off the top of my head. But learning is cool! So I decided to go with node.

I started this off the way I start most of these kinds of projects: by going to Google to see if anyone has ever done anything like this before. I found two blog posts that became my jumping off points (and I even ended up borrowing some of each of their code in the final bot):

* [Jeff Yates' Somewhat Abstract blog: "Writing A Simple Slack Bot With Node slack-client"](http://blog.somewhatabstract.com/2015/03/02/writing-a-simple-slack-bot-with-node-slack-client/)
* [Nordic APIS blog: "Building an Intelligent Bot Using the Slack API" by Dennis Hotson](http://nordicapis.com/building-an-intelligent-bot-using-the-slack-api/)

Both projects started with the _slack-client_ API wrapper, so I went ahead and used that as my base as well.

## Setup

I'm going to start off by going over some of the set up involved in building my "TaskBot" (as she shall hereafter be known), and giving you a preview of the final result. I'm not going to walk you through how to install node or any of the other basic prerequisites, but there are a lot of resources online about that--believe in yourself.

In order to test and ultimately run TaskBot, you need to set up the integration for your Slack team. Don't be intimidated: it's very easy and there's no real risk (you're not using up precious resources or anything). [Follow this link to your Slack settings](https://my.slack.com/services/new/bot), pick a username for your bot, and add the integration. Slack will return an API token when the bot is activated--copy this! You'll need it in the final code.

Once the integration is set up, you'll also need to invite your bot to some channels. She needs to be a member of any channels you want her to participate in (or, in this case, analyze). Invite her the same way you'd invite any user.

Additionally, we'll be using two libraries that need to be installed via npm:

```
$ npm install xmlhttprequest
$ npm install slack-client
```

That ought to do it! Once you've got the code finalized (see the next couple sections of this post) and saved as a .js file (I called mine _app.js_), launch Terminal and the Slack app, and run the app via the command line like this:

```
$ node app.js
```

Here's what you'll see in the Slack app and in the terminal when you make a request to the bot with the final app:

<img src="https://raw.githubusercontent.com/macmillanpublishers/blog/gh-pages/images/bot-final-1.png" alt="The Slack client displaying the bot query and results"/>

<img src="https://raw.githubusercontent.com/macmillanpublishers/blog/gh-pages/images/bot-final-2.png" alt="The terminal showing a successful query"/>

This bot lives and runs locally via node, and needs to be started up when you want to use it. The next step would be to put it on a real web server (like Heroku) so everyone can use it whenever they want.

## The App Logic

Alright, we've seen what the bot will do and how to run it, so let's take a step back and talk about how to actually build it. I start most of my new projects by writing out the logic of what my app will need to do. Of course things change as I start coding, but it's a great way to start thinking through how you'll need to structure things and where the dependencies are. Here's a linear outline of what we'll be doing in our bot code:

<ol>
<li><p><strong>Watch a channel for messages</strong></p></li>
<li><p><strong>Ignore the bot's own messages</strong></p>

<p>Let's not create an infinite loop.</p></li>

<li><p><strong>Get the current channel, so we know where to post messages to later</strong></p>

<p>Your bot needs to be a member of any channels that you want her to be active in--this means both channels where you're planning to talk to her, and channels that you want her to analyze (I don't want my bot--who I shall call TaskBot--posting in my tasks channel, so I'm planning on talking to her and asking her to post my task lists in a different channel). Invite her the same way you'd invite any other user.</p></li>

<li><p><strong>Only respond to messages directed at this bot</strong></p>

<p>We don't want our bot to respond to every single message posted in a channel--she only needs to pay attention when someone mentions her name.</p></li>

<li><p><strong>Take the message that was sent to the bot and pull out just the name of the channel that is being queried</strong></p>

<p>For TaskBot, I chose to require a structured input message. When users want to call TaskBot, they must use the following format: “@taskbot: channel-name” (e.g., “@taskbot: nellie-notes”). This way, we can split up the message and feed TaskBot just the name of the channel she should analyze; she'll use this to find the channel object, using functions that are part of the <em>slack-client</em> library. (Though the reality is that TaskBot is a bit smarter than we’re giving her credit for: as long as her name is referenced in the message, and the very last item in the message is a channel name, she’ll know what to do. So you could also say, "Hey @taskbot you incredible genius, tell me the unresolved tasks that are in nellie-notes".)</p></li>

<li><p><strong>Retrieve the requested channel object using functions from the <em>slack-client</em> library</strong></p>

<p>I couldn't find robust documentation of all the functions available in the <em>slack-client</em> library, but you can see what most of your options are by looking at the library code. For example, the function we'll be using to get a channel object by it's name <a href="https://github.com/slackhq/node-slack-client/blob/master/src/client.coffee#L293">can be found here</a>.</p></li>

<li><p><strong>Check to verify if we've returned a valid channel object</strong></p></li>

<li><p><strong>Get the channel ID from the channel object</strong></p></li>

<p><em>channels.history</em> and <em>groups.history</em> (see the following note) require a channel ID. First you have to get the channel object, and then get the ID from that object.</p></li>

<li><p><strong>Determine whether this is a public or private channel (i.e., group) by looking at the data in the channel object</strong></p>

<p>Public channels and private channels need different API calls. Although this bot is just for me right now in my private channels, eventually I'll implement a similar collaborative channel for my team to track which issues and support requests we’re each working on, so the bot needs to be able to handle both. Fortunately the calls are pretty similar: public channels need the prefix “channels”, and private channels need the prefix “groups”. The specific function calls are the same for both, so all we need to worry about is swapping out the prefix as appropriate.
</p></li>

<li><p><strong>Get the message history for the channel using a custom API call</strong></p>

<p>I couldn't get the <em>slack-client</em> library's history-fetching functions to work for me, so I decided to write my own API call function instead. It turns out that writing a JavaScript function to make an API call for a channel or group is actually pretty simple. You can use the standard httprequest format, and follow the url format listed in the slack API docs (https://api.slack.com/methods/channels.history). The url format for our <em>channels.history</em> API call will be as follows: _https://YOURTEAM.slack.com/api/channels.history?token=YOURTOKEN&amp;channel=CHANNELID_.</p>

<p>Our http request is going to run asynchronously, so remember that asynchronous functions just kind of run whenever it's convenient! This means that once we get our result from the API call, we need to put all the code that is going to do something with the returned data <em>inside</em> our function.</p>

<p>FYI, the <em>channels.history</em> API call, in its default form, maxes out at 100 messages. I'm pretty sure there's a way to get more pages of results after that, but I didn't bother trying to add that functionality. If there are more than 100 tasks between me and an unresolved task, I have bigger problems.</p></li>

<li><p><strong>Parse the returned messages to make them readable JSON objects</strong></p></li>
<li><p><strong>Filter for messages that don't have reactions</strong></p>

<p>"Reactions" are the emojis people add--in this case, my check marks and skulls. For my purposes, it doesn't matter which reaction is attached to a message; all that matters is whether a reaction is present or not.</p></li>

<li><p><strong>Combine our array of unresolved messages into one line so the bot can send just one message to Slack</strong></p>

<p>Instead of printing each unresolved task as it's own message to Slack, I want the bot to just post a single list of my unresolved tasks. We can use new lines (\n) in Slack messages to create line breaks within a single message.</p></li>

<li><p><strong>Print out our final, single message to Slack</strong></p></li>
<li><p><strong>Print to the console that the query was successful</strong></p></li>
<li><p><strong>Finally, if the message to the bot was formatted wrong, print out some help text instead of doing all that other stuff</strong></p></li>
</ol>

## The App Code

And here's ALL the final code, with comments inline to walk you through what's happening. I’m not going to explain this step-by-step or dive too deeply into the principles of each step, or why I structured my code certain ways. If you're like me, and you like to learn by looking at code and trying stuff out, then you should feel right at home. Below the code there are a few links to resources I reference, and then we're done!

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
// See the note for step 10 above
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
 
  // Print our list of channels and groups 
  // that we found above in the console
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

// What to do when a message is posted 
// to one of the bot's channels--
// This is the real meat of our app
slackClient.on('message', function(message) {
  // Ignore the bot's own messages
  if (message.user == bot.id) return;
 
  // Get the current channel, 
  // so we know where to post messages to later
  var channel = slackClient.getChannelGroupOrDMByID(message.channel);

  // We're wrapping our functionality in an IF statement
  // to only respond to messages directed at this bot
  if (message.type === 'message' && message.text.length >= 0 && message.text.indexOf(slackClient.self.id) > -1) { 
    // Take the message that was sent to the bot
    // And split off just the relevant bit
    // See the note for step 4 above
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

      // Determine whether this is a public 
      // or private channel (i.e., group)
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
        // Now that we have our messages, 
        // let's parse them to make them readable
        json = JSON.parse(response);

        // The history object had a few children, 
        // so we're just pulling out the "messages" child
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
      console.log("Have you invited TaskBot to join the requested channel?");
    }
  }; 
});

slackClient.login();
```

## Useful Links

* [The slack-client library](https://www.npmjs.com/package/slack-client)
* [The Slack API docs](https://api.slack.com/methods)