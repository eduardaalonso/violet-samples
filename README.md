## Table Of Contents

* [Prerequisites & Setup](#prerequisites--setup)
* [Getting Started with Voice Scripting](#getting-started-with-voice-scripting)
  * [Basics](#basics)
  * [Conversational Goals](#goals)
  * [Plugins](#plugins)
    * [Persistence](#persistence)
    * [Timed delay](#timed-delay)
    * [Violet Client Integration](#violet-client-integration)
  * [Advanced](#advanced)
    * [Custom types](#custom-types)
* [Local execution](#local-execution)
* [Deploying a Conversation Script](#deploying-a-conversation-script)
* [Debugging Conversations](#debugging-conversations)
* [Contribution/Supporting](#contributionsupporting)
* [Filing issues](#filing-issues)


# violet-conversations

`violet-conversations` provides support for building sophisticated
conversational apps/bots on Amazon's Alexa. Conversations are built via scripts,
and Violet provides a conversation engine that runs as an Alexa Skill. This
project is configured to create a skill at the `alexa/einstein` end point.

## Prerequisites & Setup

The Conversation Engine needs to run in the cloud so that Amazon's voice servers can access it. It can also be run locally via a non-voice (web-only) interface.

* Install Node v6 or greater - if you need to maintain an older version of node, consider using `nvm`.

* Download dependencies: `npm install`

* If you are using the Salesforce integration plugin (as used by the Diabetes
Script) you will need to set up environment variables (for more information see
the [Persistence](#persistence) plugin section below).

* To run locally: `npm start`

## Getting Started with Voice Scripting

See `server.js` for a typical initialization and script loading code. Every
voice script should start will typically start with declaring `violet` for use
throughout:
```javascript
var violet = require('../lib/violet.js').script();
```

In the above `einstein` is the script name and defines the end-point where the
script logic is to be made available.

See `examples/tutorial.js` for documentation on how to build a skill.

### Basics

A simple way to respond to a user request:
```javascript
violet.respondTo(['Can you help me', 'What can I do'],
 (response) => {
   response.say(`I can help you with your current account balance, with
                  financial planning, budgeting, investing, or taking out a
                  loan`);
 });
 ```

If you want to accept data from a user, you will need to declare it with its
type and then tell the script engine where to expect it:
```javascript
violet.addKeyTypes({
  'name': 'AMAZON.US_FIRST_NAME',
});

violet.respondTo(['My name is [[name]]'],
 (response) => {
   response.say('I like the name [[name]]')
 });
```

You can use phrase equivalents to tell the engine that two phrases are identical
in all situations:
```javascript
violet.addPhraseEquivalents([
  ['My name is', 'I call myself'],
]);
```

### Conversational Goals

Sophisticated conversations are supported by Violet using goals. When a user
indicates a need for a discussion on a particular topic, a script can just add
the topic as a goal to be met. Goals could theoretically also be set by a script
to proactively check-in with a user or to raise an item related to another goal.

The below simply adds a goal to be met by the system when the user asks for
flight arrival information.
```javascript
violet.respondTo('What time does the [[airline]] flight arrive', 'from [[city]]',
  (response) => {
    response.addGoal('[[flightArrivalTime]]');
});

violet.respondTo('What time does the flight arrive from [[city]]',
  (response) => {
    response.addGoal('[[flightArrivalTime]]');
});
```

The conversation engine tries to meet goals based on goal definitions provided
in the script. Goals are one of two types. The first type of goal is when
additional information is needed
```javascript
violet.defineGoal({
  goal: '[[flightArrivalTime]]',
  resolve: (response) => {
    if (!response.goalFilled('airline')
        || !response.goalFilled('city')
        || !response.goalFilled('flightDay') ) {
          return false; // dependent goals not met
        }
    var airline = response.get('[[airline]]');
    var city = response.get('[[city]]');
    var flightDay = response.get('[[flightDay]]');
    flightArrivalTimeSvc.query(airline, city, flightDay, (arrivalTime)=>{
      response.say('Flight ' + airline + ' from ' + city + ' is expected to arrive ' + flightDay + ' at ' + arrivalTime);
    });
    return true;
  }
});
```

```javascript
violet.defineGoal({
  goal: '[[city]]',
  prompt: ['What city do you want the flight to be arriving from'],
  respondTo: [{
    expecting: '[[city]]',
    resolve: (response) => {
      response.set('[[city]]', response.get('[[city]]') );
  }}]
});
```

### Plugins
There are a number of plugins that allow you to further extend the capabilities
of Violet Skills

#### Persistence
These plugins will allow you to easily store and load data from a data store.

The main plugin provided currently is the Salesforce integration plugin
`violetStoreSF`. If you are using this integration,
then you will need to set up the following environment variables (locally and on
any deployed platform): `V_SFDC_CLIENT_ID`, `V_SFDC_CLIENT_SECRET`,
`V_SFDC_USERNAME` and `V_SFDC_PASSWORD`.

To include the plugin in your code you would need to add this after the violet
script:
```javascript
var violetStoreSF = require('../lib/violetStoreSF.js')(violet);
```


```javascript
violet.respondTo('I recieved a bill from [[company]] today for [[amount]]',
  (response) => {
    response.store('bills',
        'user': response.get('[[userId]]'),
        'from': response.get('[[company]]'),
        'amount': response.get('[[amount]]')
      });
    response.say('xxxx');
});
```

To retried data from the database you need to use the load method. **Caution**
Load returns a promise to the db results  - if you need to use the
values retrieved from it, you need to (a) either use a .then after it OR
(b) you need to make you resolve method a generator and place a yield before
the call to the load method

```javascript
// example a:
violet.respondTo(
  expecting: 'Who did I receive my bill from most recently?',
  resolve: (response) => {
    return response.load('bills', 'bills.user', response.get('[[userId]]') )
      .then(()=>{
        response.say('You received a bill from <<bills.from>> for <<bills.amount>>');
      });
});
// example b: (note the 'function *' and the 'yield' below)
violet.respondTo(
  expecting: 'Who did I receive my bill from most recently?',
  resolve: function *(response) {
    yield response.load('bills', 'bills.user', response.get('[[userId]]') )
    response.say('You received a bill from <<bills.from>> for <<bills.amount>>');
});
```

#### Timed delay
```javascript
var violetTime = require('../lib/violetTime.js')(violet);
```

Possible spoken commands:
* What is the current time
* Advance 5 days, hours, or minutes

```javascript
violetTime.repeat(48*60, ()=>{ violet.addGoal('[[checkIn]]'); });
```


#### Violet Client Integration
```javascript
var violetClientTx = require('../lib/violetClientTx.js')(violet);
```

Allows setting voice and creating alerts (the latter needs testing)


### Advanced

#### Custom types

The voice engine can recognize better what the user is saying when it knows to
expect one of a limited set of inputs. One helpful way to do this is to use
create custom type and have it as an input parameter in your script.

In the below example - from violetTime.js - a custom type called `timeUnit` is
declared.

```javascript
violet.addKeyTypes({
  "time": "NUMBER",
  "timeUnit": {
    "type": "timeUnitType",
    "values": ["days", "hours", "minutes"]
  },
});
```

After declaring a custom type, the values and the type name need to be provided
to Amazon's Skill Configuration Site as a custom slot type.

## Local execution

Can be run locally to ensure no syntax errors and to view intent schemas (for
the interaction model) and supported utterances.

To run locally `npm install` followed by `npm start`
and view the einstein skill configuration information by going to
 http://localhost:8080/alexa/einstein

## Deploying a Conversation Script

Deploy the service (already has `Procfile` for deploying to `heroku`), and use
those settings to create a new skill at Amazon's Skill Configuration Site:
https://developer.amazon.com/edw/home.html#/skills/list. The skill will be
using a `Custom Interaction Model`, will need values from the services skill
configuration page.

For step-by-step instructions - see here: https://salesforce.quip.com/I8YOAC3Q1UGC

## Debugging Conversations

When developing conversational scripts - it helps to debug/test it in three phrases:
1. Make sure the code compiles/runs by typing `npm start`. Fix any errors and keep re-starting the service until misplaced punctuations and declarations have been fixed.
2. Test the script in the included tester view, by running the script and opening it in a browser, for example: http://localhost:8080/alexa/einstein You will likely want to submit IntentRequest's based on the Utterance's at the bottom of the page. Once you submit a request, verify that the response output SSML is per your needs. Additionally, it is helpful to walk through the script a few times to ensure that the application supports the different user scenarios.
3. Once the script works locally, deploy it to the cloud and configure Alexa to talk to the underlying skill using Amazon's Skill Configuration site. At this stage you will likely benefit from testing by iterating rapidly with: invoking the voice-client, examining the conversational-app's logs, and tweaking the utterances in Amazon's Configuration. Testing a voice client is likely best done first through a PC based tool that provides additional debugging information like the [Violet Client](https://github.com/sfxi/violet-client) or a web testing tool like [Echosim.io](https://echosim.io).

## Contribution/Supporting

We appreciate any help. In order of decreasing priority, we would like to see:

* Sophisticated conversational applications built on top of the `violet-conversations` project. As you build it, let us know how it went. What worked well? What was challenging?

* Tests: If you want to make sure that we improve the framework that we don't break a use case that you are depending on - make sure we have an automated test.

## Filing issues

To help us look into any issue please create a document with the following:
* **Record Type** - Defect or Enhancement?
* **Date**
* **Description**
* **Supporting Information**
  - If **defect**, please include steps to reproduce AND include Violet-Conversation app logs here.
  - If **enhancement**, please include use case info here.
* **Owner** - Who is logging the issue?

Please post any such document in the Violet trailblazer chatter group.
