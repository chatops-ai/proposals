# Architecture Proposal

Check out and comment on the google doc: https://docs.google.com/document/d/1aa7RVAPvgCcIfIKY3ooBYMVCjVf2np5Z1cYGBruj7lE/edit?usp=sharing

You can join the slack community by hitting our [invite link](https://t.co/jAGkypjcQDJ). We also have a GitHub team for contributors! Once you've gotten a PR merged in, notify anyone from the `og` team and they'll add you as a contributor.

## Event Lifecyle

In order to clarify exactly what is being built, I'd like to explain the lifecycle of an event handled by both the chatops.ai router and DialogFlow. For any event to occur, the service-specific agent is deployed. A service is defined as a communication platform than can host chatbots, i.e. Slack, HipChat, Skype. For example, Slack defines custom bots as "Apps" even though chatops.ai bots take advantage of webhooks, their multiple API's via their API client (node.js?). In this example, Gunter, along with his Slack plugin, would be deployed onto slack and send out a `ack` `POST` request to the event router. 

The Agent is composed of both the service-specific deployment configurations (plugins) and their event router. 

Via communication with the service-specific Agent, the even router will receive `POST`ed payload from Slack based on triggered events, webhooks, and any other configuration setup for service-specific use. 

Once the [event router](https://github.com/chatops-ai/proposals/issues/2) has parsed the JSON, it stores (?) relevant information into the `store`; a library built to around an interface that allows multiple backends to be switch in and out without conflict. Google's DataStore k:v NoSQL service is the preferred `store` backend. Once complete, the event router parses the data into the correct format for `POST`ing to `/query` in DialogFlow, in order to retrieve a response from the users message. An example request and response is below.

```JSON
 POST https://api.dialogflow.com/v1/query?v=20150910

  Headers:
  Authorization: Bearer YOUR_CLIENT_ACCESS_TOKEN
  Content-Type: application/json

  POST body:
  {
    "contexts": [
      "shop"
    ],
    "lang": "en",
    "query": "I need apples",
    "sessionId": "12345",
    "timezone": "America/New_York"
  }
```
Response:
```JSON
{
  "id": "3622be70-cb49-4796-a4fa-71f16f7b5600",
  "lang": "en",
  "result": {
    "action": "pickFruit",
    "actionIncomplete": false,
    "contexts": [
      "shop"
    ],
    "fulfillment": {
      "messages": [
        {
          "platform": "google",
          "textToSpeech": "Okay how many apples?",
          "type": "simple_response"
        },
        {
          "platform": "google",
          "textToSpeech": "Okay. How many apples?",
          "type": "simple_response"
        },
        {
          "speech": "Okay how many apples?",
          "type": 0
        }
      ],
      "speech": "Okay how many apples?"
    },
    "metadata": {
      "intentId": "21478be9-bea6-449b-bcca-c5f009c0a5a1",
      "intentName": "add-to-list",
      "webhookForSlotFillingUsed": "false",
      "webhookUsed": "false"
    },
    "parameters": {
      "fruit": [
        "apples"
      ]
    },
    "resolvedQuery": "I need apples",
    "score": 1,
    "source": "agent"
  },
  "sessionId": "12345",
  "status": {
    "code": 200,
    "errorType": "success"
  },
  "timestamp": "2017-09-19T21:16:44.832Z"
}
```
It is not handed off for preparation for an API called to the integration router. This is where integration's, not services, communicate via API requests from the integration router. A [plugin](https://github.com/chatops-ai/proposals/issues/4) consists of a few parts:
- The API client
- The [DialogParser](https://github.com/chatops-ai/proposals/issues/5), which runs on the integration router (proposal PR linked)
- An intents/ folder with DialogFlow intents
- An entities/ folder with DialogFlow entities
- A data/ folder with .txt files containing train data; one sentence per line.
Once the API client returns the appropriate data, the DialogFlow parser returns it to DialogFlow, and takes the same JSON payload from earlier, posting it to slack via DialogFlow's slack integration. The following is an example of a robust slack rich message response, which would be populated with data from the DialogParser:
```JSON
{
    "text": "New comic book alert!",
    "attachments": [
        {
            "title": "The Further Adventures of Slackbot",
            "fields": [
                {
                    "title": "Volume",
                    "value": "1",
                    "short": true
                },
                {
                    "title": "Issue",
                    "value": "3",
                    "short": true
                }
            ],
            "author_name": "Stanford S. Strickland",
            "author_icon": "http://a.slack-edge.com/7f18/img/api/homepage_custom_integrations-2x.png",
            "image_url": "http://i.imgur.com/OJkaVOI.jpg?1"
        },
        {
            "title": "Synopsis",
            "text": "After @episod pushed exciting changes to a devious new branch back in Issue 1, Slackbot notifies @don about an unexpected deploy..."
        },
        {
            "fallback": "Would you recommend it to customers?",
            "title": "Would you recommend it to customers?",
            "callback_id": "comic_1234_xyz",
            "color": "#3AA3E3",
            "attachment_type": "default",
            "actions": [
                {
                    "name": "recommend",
                    "text": "Recommend",
                    "type": "button",
                    "value": "recommend"
                },
                {
                    "name": "no",
                    "text": "No",
                    "type": "button",
                    "value": "bad"
                }
            ]
        }
    ]
}
```

And that is the lifecycle of an event. I will give a more detailed example, later.

## Persistent Store 

## API Gateway and Event Triggers

Triggers:
- AWS Simple Notification Service
- AWS Simple Queue Service
- AWS API Gateway
- External webhooks
- Service adapter triggers

## Serverless Architecture via AWS lambda

Due to the nature of each events lifecycle, I see no reason why we cannot take advantage of the cost savings + future proof layer of abstraction that is gcp cloud functions/aws lambda functions. Each router could be broken down into functions trigger via services and plugins. This will be expanded on, but is most likely the direction I will be taking.

Google cloud supports node.js currently; AWS supports multiple languages; most importantly, Go, using `HandlerFunc`s, which is basically no different than the original approach.

```Go
// Handler is your Lambda function handler
// It uses Amazon API Gateway request/responses provided by the aws-lambda-go/events package,
// However you could use other event sources (S3, Kinesis etc), or JSON-decoded primitive types such as 'string'.
func Handler(request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {

 // stdout and stderr are sent to AWS CloudWatch Logs
 log.Printf("Processing Lambda request %s\n", request.RequestContext.RequestID)

 // If no name is provided in the HTTP request body, throw an error
 if len(request.Body) < 1 {
  return events.APIGatewayProxyResponse{}, ErrNameNotProvided
 }

 return events.APIGatewayProxyResponse{
  Body:       "Hello " + request.Body,
  StatusCode: 200,
 }, nil

}

func main() {
 lambda.Start(Handler)
}
```

This also encourages node.js developers to also build DialogParsers/ChatOps.ai integration with more services.

Resources:
- [AWS API Gateway](https://aws.amazon.com/api-gateway/)
- [AWS Lambda](https://aws.amazon.com/lambda/)

## Plugin Architecture

## Plugin Architecture

In order to allow room for growth and development with trained chatbots, we should provide a defined solution for extending a trained bot's capabilities. This assumes the bot is being trained via DialogFlow, a free NLU managed service offered by Google.

A plugin should consist of the following components:
- API client
- Dialog parsers
- config.yaml
- Intents
- Entities
- Training materials
 
The API client could be an existing third party client, an official one, or a custom built client for use specifically with the plugin.

### Dialog Parsers

A dialog parser is similar to a collection of middleware layers; it defines a sort of schema that your plugin will accept; it then parses it, passes it to and from DialogFlow, and finally delivery the payload back to the router.

### Configuration

For configuration, I mirror my proposal of using `/v1/yaml` to ingest `yaml` files and store them in the `store` after processing them. When the code needs to get credentials or variables defined in the `config.yaml` file, it can be retrieved from the `store` and used as a global constant (?). 

The `yaml` must provide, at minimum, the following information for the router:

- Routes -> `HandlerFunc(s)` name, corresponds with API actions
- Attributes -> All entities involved with the routes
- Credentials in the form of upper case strings with the `_` separator
- Author
- Website
- Version

Example:

```Yaml
Version: v1.2.9
Website: chatops.ai
Author: Alec Cunningham
Credentials:
  // These are stored, encrypted (dependent on the `store` backend)
  // as KEY:VALUE pairs for retrieval when the router 
  // initializes its connection with the service
  DATABASE_USER: Alec
  DATABASE_PASS: gunter123!
  SERVICE_TOKEN: fkladfj98032hbui3f8
Routes:
  // We name space via the API version and plugin name
  // to avoid issues with different services + same action
  - /v1/jenkins/start
  - /v1/jenkins/stop
  - /v1/jenkins/build
  - /v1/jenkins/jobs
Attributes:
  // Note, any attribute values you recieve will
  // be from the DialogFlow payload 
  ID: [start, stop, build, jobs]
  BRANCH: [start, stop, jobs]
  JOB_PARAM: [start, jobs]
```
Following the definitions from the `config.yaml`, the router should be a singular file named after its service, like `jenkins.go`. In it it need to, at minimum, fufill this interface:
```Go
type Plugin interface {
          Connect(string)      (Client, error)
          Close(string)
          HealthCheck(string)  *Status
          Trigger(string)      *Action
}
```
As shown above, the plugin interface requires three structs:
```Go
// i.e. type Jenkins struct
type Client struct {
         // plugin satisfies Plugin interface
         plugin plugin.Plugin
         // baseUrl is  the API's base URL domain
         baseUrl string
         // request is a Request object that executes API calls
         request *Requester
}

// APIRequest defines what a request to <plugin> API needs
type APIRequest struct {
	Method   string
	Endpoint string
	Payload  io.Reader
	Headers  http.Header
	Suffix   string
}

// Requester is the one making calls
type Requester struct {
	Base      string
	BasicAuth *BasicAuth
	Client    *http.Client
	CACert    []byte
	SslVerify bool
}

// Status ties a build and a status together
type Status struct { 
        // The external job in question
        job string
        // version is either version number or can be used for build number
        version int
        // status follows the global constants for reporting healthchecks
        status string
}
```
Universal constants: 
```Go
// Universal statuses for plugins
const (
	STATUS_FAIL           = "FAIL"
	STATUS_ERROR          = "ERROR"
	STATUS_ABORTED        = "ABORTED"
	STATUS_REGRESSION     = "REGRESSION"
	STATUS_SUCCESS        = "SUCCESS"
	STATUS_FIXED          = "FIXED"
	STATUS_PASSED         = "PASSED"
)
```
### Service Handlers vs Plugins

For clarification; service providers are the platforms that run the bot, while plugins are packages the router uses to enable communication with another service.

Service providers include:
- Facebook Messenger
- Slack
- Twitter
- Skype
- A bunch more that are not as relevant
**WIP**

## Service Adapters (Handlers)

Ex: 

```Go
/*
Copyright 2016 Skippbox, Ltd.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package handlers

import (
	"github.com/bitnami-labs/kubewatch/config"
	"github.com/bitnami-labs/kubewatch/pkg/handlers/slack"
	"github.com/bitnami-labs/kubewatch/pkg/handlers/hipchat"
)

// Handler is implemented by any handler.
// The Handle method is used to process event
type Handler interface {
	Init(c *config.Config) error
	ObjectCreated(obj interface{})
	ObjectDeleted(obj interface{})
	ObjectUpdated(oldObj, newObj interface{})
}

// Map maps each event handler function to a name for easily lookup
var Map = map[string]interface{}{
	"default": &Default{},
	"slack":   &slack.Slack{},
	"hipchat": &hipchat.Hipchat{},
}

// Default handler implements Handler interface,
// print each event with JSON format
type Default struct {
}

// Init initializes handler configuration
// Do nothing for default handler
func (d *Default) Init(c *config.Config) error {
	return nil
}

func (d *Default) ObjectCreated(obj interface{}) {

}

func (d *Default) ObjectDeleted(obj interface{}) {

}

func (d *Default) ObjectUpdated(oldObj, newObj interface{}) {

}
```
