# Botkit and Slack

Table of Contents

* [Getting Started](#getting-started)
* [Create a Controller](#create-a-controller)
* [Slack-specific Events](#event-list)
* [Events API](#events-api)
* [Slash commands](#slash-commands)
* [Incoming Webhooks](#incoming-webhooks)
* [Using the Slack Web API](#using-the-slack-web-api)
* [Interactive Messages](#interactive-messages)
* [Dialogs](#dialogs)
* [Uploaded Files](#files)
* [Ephemeral Messages](#ephemeral-messages)
* [Slack Threads](#slack-threads)
* [Using the RTM Connection](#using-the-rtm-connection)

## Getting Started

1. [Install Botkit on your computer](https://botkit.ai/getstarted.html)

2. Create a Botkit powered Node app:
  * [Remix the starter project on Glitch](https://glitch.com/~botkit-slack)
  * Or: Use the command line tool:

      ```
      mkdir mybot
      cd mybot
      yo botkit
      ```

3. [Follow this guide to configuring the Slack API](/docs/provisioning/slack-events-api.md)

### Things to note
Much like a vampire, a bot has to be invited into a channel. DO NOT WORRY bots are not vampires.

Type: `/invite @<my bot>` to invite your bot into another channel.


## Create a Controller

To connect Botkit to Slack, use the Slack constructor method, [Botkit.slackbot()](#botkitslackbot).
This will create a Botkit controller with [all core features](core.md#botkit-controller-object) as well as [some additional methods](#additional-controller-methods).

#### Botkit.slackbot()
| Argument | Description
|--- |---
| config | an object containing configuration options

Returns a new Botkit SlackBot controller.

The `config` argument is an object with these properties:

| Name | Type | Description
|--- |--- |---
| studio_token | String | An API token from [Botkit Studio](readme-studio.md)
| debug | Boolean | Enable debug logging
| clientId | string | client Id value from a Slack app
| clientSecret | string | client secret value from a Slack app
| scopes | array | list of scopes to request during oauth
| stale_connection_timeout  | Positive integer | Number of milliseconds to wait for a connection keep-alive "pong" response before declaring the connection stale. Default is `12000`
| send_via_rtm  | Boolean   | Send outgoing messages via the RTM instead of using Slack's RESTful API which supports more features
| retry | Positive integer or `Infinity` | Maximum number of reconnect attempts after failed connection to Slack's real time messaging API. Retry is disabled by default
| api_root | url |  Alternative root URL which allows routing requests to the Slack API through a proxy, or use of a mocked endpoints for testing. defaults to `https://slack.com`
| disable_startup_messages | Boolean | Disable start up messages, like: `"Initializing Botkit vXXX"`
| clientSigningSecret        | String | Value of signing secret from Slack used to confirm source of incoming messages
| clientVerificationToken | String | [Deprecated](https://api.slack.com/docs/verifying-requests-from-slack#about) Value of verification token from Slack used to confirm source of incoming messages
For example:

```javascript
var controller = Botkit.slackbot({
    clientId: process.env.clientId,
    clientSecret: process.env.clientSecret,
    scopes: ['bot'],
});
```

## Event List

In addition to the [core events that Botkit fires](core.md#receiving-messages-and-events), this connector also fires some platform specific events.

In fact, Botkit will receive, normalize and emit any event that it receives from Slack.
This includes all of the events [listed here](https://api.slack.com/events), as well as
events based on the `subtype` field of incoming messages, [as listed here](https://api.slack.com/events/message#message_subtypes)

### Incoming Message Events
| Event | Description
|--- |---
| direct_message | the bot received a direct message from a user
| direct_mention | the bot was addressed directly in a channel
| mention | the bot was mentioned by someone in a message
| ambient | the message received had no mention of the bot

### User Activity Events:
| Event | Description
|--- |---
| bot_channel_join | the bot has joined a channel
| user_channel_join | a user has joined a channel
| bot_group_join | the bot has joined a group
| user_group_join | a user has joined a group

### Websocket Events:

| Event | Description
|--- |---
| rtm_open | a connection has been made to the RTM api
| rtm_close | a connection to the RTM api has closed
| rtm_reconnect_failed | if retry enabled, retry attempts have been exhausted

### Application Lifecycle Events:

| Event | Description
|--- |---
| create_incoming_webhook | The app was installed and created an incoming webhook integration
| create_bot | The app was installed and created a bot user integration
| create_team | The app was installed on a new team
| update_team | The app was installed by a second user on an existing team
| create_user | A new user completed the oauth process
| update_user | An existing user completed the oauth process again
| oauth_error | An error occured during the oauth process


## Events API

The [Events API](https://api.slack.com/events-api) is a streamlined way to build apps and bots that respond to activities in Slack. You must setup a [Slack App](https://api.slack.com/slack-apps) to use Events API. Slack events are delivered to a secure webhook, and allows you to connect to slack without the RTM websocket connection.

[Read our step-by-step guide to setting up the Events API to work with Botkit](provisioning/slack-events-api.md)


## Slash commands

Slash commands are special commands triggered by typing a "/" then a command.
[Read Slack's official documentation here.](https://api.slack.com/slash-commands)

### Handling `slash_command` events

```javascript
controller.on('slash_command',function(bot,message) {

    // reply to slash command
    bot.replyPublic(message,'Everyone can see this part of the slash command');
    bot.replyPrivate(message,'Only the person who used the slash command can see this.');

})
```

### Securing Outgoing Webhooks and Slash commands

You can optionally protect your application with authentication of the requests
from Slack.  Slack will generate a unique request token for each Slash command and
outgoing webhook (see [Slack documentation](https://api.slack.com/slash-commands#validating_the_command)).
You can configure the web server to validate that incoming requests contain a valid api token
by adding an express middleware authentication module.

This can also be achieved by passing a signing secret as `clientSigningSecret` into the initial call `Botkit.slackbot()` used to create the controller. This was previously achieved by passing a verification token as `clientVerificationToken` into the initial call `Botkit.slackbot()` used to create the controller.


```javascript
controller.setupWebserver(port,function(err,express_webserver) {
  controller.createWebhookEndpoints(express_webserver, ['AUTH_TOKEN', 'ANOTHER_AUTH_TOKEN']);
  // you can pass the tokens as an array, or variable argument list
  //controller.createWebhookEndpoints(express_webserver, 'AUTH_TOKEN_1', 'AUTH_TOKEN_2');
  // or
  //controller.createWebhookEndpoints(express_webserver, 'AUTH_TOKEN');
});
```



#### bot.replyAcknowledge

| Argument | Description
|---  |---
| callback | optional callback

When used with slash commands, this function responds with a 200 OK response
with an empty response body.
[View Slack's docs here](https://api.slack.com/slash-commands)


#### bot.replyPublic()

| Argument | Description
|---  |---
| src | source message as received from slash or webhook
| reply | reply message (string or object)
| callback | optional callback

When used with outgoing webhooks, this function sends an immediate response that is visible to everyone in the channel.

When used with slash commands, this function has the same functionality. However,
slash commands also support private, and delayed messages. See below.

[View Slack's docs here](https://api.slack.com/slash-commands)

#### bot.replyPrivate()

| Argument | Description
|---  |---
| src | source message as received from slash
| reply | reply message (string or object)
| callback | optional callback

Send a response to the slash command that is only visible to the user who executed the command.

Note that this function can only be used ONCE, and must be used within 3 seconds of the initial event or Slack will consider it to have failed.

To send multiple or delayed replies, use [bot.replyPrivateDelayed()](#botreplyprivatedelayed)


#### bot.replyPublicDelayed()

| Argument | Description
|---  |---
| src | source message as received from slash
| reply | reply message (string or object)
| callback | optional callback

#### bot.replyPrivateDelayed()

| Argument | Description
|---  |---
| src | source message as received from slash
| reply | reply message (string or object)
| callback | optional callback

Send up to 5 private responses within 30 minutes of the initial slash command event.


## Incoming webhooks

Incoming webhooks allow you to send data from your application into Slack.
To configure Botkit to send an incoming webhook, first set one up
via [Slack's integration page](https://my.slack.com/services/new/incoming-webhook/).

Once configured, use the `sendWebhook` function to send messages to Slack.

[Read official docs](https://api.slack.com/incoming-webhooks)

#### bot.configureIncomingWebhook()

| Argument | Description
|--- |---
| config | Configure a bot to send webhooks

Add a webhook configuration to an already spawned bot.
It is preferable to spawn the bot pre-configured, but hey, sometimes
you need to do it later.

#### bot.sendWebhook()

| Argument | Description
|--- |---
| message | A message object
| callback | _Optional_ Callback in the form function(err,response) { ... }

Pass `sendWebhook` an object that contains at least a `text` field.
 This object may also contain other fields defined [by Slack](https://api.slack.com/incoming-webhooks) which can alter the
 appearance of your message.

```javascript
var bot = controller.spawn({
  incoming_webhook: {
    url: <my_webhook_url>
  }
})

bot.sendWebhook({
  text: 'This is an incoming webhook',
  channel: '#general',
},function(err,res) {
  if (err) {
    // ...
  }
});
```

## Using the Slack Web API

All (or nearly all - they change constantly!) of Slack's current web api methods are supported
using a syntax designed to match the endpoints themselves.

If your bot has the appropriate scope, it may call [any of these methods](https://api.slack.com/methods) using this syntax:

```javascript
bot.api.channels.list({},function(err,response) {
  //Do something...
})
```

## Interactive Messages

Slack applications can use "interactive messages" to include buttons, menus and other interactive elements to improve the user's experience.

### via Blocks
The preferred way of composing interactive messages is using Slack's Block Kit.  [Read the official Slack documentation here](https://api.slack.com/messaging/composing/layouts). Slack provides a UI to help create your interactive messages. Check out [Block Kit Builder](https://api.slack.com/tools/block-kit-builder).

Interactive messages using blocks can be sent via any of Botkit's built in functions by passing in the appropriate "blocks" as part of the message.  Here is an example:

```javascript
const content = {
    blocks: [{...}]; // insert valid JSON following Block Kit specs
};

bot.reply(message, content);
```

### via Attachments
Attachments are still supported by Slack, but the preferred way is to use Block Kit. [Read the official Slack documentation here](https://api.slack.com/docs/message-buttons)

### Handling Button Tigger
If your interactive message contains a button, when the user clicks the button in Slack, Botkit triggers an event based on the message type.

|Message Type|Event Name
|---  |---
|block_actions|block_actions
|interactive_message|interactive_message_callback

When an event is received, your bot can either reply with a new message, or use the special `bot.replyInteractive` function which will result in the original message in Slack being _replaced_ by the reply. Using `replyInteractive`, bots can present dynamic interfaces inside a single message.

In order to use interactive messages, your bot will have to be [registered as a Slack application](https://api.slack.com/apps), and will have to use the Slack button authentication system.
To receive callbacks, register a callback url as part of applications configuration. Botkit's built in support for the Slack Button system supports interactive message callbacks at the url `https://_your_server_/slack/receive` Note that Slack requires this url to be secured with https.

During development, a tool such as [localtunnel.me](http://localtunnel.me) is useful for temporarily exposing a compatible webhook url to Slack while running Botkit privately.

```javascript
// set up a botkit app to expose oauth and webhook endpoints
controller.setupWebserver(process.env.port,function(err,webserver) {

  // set up web endpoints for oauth, receiving webhooks, etc.
  controller
    .createHomepageEndpoint(controller.webserver)
    .createOauthEndpoints(controller.webserver,function(err,req,res) { ... })
    .createWebhookEndpoints(controller.webserver);

});
```


### Send an interactive message uing Attachments
```javascript
controller.hears('interactive', 'direct_message', function(bot, message) {

    bot.reply(message, {
        attachments:[
            {
                title: 'Do you want to interact with my buttons?',
                callback_id: '123',
                attachment_type: 'default',
                actions: [
                    {
                        "name":"yes",
                        "text": "Yes",
                        "value": "yes",
                        "type": "button",
                    },
                    {
                        "name":"no",
                        "text": "No",
                        "value": "no",
                        "type": "button",
                    }
                ]
            }
        ]
    });
});
```

### Receive a block_actions callback event

```javascript
// receive an interactive message, and reply with a message that will replace the original
controller.on('block_actions', function(bot, message) {
...
});
```

### Receive an interactive message callback

```javascript
// receive an interactive message, and reply with a message that will replace the original
controller.on('interactive_message_callback', function(bot, message) {

    // check message.actions and message.callback_id to see what action to take...

    bot.replyInteractive(message, {
        text: '...',
        attachments: [
            {
                title: 'My buttons',
                callback_id: '123',
                attachment_type: 'default',
                actions: [
                    {
                        "name":"yes",
                        "text": "Yes!",
                        "value": "yes",
                        "type": "button",
                    },
                    {
                       "text": "No!",
                        "name": "no",
                        "value": "delete",
                        "style": "danger",
                        "type": "button",
                        "confirm": {
                          "title": "Are you sure?",
                          "text": "This will do something!",
                          "ok_text": "Yes",
                          "dismiss_text": "No"
                        }
                    }
                ]
            }
        ]
    });

});
```

### Using Interactive Messages in Conversations

It is possible to use interactive messages in conversations, with the `convo.ask` function.

When used in conjunction with `convo.ask`, Botkit will treat the button's `value` field as if were a message typed by the user.

```javascript
bot.startConversation(message, function(err, convo) {

    convo.ask({
        attachments:[
            {
                title: 'Do you want to proceed?',
                callback_id: '123',
                attachment_type: 'default',
                actions: [
                    {
                        "name":"yes",
                        "text": "Yes",
                        "value": "yes",
                        "type": "button",
                    },
                    {
                        "name":"no",
                        "text": "No",
                        "value": "no",
                        "type": "button",
                    }
                ]
            }
        ]
    },[
        {
            pattern: "yes",
            callback: function(reply, convo) {
                convo.say('FABULOUS!');
                convo.next();
                // do something awesome here.
            }
        },
        {
            pattern: "no",
            callback: function(reply, convo) {
                convo.say('Too bad');
                convo.next();
            }
        },
        {
            default: true,
            callback: function(reply, convo) {
                // do nothing
            }
        }
    ]);
});
```

## Dialogs

[Dialogs](https://api.slack.com/dialogs) allow bots to present multi-field pop-up forms in response to a button click or other interactive message interaction.
Botkit provides helper functions and special events to make using dialogs in your app possible.

### Create a dialogs

Dialogs can be created in response to `interactive_message_callback` or `slash_command` events.
Botkit provides a specialized reply function, `bot.replyWithDialog()` and an object builder function,
`bot.createDialog()` that should be used to create and send the dialog.

#### bot.createDialog()
| Argument | Description
|---  |---
| title | title of dialog
| callback_id | value for the callback_id field, used to identify the form
| submit_label | the label for the submit button.
| elements | an optional array of pre-specified [elements objects](https://api.slack.com/dialogs#elements)

This function returns an `dialog object` with additional methods for creating the form elements,
and using them with `bot.replyWithDialog`.  These functions can be chained together.


```javascript
var dialog = bot.createDialog(
         'Title of dialog',
         'callback_id',
         'Submit'
       ).addText('Text','text','some text')
        .addSelect('Select','select',null,[{label:'Foo',value:'foo'},{label:'Bar',value:'bar'}],{placeholder: 'Select One'})
        .addTextarea('Textarea','textarea','some longer text',{placeholder: 'Put words here'})
        .addUrl('Website','url','http://botkit.ai');

bot.replyWithDialog(message, dialog.asObject());
```

#### dialog.title()
| Argument | Description
|---  |---
| value | value for the dialog title

#### dialog.callback_id()
| Argument | Description
|---  |---
| value | value for the dialog title

#### dialog.submit_label()
| Argument | Description
|---  |---
| value | value for the dialog title

#### dialog.addText()
| Argument | Description
|---  |---
| label | label of field
| name | name of field
| value | optional value
| options | an object defining additional parameters for this elements

Add a one-line text element to the dialog. The `options` object can contain [any of the parameters listed in Slack's documentation for this element](https://api.slack.com/dialogs),
including `placeholder`, `optional`, `min_length` and `max_length`

#### dialog.addEmail()
| Argument | Description
|---  |---
| label | label of field
| name | name of field
| value | optional value
| options | an object defining additional parameters for this elements

Add a one-line email address element to the dialog. The `options` object can contain [any of the parameters listed in Slack's documentation for this element](https://api.slack.com/dialogs),
including `placeholder`, `optional`, `min_length` and `max_length`

#### dialog.addNumber()
| Argument | Description
|---  |---
| label | label of field
| name | name of field
| value | optional value
| options | an object defining additional parameters for this elements

Add a one-line number element to the dialog. The `options` object can contain [any of the parameters listed in Slack's documentation for this element](https://api.slack.com/dialogs),
including `placeholder`, `optional`, `min_length` and `max_length`

#### dialog.addTel()
| Argument | Description
|---  |---
| label | label of field
| name | name of field
| value | optional value
| options | an object defining additional parameters for this elements

Add a one-line telephone number element to the dialog. The `options` object can contain [any of the parameters listed in Slack's documentation for this element](https://api.slack.com/dialogs),
including `placeholder`, `optional`, `min_length` and `max_length`

#### dialog.addUrl()
| Argument | Description
|---  |---
| label | label of field
| name | name of field
| value | optional value
| options | an object defining additional parameters for this elements

Add a one-line url element to the dialog. The `options` object can contain [any of the parameters listed in Slack's documentation for this element](https://api.slack.com/dialogs),
including `placeholder`, `optional`, `min_length` and `max_length`

#### dialog.addTextarea()
| Argument | Description
|---  |---
| label | label of field
| name | name of field
| value | optional value
| options | an object defining additional parameters for this elements
| subtype | optional: can be email, url, tel, number or text

Add a one-line text element to the dialog. The `options` object can contain [any of the parameters listed in Slack's documentation for this element](https://api.slack.com/dialogs),
including `placeholder`, `optional`, `min_length` and `max_length`

#### dialog.addSelect()
| Argument | Description
|---  |---
| label | label of field
| name | name of field
| value | optional value
| option_list | an array of option objects in the form [{label:, value:}]
| options | an object defining additional parameters for this elements
| subtype | optional: can be email, url, tel, number or text

Add a one-line text element to the dialog. The `options` object can contain [any of the parameters listed in Slack's documentation for this element](https://api.slack.com/dialogs),
including `placeholder` and `optional`

#### dialog.toObject()

Return the dialog as a Javascript object

#### dialog.asString()

Return the dialog as JSON

#### bot.replyWithDialog
| Argument | Description
|---  |---
| src | incoming `interactive_message_callback` or `slash_command` event
| dialog | a dialog object, created by `bot.createDialog().toObject()` or defined to Slack's spec

Sends a dialog in response to a button click or slash command. The dialog will appear in Slack as a popup window.
This function uses the API call `bot.api.dialog.open` to actually deliver the dialog.

```javascript
// launch a dialog from a button click
controller.on('interactive_message_callback', function(bot, trigger) {

  // is the name of the clicked button "dialog?"
  if (trigger.actions[0].name.match(/^dialog/)) {

        var dialog =bot.createDialog(
              'Title of dialog',
              'callback_id',
              'Submit'
            ).addText('Text','text','some text')
              .addSelect('Select','select',null,[{label:'Foo',value:'foo'},{label:'Bar',value:'bar'}],{placeholder: 'Select One'})
             .addTextarea('Textarea','textarea','some longer text',{placeholder: 'Put words here'})
             .addUrl('Website','url','http://botkit.ai');


        bot.replyWithDialog(trigger, dialog.asObject(), function(err, res) {
          // handle your errors!
        });

    }

});
```


### Receive Dialog Submissions

When a user in Slack submits a dialog, your bot will receive a `dialog_submission` event
which can be handled using standard `controller.on('dialog_submission', function handler(bot, message) {})` format.

The form submission values can be found in the `message.submission` field, which is an object containing
key value pairs that match the fields specified in the dialog.
In addition, the message will have a `message.callback_id` field which identifies the form.

Slack recommends an additional layer of server-side validation of the values in `message.submission`.
To respond with an error, use `bot.dialogError()`

Otherwise, you must call `bot.dialogOk()` to tell Slack that the submission has been successfully received by your app.
They also recommend that the bot respond in some other way, such as sending a follow-up message. You can use
the normal `bot.reply` or `bot.whisper` functions to send such a message.

In the example below, we define a `receive middleware` that performs additional validation on the submission.

```javascript
// use a receive middleware hook to validate a form submission
// and use bot.dialogError to respond with an error before the submission
// can be sent to the handler
controller.middleware.receive.use(function validateDialog(bot, message, next) {


  if (message.type=='dialog_submission') {

    if (message.submission.number > 100) {
       bot.dialogError({
          "name":"number",
          "error":"Please specify a value below 100"
          });            
      return;
    }
  }

  next();

});


// handle a dialog submission
// the values from the form are in event.submission    
  controller.on('dialog_submission', function(bot, message) {
    var submission = message.submission;
    bot.reply(message, 'Got it!');

    // call dialogOk or else Slack will think this is an error
    bot.dialogOk();
});
```

#### bot.dialogOk()

Send a success acknowledgement back to Slack. You _must_ call either dialogOk() or dialogError() in response to `dialog_submission` events,
otherwise Slack will display an error and will appear to reject the form submission.

#### bot.dialogError()
| Argument | Description
|---  |---
| error | a single error object {name:, error:} or an array of error objects

Send one or more validation errors back to Slack to display in the dialog.
The parameter can be one or more objects, where the `name` field matches the
name of the field in which the error is present.


## Updating Messages

Messages in Slack can be replaced with new versions in response to button clicks and other events.

#### bot.replyAndUpdate()

| Argument | Description
|---  |---
| src | source message as received from slash or webhook
| reply | reply message that might get updated (string or object)
| callback | optional asynchronous callback that performs a task and updates the reply message

Sending a message, performing a task and then updating the sent message based on the result of that task is made simple with this method:

> **Note**: For the best user experience, try not to use this method to indicate bot activity. Instead, use `bot.startTyping`.

```javascript
// fixing a typo
controller.hears('hello', ['ambient'], function(bot, msg) {
  // send a message back: "hellp"
  bot.replyAndUpdate(msg, 'hellp', function(err, src, updateResponse) {
    if (err) console.error(err);
    // oh no, "hellp" is a typo - let's update the message to "hello"
    updateResponse('hello', function(err) {
      console.error(err)
    });
  });
});
```


## Files

Downloading files shared in Slack requires a special authorization header in the request. The necessary value is located in `bot.config.bot.token`.

Below is an example handler for the `file_share` event that fetches the file and writes it to the filesystem.

```javascript
var request = require('request'); 
var fs = require('fs');
  
controller.on('file_share', function(bot, message) {

	var destination_path = '/tmp/uploaded';

	// the url to the file is in url_private. there are other fields containing image thumbnails as appropriate
	var url = message.file.url_private;

	var opts = {
		method: 'GET',
		url: url,
		headers: {
		  Authorization: 'Bearer ' + bot.config.bot.token, // Authorization header with bot's access token
		}
	};

	request(opts, function(err, res, body) {
		// body contains the content
		console.log('FILE RETRIEVE STATUS',res.statusCode);          
	}).pipe(fs.createWriteStream(destination_path)); // pipe output to filesystem
});
```  


## Ephemeral Messages

Using the Web API, messages can be sent to a user "ephemerally" which will only show to them, and no one else. Learn more about ephemeral messages at the [Slack API Documentation](https://api.slack.com/methods/chat.postEphemeral). When sending an ephemeral message, you must specify a valid `user` and `channel` id. Valid meaning the specified user is in the specified channel. Currently, updating interactive messages are not supported by ephemeral messages, but you can still create them and listen to the events. They will not have a reference to the original message, however.

### Ephemeral Message Authorship

Slack allows you to post an ephemeral message as either the user you have an auth token for (would be your bot user in most cases), or as your app. The display name and icon will be different accordingly. The default is set to `as_user: true` for all functions except `bot.sendEphemeral()`. Override the default of any message by explicitly setting `as_user` on the outgoing message.


#### bot.whisper()
| Argument | Description
|--- |---
| src | Message object to reply to, **src.user is required**
| message | _String_ or _Object_ Outgoing response
| callback | _Optional_ Callback in the form function(err,response) { ... }

Functions the same as `bot.reply()` but sends the message ephemerally. Note, src message must have a user field set in addition to a channel


`bot.whisper()` defaults to `as_user: true` unless otherwise specified on the message object. This means messages will be attributed to your bot user, or whichever user who's token you are making the API call with.


#### bot.sendEphemeral()
| Argument | Description
|--- |---
| message | _String_ or _Object_ Outgoing response, **message.user is required**
| callback | _Optional_ Callback in the form function(err,response) { ... }

To send a spontaneous ephemeral message (which slack discourages you from doing) use `bot.sendEphemeral` which functions similarly as `bot.say()` and `bot.send()`


```javascript
controller.hears(['^spooky$'], function(bot, message) {
	// default behavior, post as the bot user
	bot.whisper(message, 'Booo! This message is ephemeral and private to you')
})

controller.hears(['^spaghetti$'], function(bot, message) {
	// attribute slack message to app, not bot user
	bot.whisper(message, {as_user: false, text: 'I may be a humble App, but I too love a good noodle'})
})

controller.on('custom_triggered_event', function(bot, trigger) {
	// pretend to get a list of user ids from out analytics api...
	fetch('users/champions', function(err, userArr) {
		userArr.map(function(user) {
			// iterate over every user and send them a message
			bot.sendEphemeral({
				channel: 'general',
				user: user.id,
				text: "Pssst! You my friend, are a true Bot Champion!"})
		})
	})
})
```

### Ephemeral Conversations

To reply to a user ephemerally in a conversation, pass a message object to `convo.say()` `convo.sayFirst()` `convo.ask()` `convo.addMessage()` `convo.addQuestion()` that sets ephemeral to true.

When using interactive message attachments with ephemeral messaging, Slack does not send the original message as part of the payload. With non-ephemeral interactive messages Slack sends a copy of the original message for you to edit and send back. To respond with an edited message when updating ephemeral interactive messages, you must construct a new message to send as the response, containing at least a text field.

```javascript
controller.hears(['^tell me a secret$'], 'direct_mention, ambient, mention', function(bot, message) {
	bot.startConversation(message, function(err, convo) {
		convo.say('Better take this private...')
		convo.say({ ephemeral: true, text: 'These violent delights have violent ends' })
	})
})

```

## Slack Threads

Messages in Slack may now exist as part of a thread, separate from the messages included in the main channel.
Threads can be used to create new and interesting interactions for bots. [This blog post discusses some of the possibilities.](https://blog.howdy.ai/threads-serious-software-in-slack-ba6b5ceec94c#.jzk3e7i2d)

Botkit's default behavior is for replies to be sent in-context. That is, if a bot replies to a message in a main channel, the reply will be added to the main channel. If a bot replies to a message in a thread, the reply will be added to the thread. This behavior can be changed by using one of the following specialized functions:

#### bot.replyInThread()
| Argument | Description
|--- |---
| message | Incoming message object
| reply | _String_ or _Object_ Outgoing response
| callback | _Optional_ Callback in the form function(err,response) { ... }

This specialized version of [bot.reply()](core.md#botreply) ensures that the reply being sent will be in a thread.
When used to reply to a message that is already in a thread, the reply will be properly added to the thread.
Developers who wish to ensure their bot's replies appear in threads should use this function instead of bot.reply().

#### bot.startConversationInThread()
| Argument | Description
|---  |---
| message   | incoming message to which the conversation is in response
| callback  | a callback function in the form of  function(err,conversation) { ... }

Like [bot.startConversation()](core.md#botstartconversation), this creates conversation in response to an incoming message.
However, the resulting conversation and all followup messages will occur in a thread attached to the original incoming message.

#### bot.createConversationInThread()
| Argument | Description
|---  |---
| message   | incoming message to which the conversation is in response
| callback  | a callback function in the form of  function(err,conversation) { ... }

Creates a conversation that lives in a thread, but returns it in an inactive state.  See [bot.createConversation()](core.md#botcreateconversation) for details.



## Custom auth flows
In addition to the Slack Button, you can send users through an auth flow via a Slack interaction.
The `getAuthorizeURL` provides the url. It requires the `team_id` and accepts an optional `redirect_params` argument.
```javascript
controller.getAuthorizeURL(team_id, redirect_params);
```

The `redirect_params` argument is passed back into the `create_user` and `update_user` events so you can handle
auth flows in different ways. For example:

```javascript
controller.on('create_user', function(bot, user, redirect_params) {
    if (redirect_params.slash_command_id) {
        // continue processing the slash command for the user
    }
});
```



## Using the RTM Connection

Legacy custom bot users connect to Slack using a [real time API based on web sockets](https://api.slack.com/rtm). The bot connects to Slack using the same protocol that the native Slack clients use.

It is now Slack's official recommendation that developers use the [Events API](#events-api)
to receive messages rather than RTM connections, but it still works. [Read this FAQ to learn more.](https://api.slack.com/faq#events_api)

To connect a bot to Slack, [get a Bot API token from the Slack integrations page](https://my.slack.com/services/new/bot).

Note: Since API tokens can be used to connect to your team's Slack, it is best practices to handle API tokens with caution. For example, pass tokens into your application via an environment variable or command line parameter, rather than including it in the code itself.
This is particularly true if you store and use API tokens on behalf of users other than yourself!

[Read Slack's Bot User documentation](https://api.slack.com/bot-users)

### Require Delivery Confirmation for RTM Messages

In order to guarantee the order in which your messages arrive, Botkit supports an optional
delivery confirmation requirement. This will force Botkit to wait for a confirmation events
for each outgoing message before continuing to the next message in a conversation.

Developers who send many messages in a row, particularly with payloads containing images or attachments,
should consider enabling this option. Slack's API sometimes experiences a delay delivering messages with large files attached, and this delay can cause messages to appear out of order. Note that for Slack, this is only applies to bots with the `send_via_rtm` option enabled.

To enable this option, pass in `{require_delivery: true}` to your Botkit Slack controller, as below:

```javascript
var controller = Botkit.slackbot({
    require_delivery: true,
})
```

#### bot.startRTM()
| Argument | Description
|--- |---
| callback | _Optional_ Callback in the form function(err,bot,payload) { ... }

Opens a connection to Slack's real time API. This connection will remain
open until it fails or is closed using `closeRTM()`.

The optional callback function receives:

* Any error that occurred while connecting to Slack
* An updated bot object
* The resulting JSON payload of the Slack API command [rtm.start](https://api.slack.com/methods/rtm.start)

The payload that this callback function receives contains a wealth of information
about the bot and its environment, including a complete list of the users
and channels visible to the bot. This information should be cached and used
when possible instead of calling Slack's API.

A successful connection the API will also cause a `rtm_open` event to be
fired on the `controller` object.


#### bot.closeRTM()

Close the connection to the RTM. Once closed, an `rtm_close` event is fired
on the `controller` object.


```javascript
var Botkit = require('botkit');

var controller = Botkit.slackbot();

var bot = controller.spawn({
  token: my_slack_bot_token
})

bot.startRTM(function(err,bot,payload) {
  if (err) {
    throw new Error('Could not connect to Slack');
  }

  // close the RTM for the sake of it in 5 seconds
  setTimeout(function() {
      bot.closeRTM();
  }, 5000);
});
```

#### bot.destroy()

Completely shutdown and cleanup the spawned worker. Use `bot.closeRTM()` only to disconnect
but not completely tear down the worker.


```javascript
var Botkit = require('botkit');
var controller = Botkit.slackbot();
var bot = controller.spawn({
  token: my_slack_bot_token
})

bot.startRTM(function(err, bot, payload) {
  if (err) {
    throw new Error('Could not connect to Slack');
  }
});

// some time later (e.g. 10s) when finished with the RTM connection and worker
setTimeout(bot.destroy.bind(bot), 10000)
```

## Additional Controller Methods

#### controller.createWebhookEndpoints()

This function configures the route `http://_your_server_/slack/receive`
to receive webhooks from Slack.

This url should be used when configuring Slack.

When a slash command is received from Slack, Botkit fires the `slash_command` event.


#### controller.configureSlackApp()

| Argument | Description
|---  |---
| config | configuration object containing clientId, clientSecret, redirectUri and scopes

Configure Botkit to work with a Slack application.

Get a clientId and clientSecret from [Slack's API site](https://api.slack.com/applications).
Configure Slash command, incoming webhook, or bot user integrations on this site as well.

Configuration must include:

* clientId - Application clientId from Slack
* clientSecret - Application clientSecret from Slack
* redirectUri - the base url of your application
* scopes - an array of oauth permission scopes

Slack has [_many, many_ oauth scopes](https://api.slack.com/docs/oauth-scopes)
that can be combined in different ways. There are also [_special oauth scopes_
used when requesting Slack Button integrations](https://api.slack.com/docs/slack-button).
It is important to understand which scopes your application will need to function,
as without the proper permission, your API calls will fail.

#### controller.createOauthEndpoints()
| Argument | Description
|---  |---
| webserver | an Express webserver Object
| error_callback | function to handle errors that may occur during oauth

Call this function to create two web urls that handle login via Slack.
Once called, the resulting webserver will have two new routes: `http://_your_server_/login` and `http://_your_server_/oauth`. The second url will be used when configuring
the "Redirect URI" field of your application on Slack's API site.


```javascript
var Botkit = require('botkit');
var controller = Botkit.slackbot();

controller.configureSlackApp({
  clientId: process.env.clientId,
  clientSecret: process.env.clientSecret,
  redirectUri: 'http://localhost:3002',
  scopes: ['incoming-webhook','team:read','users:read','channels:read','im:read','im:write','groups:read','emoji:read','chat:write:bot']
});

controller.setupWebserver(process.env.port,function(err,webserver) {

  // set up web endpoints for oauth, receiving webhooks, etc.
  controller
    .createHomepageEndpoint(controller.webserver)
    .createOauthEndpoints(controller.webserver,function(err,req,res) { ... })
    .createWebhookEndpoints(controller.webserver);

});

```
