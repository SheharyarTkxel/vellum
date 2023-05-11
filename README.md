# a)plan AI

This doc provides basic instructions for generating messages for AI-powered coaching sessions between users of the **a)plan coaching app** and `a)plan AI`, a performance and life coach AI that will be integrated into the app.

Your humble author is James Ryan, the Narrative Systems Lead at [Hexagram](https://www.hexagram.io/); we are collaborating with a)plan to develop an AI-driven coaching platform for which `a)plan AI` is an initial proof of concept. **This document contains sensitive information and should only be shared between Hexagram, a)plan, and the a)plan coaching app development team.**

## Contents
* [Workflow](#workflow)
* [Examples](#examples)
* [Failure Cases](#failure-cases)
* [Additional Considerations](#additional-considerations)


## Workflow

Here’s the basic workflow for integrating `a)plan AI` into the a)plan coaching app:

* To get an `a)plan AI` message, the a)plan coaching app calls out to [Vellum](https://www.vellum.ai/), who is hosting a deployment of `a)plan AI`'s underlying model.
  - In Vellum terms, a *deployment* combines a model identifier (in our case, OpenAI’s `text-davinci-003`), an associated API key (the a)plan OpenAI API key), a prompt template, and a hyperparameter setting for the selected model.
  
  - The request to Vellum (examples below) must include the following:

      - name of the user for whom an `a)plan AI` message is being generated

      - transcript of the user's `a)plan AI` session so far

* Vellum forwards this request to OpenAI, who sends Vellum a response package including the generated reply.
  - Vellum brokers our connection to OpenAI, forwarding requests to them and sending back the results. It provides value through various development and analysis features exposed via the middle layer that is manages.

* Vellum sends its own response package to the a)plan coaching app (examples below), which includes the bot’s generated message.

* The a)plan coaching app parses out the message text and displays it to the user.

* The user replies.

* Rinse, repeat.


## Examples

Vellum’s API is pretty straightforward, and they do have Python ([github](https://github.com/vellum-ai/vellum-client-python), [pypi](https://pypi.org/project/vellum-ai/)) and JavaScript/TypeScript ([github](https://github.com/vellum-ai/vellum-client-node), [npm](https://www.npmjs.com/package/vellum-ai)) clients that you can use if you wish. Because details such as the model identifier and hyperparameter setting are contained in the deployment on Vellum, this information does not need to be included in requests to Vellum. This will also allow us to tune the model without necessitating close collaboration between Hexagram and the a)plan coaching app development team.

Here’s an example `curl` command for starting a new session between `a)plan AI` and a client named `Rob Auten`:

```
curl --request POST \
'https://predict.vellum.ai/v1/generate' \
--header 'accept: application/json' \
--header 'X-API-KEY: XDVFH9MX.FZtn6dr9F8ghMeyI7afh7FIUITfWTzay' \
--header 'content-type: application/json' \
--data-raw '{
    "requests": [
        {
            "input_values": {
                "clientName": "Rob Auten", 
                "transcript": ""
            },
            "external_ids": null
        }
    ],
    "deployment_id": null,
    "deployment_name": "a-plan-ai"
}'
```

First off, note the `X-API-KEY` header. This contains a)plan's Vellum API key (`XDVFH9MX.FZtn6dr9F8ghMeyI7afh7FIUITfWTzay`), not its OpenAI API key (which is contained in the Vellum deployment). The rest of the package should be fairly self-explanatory. The only fields that should be modified for any request are the nested `input_values` fields. These are strings that are used to fill gaps in the `a)plan AI` prompt template. At a higher level, they provide all the information that `a)plan AI` needs in order to generate a message that is tailored to the client and session at hand. As this project develops, we'll likely expand this set to include a number of other concerns.

In this case, we know this is an initial request for a new session becasue the `transcript` value is `""`. (Note that this assumes that the coach speaks first; if the user speaks first, you'll need to conform to the transcript format given just below.)

Here's the response that I got from Vellum when I submitted this request:

```
{
  "results": [
    {
      "data": {
        "completions": [
          {
            "id": "f98792c8-6af2-4662-8fc4-d2307f6dc429",
            "external_id": null,
            "text": " Hi, Rob. How are you doing today?",
            "finish_reason": "STOP",
            "logprobs": null,
            "model_version_id": "2f828506-4177-4f7b-a4bf-c6b0759bf7e1"
          }
        ]
      },
      "error": null
    }
  ]
}
```

In the future, we will want to extract some of this metadata, but for now you can simply grab the generated message text `" Hi, Rob. How are you doing today?"` that is stored at `results[0].data.completions[0].text`. Generated messages will always be retrievable from that same path (and they will likely always contain a leading whitespace character).

Now, let's say you've trimmed off the leading whitespace of this reply (to produce `"Hi, Rob. How are you doing today?"`) and displayed it to the user, who replies, `"Horrible!"`. Here's how the next request would look in this case:

```
curl --request POST \
'https://predict.vellum.ai/v1/generate' \
--header 'accept: application/json' \
--header 'X-API-KEY: XDVFH9MX.FZtn6dr9F8ghMeyI7afh7FIUITfWTzay' \
--header 'content-type: application/json' \
--data-raw '{
    "requests": [
        {
            "input_values": {
                "clientName": "Rob Auten", 
                "transcript": "Coach: Hi, Rob. How are you doing today?\nClient: Horrible!"
            },
            "external_ids": null
        }
    ],
    "deployment_id": null,
    "deployment_name": "a-plan-ai"
}'
```

As you can see, the `transcript` value now contains the history of the conversation so far. The format here is simple: `Coach: <coach line>\nClient: <client line>\nCoach: ...`. Please do take care to follow this format precisely, including insertion of a single whitespace character between an attributer (e.g., `Coach:`) and a line.

Here's the response that I got when I submitted this request to Vellum:

```
{
  "results": [
    {
      "data": {
        "completions": [
          {
            "id": "a3aa8835-2a9f-4d10-abbf-6f746e7311bd",
            "external_id": null,
            "text": " I'm sorry to hear that. What has been happening that's been making you feel this way?",
            "finish_reason": "STOP",
            "logprobs": null,
            "model_version_id": "2f828506-4177-4f7b-a4bf-c6b0759bf7e1"
          }
        ]
      },
      "error": null
    }
  ]
}
```

From here, the conversation can develop by the simple mechanism of updating the `transcript` field of each request.


## Failure Cases

* Total usage exceeds a)plan's OpenAI monthly spending cap.

  - The a)plan OpenAI account that funds the Vellum deployment has a monthly spending cap. If that cap is exceeded, you'll receive a response like this:

```
{
  "results": [
    {
      "data": null,
      "error": {
        "message": "Provider Error: OpenAI error: You exceeded your current quota, please check your plan and billing details."
      }
    }
  ]
}
```

* A given conversation goes too long.

  - OpenAI's `text-davinci-003` model has a maximum context of `4096` tokens, or approximately 3000 words. Currently, our base prompt (prior to appending the transcript) takes up nearly half that context, leaving only about 2000 tokens for the contents of the `transcript` field. In the future, we can consider various options for supporting longer conversation, but for now we may want to do something like display a message indicating that the user has maxed out their `a)plan AI` session. Once the transcript grows too large, the context will overflow, and you'll receive a response like this:

```
{
  "results": [
    {
      "data": null,
      "error": {
        "message": "Provider Error: OpenAI error: This model's maximum context length is 4097 tokens, however you requested 4132 tokens (3876 in your prompt; 256 for the completion). Please reduce your prompt; or completion length."
      }
    }
  ]
}
```  


## Additional Considerations

* `text-davinci-003` is an expensive model. In our current configuration, we can expect each generated message to cost between five and eight cents.

* All requests to Vellum persist there in a human-readable format.
