# SignalWire ML (SWML)

SignalWire ML (SWML) is SignalWire's markup language for controlling calls and messages. The script native format is JSON, but can also be specified in YAML for human readability.

## Document fetching

1. Incoming call is routed to a call node
2. Call node sends RTC call_accept to Prime Rails.
3. Prime Rails replies with URL for SWML
4. SignalWire sends POST to webhook with body containing customer-defined script variables and the current call object:

```json
{
  "call": {
    "project_id": "project uuid",
    "space_id": "space_uuid",
    "call_id": "uuid",
    "state": "created",
    "type": "sip",
    "from": "sip:+15551231234@example.com",
    "to": "sip:+15553214321@example.com",
    "headers": [
      { "name": "X-Header-1", "value": "X-Header-1-Value" },
      { "name": "X-Header-2", "value": "X-Header-2-Value" }
    ]
  },
  "vars": {
    "customer_script_var_1": "customer_script_var_1_value",
    "customer_script_var_2": "customer_script_var_2_value"
  },
  "params": {
    "param_1": "param_1_value",
    "param_2": "param_2_value"
  }
}
```

5. Server replies with SignalWire ML Script with `application/json` or `application/yaml` or `text/x-yaml` content-type.

## Versioning

At the top level of a script will be a `version` field that is a string containing a semantic version for the document schema. Suggested use is MAJOR version reflects breaking changes where a script may not operate correctly without changes such as existing methods having new required parameters or renaming and refactoring.  The MINOR version can be used as new methods are introduced or new optional fields for existing methods, and PATCH version can be used to reflect only the most minor bug fixes which still have some impact on the schema.

## Document Execution

The document contains a `sections` field that is a mapping of named sections for execution. The `main` section is always executed first. There must always be a `main` section in the document. The `transfer` method is used to transfer the call execution to a new URL.

Each section is an Array of strings or objects. Strings can be used for methods that do not require parameters like `answer`, `hangup`, or `denoise`. Use objects to execute methods with parameters. Each string or object in the section's array is executed in order until the section is completed. If there are no more methods to execute, the call ends.

## Expression Evaluation with JavaScript

All text wrapped in `%{}` is evaluated as JavaScript or as a plain variable substitution.

### Variables and call parameters

All variables and call parameters are delivered in the POST body to the URL in `transfer` and `execute`. Use the `set` and `unset` methods to set or delete variables.

Variables are only set or unset as a side effect of SignalWire ML methods executed by the script. All variables can be referenced inside `%{}` JavaScript expressions in the `vars.` JavaScript object. Variables can not be permanently changed inside JavaScript expressions. 

All Call object parameters are referenceable inside JavaScript expressions. For example `%{call.from}` or `%{call.headers.X-Customer-ID}`. Call object parameters are read only.

## Script example

In this example, the main section is executed to answer the call, play two files, then prompt for input. If 1 is pressed, the flow loops back to the welcome message. Otherwise the call is hung up.

### JSON

```json
{
  "version": "1.0.0",
  "sections": {
    "main": [
      "answer",
      {
        "label": "welcome",
        "play": {
          "volume": 10.0,
          "urls": [
            "https://example.com/welcome.mp3",
          ]
        }
      },
      { "prompt": ["https://example.com/prompt.wav"] },
      {
        "goto": {
          "when": "vars.prompt_value === '1'",
          "label": "welcome",
          "max": 3
        }
      },
      "hangup"
    ]
  }
}
```

### YAML

```yaml
version: 1.0.0
sections:
  main:
    - answer
    - label: welcome
      play:
        volume: 10.0
        urls:
          - https://example.com/welcome.mp3
    - prompt:
        - https://example.com/prompt.mp3
    - goto:
        when: vars.prompt_value === '1'
        label: welcome
        max: 3
    - hangup
```

### JSON - compact form (Expected on 9/4/2023)

```json
  [
      "answer",
      {
        "play": {
          "volume": 10.0,
          "urls": [
            "https://example.com/welcome.mp3",
          ]
        }
      },
      { "prompt": ["https://example.com/prompt.wav"] },
      "hangup"
    ]
```

### YAML - compact form (Expected on 9/4/2023)

```yaml
    - answer
    - play:
        volume: 10.0
        urls:
          - https://example.com/welcome.mp3
    - prompt:
        - https://example.com/prompt.mp3
    - hangup
```

## Implementing subroutines or integrations
The `execute` method provides a mechanism for calling subroutines. A subroutine is implemented as a section in the document that can optionally accept named parameters and return a result or can be hosted by a server.

The subroutine may optionally be implemented as an object with a `meta` and `code` field to add user-defined data and the code to execute, respectively.

### JSON subroutine in script
```json
{
  "sections": {
    "main": [
      {
        "execute": {
          "params": { "file": "https://example.com/foo.wav" },
          "meta": { "x": 100, "y": 100 },
          "dest": "my_subroutine"
        },
        "execute": {
          "params": { "file": "https://example.com/bar.wav" },
          "dest": "my_subroutine",
          "result": {
              "case": {
                "1": [ "hangup" ],
                "2": [ { "play": "say:You pressed two" } ]
            },
            "default": []
          }
        }
      }
    ],
    "my_subroutine": {
      "meta": { "foo": "bar" },
      "code": [
        { "play": "%{params.file}" },
        { "return": "1" }
      ]
    }
  }
}
```

### JSON subroutine hosted by server
Same as previous example, except the script will execute a URL that will reply with the following document:
```json
{
  "sections": {
    "main": [
      { "play": "%{params.file}" },
      { "return": "1" }
    ]
  }
}
```


## Executing from RELAY application example

The following example is a node.js application that listens on a relay application name `swml`, and then transfers the call to a relay ML script. The script may be defined in the request or may specify a URL that will return the document in response to an `HTTP POST` to it. You must map a phone number or domain app to the `swml` application name in the https://dev.swire.io dashboard.

```javascript
const { Voice } = require("@signalwire/realtime-api");

var script = `
version: 1.0.0
sections:
  main:
  - answer
  - play:
    - "silence:2"
    - "say:Please hold while we connect you"
  - connect:
      to: "+15551231234"
      from: "+15553214321"
      result:
        case:
          connected:
          - play: "say: Goodbye"
          - hangup
        default:
        - execute: voicemail

  voicemail:
  - play:
    - "say:Im sorry the person you are trying to reach is not available"
    - "say:Please leave a message after the tone"
  - record:
      beep: true
  - play: "say: Thank you. Goodbye."
  - hangup
`;

const client = new Voice.Client({
  project: "<your project token>",
  token: "<your project API key>",
  host: "relay.swire.io/api",
  contexts: ["swml"],
});

client.on("call.received", async (call) => {
  try {
    await client.execute({
      method: "calling.transfer",
      params: {
        node_id: call.nodeId,
        call_id: call.callId,
        dest: script,
      },
    });
  } catch (error) {}
});
```

---

## Section methods

### Method format

Methods have the format:

```
{
  "method_name": {
    "parameter_1": "parameter_1_value"
  },
  "label": "Optional label for goto"
}
```

And the shorthand formats:


```
  "method_name"
```

```
{ "method_name": "parameter_1_value" }
```

```
{ "method_name": [ "parameter_1_value", "parameter_2_value" ] }
```


The following methods may be executed in a section:

### Script control flow and variables
- [cond](#cond-when-then-else)
- [eval](#eval-varexpression) (DEPRECATED - use `set` instead)
- [execute](#execute-dest)
- [if](#if-condition-then-else)
- [transfer](#transfer-dest)
- [goto](#goto-label)
- [label](#label-label)
- [return](#return-value)
- [set](#set-varvalue)
- [switch](#switch-variable-case)
- [unset](#unset-var)

### External HTTP APIs
- [request](#request-url)
- [execute_rpc](#execute_rpc) (EXPERIMENTAL)

### Call control and bridging
- [answer](#answer-max_duration)
- [connect](#connect-)
- [dial](#dial) (EXPERIMENTAL)
- [hangup](#hangup-reason)
- [join_room](#join_room-name)
- [join_conference](#join_conference-name)
- [enter_queue](#enter_queue)
- [sip_refer](#sip_refer-to_uri)

### IVRs
- [bind_digit](#bind_digit) (EXPERIMENTAL)
- [clear_digit_bindings](#clear_digit_bindings) (EXPERIMENTAL)
- [detect_machine](#detect_machine)
- [echo](#echo-timeout)
- [pay](#pay)
- [prompt](#prompt-play)
- [play](#play-urls)
- [record](#record)
- [record_call](#record_call)
- [send_digits](#send_digits-digits)
- [sleep](#sleep-duration)
- [stop_record_call](#stop_record_callcontrol_id)

### Media processing
- [denoise](#denoise)
- [stop_denoise](#stop_denoise)
- [live_transcribe](#live_transcribe)
- [live_translate](#live_translate)
- [stream](#stream-url)
- [stop_stream](#stop_stream-control_id)
- [transcribe](#transcribe)
- [transcribe_stop](#transcribe_stop-control_id)

### AI
- [ai](#ai-agent)
- [amazon_bedrock](#amazon_bedrock)
- [stop_tap](#stop_tap-control_id)
- [tap](#tap-uri-direction)
- [user_event](#user_event) (EXPERIMENTAL)

### Faxing
- [receive_fax](#receive_fax)
- [send_fax](#send_fax-document)

### Messaging
- [send_sms](#send_sms)

### Unimplemented
- [log](#log-message) (NOT IMPLEMENTED)


---

### goto (label)
(prod 2024-01-29, DT stack 2024-02-02) Jump to a label in the current or parent section if condition is satisfied.

#### Parameters
- **label:** Required valid label in current or parent section to jump to.
- **when:** Optional JavaScript expression to evaluate. Default is true / unconditionally jump
- **max:** Optional maximum times to loop (1-100). Default is 100.
- **meta:** Optional user data, ignored by SignalWire. Default is not set.

#### Variables

None

#### Examples

##### Loop with max

```
"main": [
  {
    "label": "foo",
    "play": "say:This is some text that will be repeated 3 times."
  },
  {
    "goto": {
      "label": "foo",
      "max": 3
    }
  }
]
```

##### Loop if condition is satisfied

```
"main": [
  {
    "label": "foo",
    "play": "say:This is some text that will be repeated 3 times."
  },
  {
     "set": { "vars.do_loop": true }
  },
  {
    "goto": {
      "label": "foo",
      "when": "vars.do_loop === true",
      "max": 3
    }
  }
]
```

---

### label (label)
(prod 2024-01-29, DT stack 2024-02-02) Label a goto target in a section

#### Parameters
- **label:** Required label in section to jump to.

#### Variables

None

#### Examples

##### JSON label

```
"main": [
  { "label": "foo" },
  {
    "play": "say:This is some text that will be repeated 3 times."
  },
  {
    "goto": {
      "label": "foo",
      "max": 3
    }
  }
]
```

##### YAML label

```
main:
  - label: foo
  - play: "say:This is some text that will be repeated 3 times."
  - goto:
      label: "foo"
      max: 3
```


---

### log (message)
(NOT IMPLEMENTED) Log a message to the admin dashboard

#### Parameters
- **message:** Required message to log

#### Variables

None

#### Examples

##### JSON log all vars

```
{ "sections": {
    "main": [
      { "log": "%{vars}" },
      { "log": "%{call}" },
      { "log": "This is my debug log: call.to = %{call.to}" }
    ]
} }
```

##### YAML log all vars

```
sections:
  main:
  - log: "%{vars}"
  - log: "%{call}"
  - log: "This is my debug log: call.to = %{call.to}"
```

---

### transfer (dest)

Transfer execution of script to a new URL.

#### Parameters

- **dest:** Required `<URI>`
  - **`<label>`** - (DEPRECATED) section in document to jump to
  - **`relay:<relay application>`** - relay application to notify (NOT IMPLEMENTED)
  - **`https://<URL>`** - URL to fetch next document from. Sends HTTP POST
- **params:** Optional named parameters to send to section/URL/application. Default is not set.
- **meta:** Optional user data, ignored by SignalWire. Default is not set.

#### Variables

None

#### Examples

##### JSON, named parameter

```json
{ "transfer": { "dest": "https://example.com/next" } }
```

##### JSON, named parameter, with parameters

  ```json
{
   "transfer": {
      "dest": "https://example.com/next",
      "params": {
         "foo": "bar"
      }
   }
}
  ```

##### YAML, named parameter

```yaml
- transfer:
    dest: https://example.com/next
```

##### JSON, first parameter

```json
{ "transfer": "https://example.com/next" }
```

##### YAML, first parameter

```yaml
- transfer: https://example.com/next
```

---

### execute (dest)

Execute a section or URL as a subroutine and return back to current document.

#### Parameters

- **dest:** Required. It can be one of:
  - **`<label>`** - section in document to execute
  - **`https://<URL>`** - URL to fetch document to execute. Sends HTTP POST
- **params:** Optional named parameters to send to section/URL. Default is not set.
- **meta:** Optional user data, ignored by SignalWire. Default is not set. (CFB only: internal use; not costumer-facing)
- **result:** Optional. `switch` on `return_value` if `result` is an object (`{}`), or use as a `cond` if `result` is an array (`[]`)
- **on_return:** Optional. `Object` - a SWML document that takes over the call once the execute method is finished.

#### Variables

- **return_value:** (out) - Value returned by subroutine

#### Examples

##### JSON, named parameter

```json
{
   "execute": {
      "dest": "subroutine",
      "params": {
         "file": "https://example.com/foo.wav"
      }
   }
}
```

##### YAML, named parameter

```yaml
- execute:
    dest: subroutine
```

##### JSON, execute subroutine and optionally switch on return value
```json
{
  "label": "retry_menu",
  "execute": {
    "dest": "my_menu",
    "params": { "file": "https://example.com/foo.wav", "digits": "123" },
    "result": {
      "case": {
        "1": [
          { "execute": "https://example.com/sales.swml" }
        ],
        "2": [
          { "execute": "https://example.com/support.swml" }
        ],
        "3": [
          { "execute": "https://example.com/leave_a_message.swml" }
        ]
      },
      "default": [
        { "goto": { "label": "retry_menu", "max": 3 } }
      ]
    }
  }
}
```

##### JSON, execute subroutine and optionally branch on return value
```json
{
  "label": "retry_menu",
  "execute": {
    "dest": "my_menu",
    "params": { "file": "https://example.com/foo.wav", "digits": "123" },
    "result": [
      { "when": "vars.return_value == '1'", "then": [ { "execute": "https://example.com/sales.swml" } ] },
      { "when": "vars.return_value == '2'", "then": [ { "execute": "https://example.com/support.swml" } ] },
      { "when": "vars.return_value == '3'", "then": [ { "execute": "https://example.com/leave_a_message.swml" } ] },
      { "else": [ { "goto": { "label": "retry_menu", "max": 3 } } ] }
    ]
  }
}
```

---

### return (value)
Return from `execute` or exit script.

#### Parameters
 * Optional JSON parameter(s) of any type will be saved to `return_value`

#### Variables
 * **return_value:**(out) `Optional return value`

#### Examples

##### JSON, return with value

```json
{ "return": "1" }
 ```

##### YAML, return with value

 ```yaml
- return: 1
```

##### JSON, return with multiple values

```json
{ "return": { "a": "1", "b": "2" } }
 ```

##### YAML, return with multiple values

 ```yaml
- return:
    a: 1
    b: 2
```

##### JSON, return with no value

```json
 "return"
 ```

##### YAML, return with no value

 ```yaml
- return
```


---

### request (url)
GET/POST/PUT/DELETE a URL.

#### Parameters
 * **url:** Required `https://username:password@domain:port/path/file.html?key1=value1&key2=value2` - URL to send HTTP request to.
 * **method:** Required GET/POST/PUT/DELETE
 * **headers:** Optional object containing HTTP headers to set. Default are none. Allowed headers are `Accept`, `Authorization`, `Content-Type`, `Range`, and custom `X-` headers. Default is not set.
 * **body:** Optional HTTP body string or JSON object to serialize to string. Content-Type header should be explicitly set, but we'll take best guess based on first non-whitespace character in the body. { == application/json, < == application/xml, otherwise text/plain. Default is not set.
 * **timeout:** Optional maximum time in seconds to wait for response. Default is 5.0.
 * **connect_timeout:** Optional maximum time in seconds to wait for connection. Default is 5.0.
 - **result:** Optional `switch` on `return_value` when object `{}` or `cond` when array `[]` - **NOT IMPLEMENTED**
 * **save_variables:** Optional boolean - if true, the JSON response that is parsed will be stored as variables. In a future improvement, XML will be converted to a JSON object in as described below before saving to vars. Default is false.
   * JSON response example:
   ```
   { "foo": "bar", "foo2": "bar2", "foo3": { "foo4": "bar4" } }
   ```
   The variables `request_response.foo`, `request_response.foo2`, and `request_response.foo3.foo4` are set.
   * XML response example (NOT IMPLEMENTED)
   ```
   <Response>
     <foo>bar</foo>
     <foo2>bar2</foo2>
     <foo3>
       <foo4 att2="bar5">bar4</foo4>
     </foo3>
   </Response>
   ```
   The variables `request_response.Response.foo._text` and `request_response.Response.foo2._text`, `request_response.Response.foo3.foo4._text`, and `request_response.Response.foo3.foo4._attributes.att2` are set.
   * XML response example 2 (future)
   ```
   <foo>bar</foo>
   <foo2>bar2</foo>
   <foo3>
     <foo4>bar4</foo4>
   </foo3>
   ```
   The variables `request_response.foo._text` and `request_response.foo2._text`, are `request_response.foo3.foo4._text` are set.

#### Variables

- **request_url:** (out) URL POST was sent to
- **request_result:** (out) success/failed based on status code returned
- **request_response_code:** (out) HTTP response status code from the request
- **request_response_headers.<header name lowercase>:** (out) HTTP response headers w/ names all normalized to lowercase and values trimmed of whitespace. Maximum of 64 headers saved. Access `Content-Type` header value with `%{request_response_headers.content-type}` notation.
- **request_response_body:** (out) raw HTTP response body - limited to 64KB.
- **request_response.<object_field>:** (out) Variables saved from response if `save_variables` is true and parsed as JSON.
- **return_value:** (out) `request_result`

#### Examples

##### JSON

```json
{ 
  "request": {
    "url": "https://example.com/status_update",
    "method": "POST",
    "connect_timeout": 5.0,
    "timeout": 5.0,
    "save_variables": true,
    "headers": {
      "content-type": "application/json",
      "x-custom-header": "foo"
    },
    "body": {
      "payload": "%{call.to} finished the IVR."
    },
    "result": [
       { "when": "vars.request_response.foo != null", "then": [ { "play": "say:It worked!" } ] },
       { "else": [ { "play": "say:It didn't work!" } ] }
    ]
  }
}
```

##### YAML

 ```yaml
- request:
    url: https://example.com/status_update
    method: POST
    connect_timeout: 5.0
    timeout: 5.0
    save_variables: true
    headers:
      content-type: application/json
      x-custom-header: foo
    body:
      payload: "%{call.to} finished the IVR."
    result:
      - when: "vars.request_response.foo != null"
        then:
          - play: "say:It worked!"
      - else:
          - play: "say:It didn't work!"
 ```

---

### execute_rpc

(EXPERIMENTAL)

Execute a RELAY RPC method on the current call or another call.

#### Parameters

- **method:** String, Required. The RELAY method name to execute.
- **params:** Object, Optional. Parameters to pass to the method.
- **call_id:** String, Optional. Call ID to execute the method on. If not provided, executes methods that don't require a call context (e.g., `dial`).
- **node_id:** String, Optional. Node ID where the call resides. Required if `call_id` is provided.

#### Variables

- **last_api_code:** (out) HTTP-style result code
- **last_api_message:** (out) Result message
- **last_rpc_method:** (out) The RPC method that was executed
- **last_rpc_exec_time_ms:** (out) Execution time in milliseconds

#### Examples

##### JSON, Execute on current call

```json
{
  "execute_rpc": {
    "method": "calling.play",
    "call_id": "%{call.call_id}",
    "node_id": "%{call.node_id}",
    "params": {
      "media": [{ "type": "tts", "text": "Hello world" }]
    }
  }
}
```

---

### switch (variable, case)

Branch based on variable value.

#### Parameters

- **variable:** Required variable name.
- **case:** Optional object of values mapped to array of methods to execute. Default is not set.
- **default:** Optional array of methods to execute if no cases match. Default is not set.

#### Variables

None

#### Examples

##### JSON

```json
{
  "switch": {
    "variable": "prompt_value",
    "case": {
      "1": [{ "goto": "sales" }]
    },
    "default": [{ "play": "https://example.com/no_match.wav" }]
  }
}
```

##### YAML

```yaml
-switch:
  variable: prompt_value
  case:
    1:
    - goto: sales
  default:
  - play: "https://example.com/no_match.wav
```

---

### cond (when, then, else)
Branch based on JavaScript condition

#### Parameters
Array of objects:
 * **when:** Required JavaScript condition to test
 * **then:** Required array of methods to execute on true condition

or

 * **else:** Required array of methods to execute on no matching condition.

#### Variables

None

#### Examples

##### JSON

```json
{
  "cond": [
      {
        "when": "vars.foo < 2",
        "then": [
          "hangup"
        ]
      },
      {
        "when": "call.to == '+15551231234'",
        "then": [
          { "execute": "help" }
        ],
      },
      {
        "when": "/hello/.test(vars.foo)",
        "then": [
          { "execute": "match" }
        ]
      },
      {
        "else": [
          { "execute": "match" }
        ]
      }
    ]
}
```

##### YAML

```yaml
- cond:
  - when: "vars.foo > 3"
    then:
    - hangup
  - when: "vars.foo < 2"
    then:
    - hangup
  - when: "call.to == '+15551231234'"
    then:
    - execute: help
  - when: "/hello/.test(vars.foo)"
    then:
    - execute: match
  - else:
    - goto: retry
```

---

### if (condition, then, else)

Conditional execution with if/then/else structure.

#### Parameters

- **condition:** String, Required. JavaScript expression to evaluate.
- **then:** Array, Optional. Array of SWML methods to execute if condition is true.
- **else:** Array, Optional. Array of SWML methods to execute if condition is false.

#### Variables

None

#### Examples

##### JSON

```json
{
  "if": {
    "condition": "vars.foo > vars.bar",
    "then": [
      { "play": "say:Foo is greater than bar" }
    ],
    "else": [
      { "play": "say:Foo is not greater than bar" }
    ]
  }
}
```

##### YAML

```yaml
- if:
    condition: "vars.foo > vars.bar"
    then:
      - play: "say:Foo is greater than bar"
    else:
      - play: "say:Foo is not greater than bar"
```

---

### eval ({var=expression})

(DEPRECATED - use `set` instead)

Evaluate JavaScript expressions and store results in variables.

#### Parameters

Object where keys are variable names and values are JavaScript expressions to evaluate.

#### Variables

Sets the specified variables with the evaluated results.

#### Examples

##### JSON

```json
{
  "eval": {
    "foo": "vars.x > vars.y",
    "bar": "vars.count + 1",
    "result": "4 + 4"
  }
}
```

##### YAML

```yaml
- eval:
    foo: "vars.x > vars.y"
    bar: "vars.count + 1"
    result: "4 + 4"
```

---

### answer (max_duration)

Answer call and set optional maximum duration

#### Parameters

- **max_duration:** Optional maximum duration in seconds. Can not be less than 7 seconds. Default is 14400 seconds (4 hours).
- **codecs:** String, Optional. Comma-separated string of codecs to offer. Has no effect if a codec is not supported by the network. Default is picked by SignalWire. Valid codecs are: `PCMU,PCMA,G722,G729,AMR-WB,OPUS,VP8,H264`
- **username:** String, Optional. Username for SIP requests that require authentication.
- **password:** String, Optional. Password for SIP requests that require authentication.

#### Variables

None

#### Examples

##### JSON, no parameters

```json
"answer"
```

##### JSON, first parameter

```json
{ "answer": 60.0 }
```

##### JSON, named parameter

```json
{ "answer": { "max_duration": 60.0, "codecs": "PCMU,PCMA" } }
```

##### YAML, no parameters

```yaml
- answer
```

##### YAML, first parameter

```yaml
- answer: 60.0
```

##### YAML, named parameter

```yaml
- answer:
    max_duration: 60.0
```

---

### hangup (reason)

End the call with optional reason

#### Parameters

- **reason:** Optional hangup reason (hangup, busy, decline). Default is busy.

#### Variables

None

#### Examples

##### JSON, no parameters

```json
"hangup"
```

##### JSON, first parameter

```json
{ "hangup": "busy" }
```

##### JSON, named parameter

```json
{ "hangup": { "reason": "busy" } }
```

##### YAML, no parameters

```yaml
-hangup
```

##### YAML, first parameter

```yaml
-hangup: busy
```

##### YAML, named parameter

```yaml
 -hangup
    reason: busy
```

---

### pay

Make payment

#### Parameters

- **`input`** : optional (dtmf, voice) - the method of how to collect the payment details. Currently only dtmf mode is supported.
- **`status_url:`** optional - the URL to make requests to for each status change during the payment process.
- **`payment_method:`** optional - indicates the payment method which is going to be used in this payment request. Currently only credit-card is supported.
- **`timeout:`** optional - limit in seconds that pay IVR waits for the caller to press another digit before moving on to validate the digits captured. Default: 5 seconds
- **`max_attempts:`** optional - number of times pay IVR should retry when collecting payment card details details. Default: 1
- **`security_code:`** optional - takes true or false to let pay IVR know whether to prompt for security code. Default: true
- **`postal_code:`** optional - takes true, false or real postcode (if it's known beforehand) to let pay IVR know whether to prompt for postal code.
- **`min_postal_code_length:`** optional - callers are expected to enter at least these many digits when prompted for postal code by pay IVR. Default: 0
- **`payment_connector_url:`** required - the URL to make payment requests to upon completion of all required payment details.
- **`token_type:`** optional (one-time, reusable) - whether the payment is a one off payment or re-occuring. Default: reusable
- **`charge_amount:`** optional - the amount to charge against payment method passed in the request. Decimal value with no currency prefix passed as string.
- **`currency:`** optional - takes the currency of the charge amount. Default: usd
- **`language:`** optional - language to use for prompts being played to the caller by pay IVR. Default: en-US
- **`voice:`** optional - text-to-speech voice to use. Default: woman
- **`description:`** optional - custom description of the payment provided in the request.
- **`valid_card_types:`** optional (visa mastercard amex maestro discover jcb diners-club) - list of payment cards allowed to use in the requested payment process separated by space. Default: visa mastercard amex
- **`parameters:`** optional - array of additional name value pairs to POST to payment connector.
  - **`name:`** required - string - Any user-defined string.
  - **`value:`** required - string - Any user-defined string.
- **`prompts:`** optional - array of additional name value pairs to POST to payment connector.
  - **`for:`** required (payment-card-number, expiration-date, security-code, postal-code, payment-processing, payment-completed, payment-failed, payment-canceled) - indicates the situation to use this prompt for.
  - **`card_type:`** optional - list of card type separated by space to use this prompt for.
  - **`error_type:`** optional (timeout, invalid-card-number, invalid-card-type, invalid-date, invalid-security-code, invalid-postal-code, session-in-progress, card-declined) - list of error types separated by space to use this prompt for.
  - **`attempts:`** optional - string - Specifies which payment attempt this prompt applies to. Increments when payment fails. Use single numbers (e.g. `"1"`) or space-separated numbers (`"2 3"`) to apply prompt to the specified attempts.
  - **`actions:`** required - array of actions to execute.
    - **`type:`** required (Say, Play) - action to perform - Say for Text-to-Speech, Play for playing audio file.
    - **`phrase:`** required - sentence to Say or URL to Play.

#### Variables

- **`pay_result:`** (out) - success, too-many-failed-attempts, payment-connector-error, caller-interrupted-with-star, relay-pay-stop, caller-hung-up, validation-error, internal-error
- **`pay_payment_results:`** (out) - JSON contains:
  - **`payment_token:`**
  - **`payment_confirmation_code:`**
  - **`payment_card_number:`** - redacted
  - **`payment_card_type:`**
  - **`payment_card_expiration_date:`**
  - **`payment_card_security_code:`** - redacted
  - **`payment_card_postal_code:`**
  - **`payment_error:`**
  - **`payment_error_code:`**
  - **`connector_error:`**:
    - **`code:`**:
    - **`message:`**:

#### Examples

##### JSON

```json
{
    "sections": {
        "main": [
            "answer",
            "pay": {
                    "input": "dtmf",
                    "status_url": "http://127.0.0.1:5001/status-callback",
                    "payment_method": "credit-card",
                    "timeout": "6",
                    "max_attempts": "3",
                    "security_code": "true",
                    "postal_code": "false",
                    "min_postal_code_length": "6",
                    "payment_connector_url": "https://127.0.0.1:5000/accept-payment",
                    "token_type": "one-time",
                    "charge_amount": "10.55",
                    "currency": "usd",
                    "language": "en-US",
                    "voice": "polly.Sally",
                    "description": "bike",
                    "valid_card_types": "mastercard visa amex",
                    "parameters": [
                        {
                            "name": "reject_payment",
                            "value": "false"
                        },
                        {
                            "name": "customer_id",
                            "value": "93652a3c-1234-dcba-abcd-7980d413bv31"
                        }
                    ],
                    "prompts": [
                        {
                            "for": "payment-card-number",
                            "actions": [
                                {
                                    "type": "Say",
                                    "phrase": "Welcome to SignalWire telephony payment gateway!"
                                },
                                {
                                    "type": "Say",
                                    "phrase": "You're gonna pay your monthly subscription fee"
                                },
                                {
                                    "type": "Say",
                                    "phrase": "Please enter your credit card number"
                                }
                            ]
                        },
                        {
                            "for": "payment-card-number",
                            "error_type": "invalid-card-number timeout invalid-card-type",
                            "actions": [
                                {
                                    "type": "Say",
                                    "phrase": "You entered an invalid credit card number. Please try again."
                                }
                            ]
                        },
                        {
                            "for": "payment-completed",
                            "actions": [
                                {
                                    "type": "Say",
                                    "phrase": "Payment completed. You can continue to use our services."
                                }
                            ]
                        }
                    ]
            },
            "hangup"
        ]
    }
}
```

```yaml
sections:
  main:
  - label: payment_gateway
    pay:
      input: dtmf
      status_url: http://127.0.0.1:5001/status-callback
      payment_method: credit-card
      timeout: 6
      max_attempts: 3
      security_code: true
      postal_code: false
      min_postal_code_length: 6
      payment_connector_url: https://127.0.0.1:5000/accept-payment
      token_type: one-time
      charge_amount: 10.55
      currency: usd
      language: en-US
      voice: polly.Sally
      description: bike
      valid_card_types: mastercard visa amex
      parameters:
        - name: customer_id
          value: 93652a3c-1234-dcba-abcd-7980d413bv31
        - name: reject_payment
          value: false
      prompts:
        - for: payment-card-number
          actions:
            - type: Say
              phrase: Welcome to SignalWire telephony payment gateway!
            - type: Say
              phrase: You're gonna pay your monthly subscription fee
            - type: Say
              phrase: Please enter your credit card number
        - for: payment-completed
          actions:
            - type: Say
              phrase: Payment completed. You can continue to use our services.
```


---

### prompt (play...)

Play a prompt and wait for digit or speech input. Note:
 
- Speech detection is not enabled unless at least one speech parameter is set.
- If only speech parameters are set (and not `digit` ones), digit detection is not enabled.
- To enable both digit and speech, set at least one parameter for each.

#### Parameters

- **play:** Required, string array of URLs to play. Allowed URLs are:
  - **`http://`** - audio file to GET
  - **`https://`** - audio file to GET
  - **`ring:[duration:]<country code>`** - ring tone to play. For example: `ring:us` to play single ring or `ring:20.0:us` to play ring for 20 seconds.
  - **`say:<text to speak>`** - Sentence to say
  - **`silence:<duration>`** - seconds of silence to play
- **volume:** Number, \<-40.0 to 40.0\> Optional gain to apply to played URLs. Default 0.
- **say_voice:** Optional voice to use for text to speech. Default is Polly.Salli.
- **say_language:** Optional language to use for text to speech. Default is en-US.
- **say_gender:** Optional gender to use for text to speech. Default is female.
- **max_digits:** Integer, Optional number of digits to collect. Default 1.
- **terminators:** String, Optional digits that terminate digit collection. Default is not set.
- **digit_timeout:** Number, Optional time in seconds to wait for next digit. Default 5.0
- **initial_timeout:** Number, Optional time in seconds to wait for start of input. Default 5.0
- **speech_timeout:** Number, Optional max time in seconds to wait for speech result. Default is not set.
- **speech_end_timeout:** Number, Optional time in seconds to wait for end of speech utterance. Default is not set.
- **speech_language:** String, Optional language to detect speech in. Default is en-US.
- **speech_hints:** String array, Optional expected words to match. Default is not set.
- **speech_engine:** String, Optional forced speech recognition engine. The engine must support the specified language. [Deepgram|Google] Default is not set (we pick the engine).
- **result:** Optional `switch` on `return_value` when object `{}` or `cond` when array `[]` - **NOT IMPLEMENTED**
- **status_url:** String, Optional. http or https URL to deliver RELAY play and collect status events

#### Variables

- **prompt_result:** (out) - failed, no_input, match_speech, match_digits, no_match
- **prompt_value:** (out) - the digits or utterance collected
- **prompt_digit_terminator:** (out) - digit terminator collected, if any
- **prompt_speech_confidence:** (out) - speech confidence measured, if any
- **return_value:** (out) the `prompt_value`

- **say_voice:** (in) - optional voice to use for text to speech.
- **say_language:** (in) - optional language to use for text to speech.
- **say_gender:** (in) - optional gender to use for text to speech.

#### Examples

##### JSON, play prompt and wait for digit press

```json
{
  "version": "1.0.0",
  "sections": {
    "main": [
      { "label": "welcome",
        "prompt": {
           "https://example.com/menu.wav",
           "result": {
             "case": {
               "1": [ "transfer": "https://example.com/sales.swml" ]
             },
             "default": [
               { "play": "https://example.com/bad_input.wav" },
               { "goto": "welcome" }
             ]
           }
         }
      }
    ]
  }
}
```

#### YAML, play prompt and wait for digit press

```yaml
version: 1.0.0
sections:
  main:
    - label: welcome
      prompt: https://example.com/menu.wav
        result:
          case:
            1:
              - transfer: https://example.com/sales.swml
          default:
            - play: https://example.com/bad_input.wav
            - goto: welcome
```

##### JSON, play prompt and wait for digit or speech

```json
{
  "version": "1.0.0",
  "sections": {
    "main": [
      {
        "label": "welcome",
        "prompt": {
          "play": "https://example.com/press_or_say_one.wav",
          "speech_language": "en",
          "max_digits": 1,
          "hints": [
            "one",
            "two",
            "three",
            "four",
            "five",
            "six",
            "seven",
            "eight",
            "nine"
          ],
          "result": {
            "case": {
              "1": [{ "transfer": "https://example.com/sales.swml" }],
              "one": [{ "transfer": "https://example.com/sales.swml" }]
            },
            "default": [
              { "play": "https://example.com/bad_input.wav" },
              { "goto": { "label": "welcome", "max": 3 } }
            ]
          }
        }
      }
    ]
  }
}
```

##### YAML, play prompt and wait for digit or speech

```yaml
version: 1.0.0
sections:
  main:
  - label: welcome
    prompt:
      play: https://example.com/press_or_say_one.wav
      speech_language: en
      max_digits: 1
      hints:
      - one
      - two
      - three
      - four
      - five
      - six
      - seven
      - eight
      - nine
      result:
        variable: prompt_value
        case:
          1:
          - transfer: https://example.com/sales.swml
          one:
          - transfer: https://example.com/sales.swml
        default:
        - play: https://example.com/bad_input.wav
        - goto:
            label: welcome
            max: 3
```

##### YAML, play prompt and collect digit then post result for next action

```yaml
version: 1.0.0
sections:
  main:
    - prompt: https://example.com/menu.wav
    - transfer: https://example.com/post_next_menu
```

---

### play (urls)

Play file(s), ringtones, speech or silence

#### Parameters

- **`auto_answer` Optional - Boolean - If `true`, the call will automatically answer as the sound is playing. If `false`, you will start playing the audio during early media
- **`urls`** (or **`url`**): Required, URL or string array of URLs to play. Allowed URLs are:
  - **`http://`** - audio file to GET
  - **`https://`** - audio file to GET
  - **`ring:<duration>:<country code>`** - ring tone to play. For example: `ring:us` to play single ring or `ring:20.0:us` to play ring for 20 seconds.
  - **`say:<text to speak>`** - Sentence to say
  - **`silence:<duration>`** - seconds of silence to play
- **`volume:`** Number, Optional \<-40.0 to 40.0\> optional gain to apply to URLs. Default 0.
- **`loop:`** Number, Optional number of times to play the file(s). Default 1. 0 will loop until the call ends or the play is stopped.
- **say_voice:** Optional voice to use for text to speech. Default is Polly.Salli.
- **say_language:** optional language to use for text to speech. Default is en-US.
- **say_gender:** optional gender to use for text to speech. Default is female.
- **status_url:** String, Optional. http or https URL to deliver RELAY play status events

#### Variables

- **`say_voice:`** (in) - optional voice to use for `say:` URLs
- **`say_language:`** (in) - optional language to use for `say:` URLs
- **`say_gender:`** (in) - optional gender to use for `say:` URLs

#### Examples

##### JSON, play single URL

```json
{ "play": "https://example.com/file.mp3" }
```

##### YAML, play single URL

```yaml
-play: https://example.com/file.mp3
```

##### JSON, play multiple URLs

```json
{
  "play": [
    "https://example.com/file1.wav",
    "https://example.com/file2.wav",
    "say: this is something to say",
    "silence:3.0",
    "ring:10.0:us"
  ]
}
```

##### YAML, play multiple URLs

```yaml
- play:
    - https://example.com/file1.wav
    - https://example.com/file2.wav
    - say:this is something to say
    - silence:3.0
    - ring:10.0:us
```

##### JSON, play multiple URLs with volume adjustment

```json
{
  "play": {
    "volume": 30,
    "urls": [
      "https://example.com/file1.wav",
      "https://example.com/file2.wav",
      "say:this is something to say",
      "silence:3.0",
      "ring:10.0:us"
    ]
  }
}
```

##### YAML, play multiple URLs with volume adjustment

```yaml
- play:
    volume: 30
    urls:
      - https://example.com/file1.wav
      - https://example.com/file2.wav
      - say:this is something to say
      - silence:3.0
      - ring:10.0:us
```

---

### bind_digit

(EXPERIMENTAL)

Bind a DTMF digit sequence to trigger a SWML method during call execution.

#### Parameters

- **digits:** String, Required. The digit sequence to bind (e.g., "1", "123", "*1").
- **method:** String, Required. The SWML method name to execute when digits are pressed.
- **params:** Object, Optional. Parameters to pass to the method when triggered.
- **realm:** String, Optional. Binding realm/namespace. Default is "default".
- **max_triggers:** Integer, Optional. Maximum number of times this binding can trigger. 0 = unlimited. Default is 0.

#### Variables

- **bind_digit_result:** (out) `success` | `failed`

#### Examples

##### JSON

```json
{
  "bind_digit": {
    "digits": "1",
    "method": "transfer",
    "params": {
      "dest": "https://example.com/support.json"
    }
  }
}
```

##### YAML

```yaml
- bind_digit:
    digits: "1"
    method: transfer
    params:
      dest: "https://example.com/support.json"
```

---

### clear_digit_bindings

(EXPERIMENTAL)

Clear all digit bindings in a realm.

#### Parameters

- **realm:** String, Optional. The realm/namespace to clear bindings from. Default is "default".

#### Variables

- **clear_digit_bindings_result:** (out) `success` | `failed`
- **digit_bindings_cleared:** (out) Number of bindings cleared

#### Examples

##### JSON

```json
{ "clear_digit_bindings": {} }
```

##### JSON, Clear specific realm

```json
{ "clear_digit_bindings": { "realm": "my_realm" } }
```

---

### detect_machine

Detect answering machine or fax machine. This method will return when:
  * `async` is true - use `status_url` to collect detector events asynchronously
  * The timeout has expired
  * when `amd` is enabled
    * `detect_message_end` is false and a machine is detected
    * `detect_message_end` is true and a machine has completed its greeting
    * human is detected
  * when `fax` is enabled
    * fax tone is detected

#### Parameters
- **wait**: Optional boolean. If false, the detector is run asynchronously and `status_url` must be set. Default true.
- **status_url**: Optional http(s) URL to deliver detector events to
- **timeout**: Optional - sec >= 0, optional max time to run detector. Default 30.0 seconds.
- **detectors**: Optional, comma-separated string: detectors to enable. Default: `amd,fax`
- **initial_timeout:** - Optional, sec > 0, how long to wait for initial voice before giving up. Default 4.5.
- **end_silence_timeout:** - Optional, sec > 0, how long to wait for voice to finish. Default 1.0.
- **detect_message_end:** Optional boolean - If true, stops detection on beep / end of voicemail greeting. Default false.
- **machine_ready_timeout:** - Optional, sec > 0, how long to wait for voice to finish before firing READY event. Default is end_silence_timeout.
- **machine_voice_threshold:** - Optional, sec > 0, how much voice in ms to decide MACHINE. Default 1.25.
- **machine_words_threshold:** - Optional, count > 0, how many words to count to decide MACHINE. Default 6.
- **tone:** Optional, CED|CNG, The tone to detect, will only receive remote side tone. Default "CED"


#### Variables

- **`detect_result:`** (out) machine, human, unknown, fax
- **`return_value:`** (out) `detect_result`
- **`detect_machine_beep:`** (out) true if a beep was heard
- **`detect_ms:`** (out) duration detector ran before making a decision

#### Examples

##### JSON, deliver message to human or machine. Hangup on fax.

```json
{
  "sections": {
    "main": [
      {
        "detect_machine": {
          "detect_message_end": true,
          "result": {
            "case": {
              "fax": "hangup",
              "machine": [ { "play": "http://example.com/machine_message.mp3" } ]
            }
          }
        }
      },
      "hangup"
    ]
  }
}
```

##### JSON, detect and bridge call to agent

```json
{
  "sections": {
    "main": [
      {
        "detect_machine": {
          "result": {
            "case": {
              "fax": "hangup",
              "machine": [ { "play": "http://example.com/machine_message.mp3" } ]
            }
          }
        }
      },
      { "connect": { "from": "+15553214321", "to": "+15551231234" } },
      "hangup"
    ]
  }
}

```

##### JSON, start async detector and execute script

```json
{
  "sections": {
    "main": [
      {
        "detect_machine": {
          "async": true,
          "status_url": "https://example.com/on_detector_event"
        }
      },
      {
          "play": "https://example.com/something_to_say.mp3"
      },
      "hangup"
    ]
  }
}

```

---

### echo (timeout)

Echo audio back to the caller. Useful for testing audio quality, latency measurement, and debugging audio paths.

#### Parameters

- **timeout:** Integer, Optional. Duration in seconds to run the echo. Default 60.

#### Variables

- **echo_result:** (out) success | failed

#### Examples

##### JSON, echo for 30 seconds

```json
{ "echo": { "timeout": 30 } }
```

##### YAML, echo with default timeout

```yaml
- echo
```

##### YAML, echo for 10 seconds

```yaml
- echo:
    timeout: 10
```

---

### set ({var=value})

Set script variables.

#### Parameters

No specific parameters. Accepts an object mapping variable names to values.

#### Variables

Any variable can be set by this method.

#### Examples

##### JSON, set multiple variables

```json
{
  "set": {
    "vars.variable1": "value1",
    "variable2": "%{value2 + 1}",
    "bool_var": true,
    "num_var": 1,
    "float_var": 10.3,
    "obj_var": { "foo": "bar" }
  }
}
```

##### YAML, set multiple variables

```yaml
- set:
    vars.variable1: value1
    variable2: "%{value2 + 1}"
    bool_var: true
    num_var: 1
    float_var: 10.3
    obj_var:
      foo: bar
```

---

### unset ([var...])

Unset script variables

#### Parameters

String array of variable names

#### Variable

Any variable can be unset by this method

#### Examples

##### JSON, unset single variable

```json
{ "unset": "variable1" }
```

##### YAML unset single variable

```yaml
- unset: variable1
```

##### JSON, unset multiple variables

```json
{ "unset": ["variable1", "variable2"] }
```

##### YAML, unset multiple variables

```yaml
- unset:
    - variable1
    - variable2
```

---

### record

Record call audio in foreground. This is recording for voicemails.

#### Parameters

- **stereo:** Boolean, Optional. Default false.
- **format:** String, Optional. wav or mp3. Default wav
- **direction:** String, Optional. Direction of audio to record. speak = what party says. hear = what party hears. Default = speak.
- **terminators:** String, Optional. Digits that stop recording. Default = #
- **beep:** Boolean, Optional. Play a beep before recording. Default = false
- **input_sensitivity:** Number (0.0 - 100.0), Optional. How sensitive the recording voice activity detector is to background noise. Large value is more sensitive. Default 44.0
- **initial_timeout:** Number, Optional. How long to wait for speech to start in seconds. Default 4.0.
- **end_silence_timeout:** Number, optional. How much silence in seconds to end recording. Default 5.0.
- **max_length:** Number, Optional. Maximum length of the recording, in seconds. Default is 0 which means unlimited recording length.
- **status_url:** String, Optional. http or https URL to deliver RELAY recording status events


#### Variables

- **record_url:** (out) the URL of the newly created recording
- **record_result:** (out) success | failed

#### Examples

##### JSON, record then play it back

```json
{ "record": { "end_silence_timeout": 5.0 } },
{ "play": "%{record_url}" }
```

##### YAML, record then play it back

```yaml
- record:
    end_silence_timeout: 5.0
- play: "%{record_url}"
```

---

### record_call

Record call in the background.

#### Parameters

- **control_id:** String, Optional identifier for this recording to use with `stop_call_record`. Default is generated and saved to record_control_id variable.
- **stereo:** Boolean, Optional. Default false
- **format:** String, Optional. wav or mp3. Default wav
- **direction:** String, Optional. Direction of audio to record. speak = what party says. hear = what party hears. both = what party hears and says. Default = both.
- **terminators:** String, Optional. Digits that stop recording. Default is not set.
- **beep:** Boolean, Optional. Play a beep before recording. Default = false
- **input_sensitivity:** Number (0.0 - 100.0), Optional. How sensitive the recording voice activity detector is to background noise. Large value is more sensitive. Default 44.0
- **initial_timeout:** Number, Optional. How long to wait for speech to start in seconds. Default 0.
- **end_silence_timeout:** Number, Optional. How much silence in seconds to end recording. Default 0.
- **max_length**: Number, Optional. Maximum length of the recording, in seconds. Default is 0 which means unlimited recording length.
- **status_url:** String, Optional. http or https URL to deliver RELAY recording status events

#### Variables

- **record_call_url:** (out) the URL of the newly started recording
- **record_call_result:** (out) success | failed
- **record_control_id:** (out) control ID of this recording

#### Examples

##### JSON, start mp3 call recording

```json
{ "record_call": { "format": "mp3" } }
```

##### YAML, start mp3 call recording

```yaml
- record_call:
    format: mp3
```

---

### stop_record_call(control_id)

Stop an active background recording

#### Parameters

- **control_id:** String, Optional ID of the recording to stop. If not set, the last recording started will be stopped.

#### Variables

- **record_control_id:** (in) control ID of last recording started
- **stop_record_call_result:** (out) success | failed

#### Examples

##### JSON, stop last call recording

```json
"stop_record_call"
```

##### YAML, stop last call recording

```yaml
- stop_record_call
```

##### JSON, stop a specific call recording

```json
{ "stop_record_call": "my-recording-id" }
```

##### JSON, stop a specific call recording

```yaml
- stop_record_call: my-recording-id
```

---

### join_room (name)

Join a RELAY room

#### Parameters

- **name**: String, Required name of room
- **result:** Optional `switch` on `return_value` when object `{}` or `cond` when array `[]` - **NOT IMPLEMENTED**

#### Variables

- **join_room_result:** (out) joined | failed
- **return_value:** (out) `join_room_result`

#### Examples

##### JSON, join a room

```json
{ "join_room": "my_room" }
```

##### YAML, join a room

```yaml
- join_room: my_room
```

##### JSON, join a room and retry on failure

```json
"main": [
  { "label": "retry_join",
    "join_room": {
      "name": "my_room",
      "result": {
        "case": {
          "joined": [ "hangup" ]
        },
        "default": [
          { "goto": { "label": "retry_join", "max": 2 } }
        ]
      }
    }
  },
  "hangup"
]
```

##### YAML, join a room and retry on failure

```yaml
main:
  - label: retry_join
    join_room:
      name: my_room
      result:
        case:
          joined:
          - hangup
        default:
        - goto:
            label: retry_join
            max: 2
- hangup
```

---

### join_conference (name)

Join an ad-hoc audio conference with RELAY and CXML calls.

#### Parameters

- **name**: String, Required name of conference
- **muted**: Optional boolean, default `false`
- **beep**: Optional String [`true`, `false`, `onEnter`, `onExit`], default `true`
- **start_on_enter**: Optional boolean, default `true`
- **end_on_exit**: Optional boolean, default `false`
- **wait_url**: Optional SWML URL. default hold music
- **max_participants**: Optional positive integer <= 250. Default 250 (TBD check SignalWire docs)
- **record**: Optional string [`do-not-record`, `record-from-start`]. Default `do-not-record`
- **region**: Optional string. Default not set
- **trim**: Optional string [`trim-silence`, `do-not-trim`]. Default `trim-silence`
- **coach**: Optional string [ SWML Call ID, CXML CallSid ]. Default not set
- **status_callback_event**: Optional string [ `start` `end` `join` `leave` `mute` `hold` `modify` `speaker` `announcement` ]. Default not set
- **status_callback**: Optional URL. Default not set
- **status_callback_method**: Optional string [ `GET`, `POST` ]. Default `POST`
- **recording_status_callback**: Optional URL. Default not set
- **recording_status_callback_method**: Optional string [ `GET`, `POST` ]. Default `POST`.
- **recording_status_callback_event**: Optional string [ `in-progress` `completed` `absent` ]. Default `completed`
- **stream**: Optional object. Stream conference audio to a WebSocket endpoint. See [stream](#stream-url) for parameter details.
- **result**: Optional `switch` on `return_value` when object `{}` or `cond` when array `[]`

#### Variables

- **join_conference_result:** (out) `completed` | `answered` | `no-answer` | `failed` | `canceled`
- **return_value:** (out) `join_conference_result`

#### Examples

##### JSON, join a conference

```json
{ "join_conference": "my_conference" }
```

##### YAML, join a conference

```yaml
- join_conference: my_conference
```

---

### enter_queue

Place the call in queue

#### Parameters

- **queue_name**: String, Required name of queue
- **transfer_after_bridge**: String, Required. SWML URL or inline SWML to execute when the bridge ends. Used to transfer either call leg after the queue call completes.
- **status_url**: String, Optional. HTTP or HTTPS URL to deliver Queue status events.
- **wait_url**: String, Optional. SWML URL for hold music/announcements. Default is hold music.
- **wait_time**: Number, Optional. Wait in the queue for this many seconds before timeout. Default: 3600s.
- **result**: Optional `switch` on `return_value` when object `{}` or `cond` when array `[]`

#### Variables

- **queue_result:** (out) `connected` | `timeout` | `hangup` | `failed`
- **return_value:** (out) queue_result

#### Examples

##### JSON, place a call in queue

```json
{ "queue": {
    "queue_name": "my_queue"
  }
}
```

##### YAML, place a call in queue

```yaml
- queue:
    queue_name: "my_queue"
```

---


### denoise()

Start noise reduction

#### Parameters

None

#### Variables

- **denoise_result:** (out) on|failed

#### Examples

##### JSON, start denoise

```json
"denoise"
```

##### YAML, start denoise

```yaml
- denoise
```

---

### stop_denoise()

Stop noise reduction

#### Parameters

None

#### Variables

- **denoise_result:** (out) off

#### Examples

##### JSON, stop denoise

```json
"stop_denoise"
```

##### YAML, stop denoise

```yaml
- stop_denoise
```

---

### live_transcribe

Start live transcription of call audio.

#### Parameters

Parameters are passed through to the RELAY live_transcribe method. See RELAY documentation for available parameters.

#### Variables

None

#### Examples

##### JSON

```json
{
  "live_transcribe": {
    "language": "en-US"
  }
}
```

---

### live_translate

Start live translation of call audio.

#### Parameters

Parameters are passed through to the RELAY live_translate method. See RELAY documentation for available parameters.

#### Variables

None

#### Examples

##### JSON

```json
{
  "live_translate": {
    "source_language": "en-US",
    "target_language": "es-ES"
  }
}
```

---

### stream (url)

Start streaming call audio to a WebSocket endpoint. Similar to `tap` but specifically for WebSocket streaming with additional options for authentication and custom parameters.

#### Parameters

- **url:** String, Required. WebSocket URL to stream audio to. Must be `wss://` protocol.
- **control_id:** String, Optional. Identifier for this stream to use with `stop_stream`. Default is generated and stored in `stream_control_id` variable.
- **name:** String, Optional. Name identifier for this stream.
- **track:** String, Optional. Which audio track to stream. One of: `inbound_track`, `outbound_track`, `both_tracks`. Default `inbound_track`.
- **codec:** String, Optional. Audio codec for the stream.
- **status_url:** String, Optional. URL to receive status callbacks for stream events.
- **status_url_method:** String, Optional. HTTP method for status callbacks.
- **authorization_bearer_token:** String, Optional. Bearer token for WebSocket authentication.
- **custom_parameters:** Object, Optional. Custom key-value parameters to include in the WebSocket connection.

#### Variables

- **stream_control_id:** (out) control ID of this stream
- **stream_result:** (out) success | failed

#### Examples

##### JSON, start WebSocket stream

```json
{
  "stream": {
    "url": "wss://example.com/audio-stream",
    "track": "both_tracks"
  }
}
```

##### YAML, stream with authentication

```yaml
- stream:
    url: wss://example.com/audio-stream
    authorization_bearer_token: "my-secret-token"
    custom_parameters:
      session_id: "12345"
      user_id: "user-abc"
```

##### YAML, simple stream

```yaml
- stream: wss://example.com/audio-stream
```

---

### stop_stream (control_id)

Stop an active audio stream.

#### Parameters

- **control_id:** String, Optional. ID of the stream to stop. If not set, the last stream started will be stopped.

#### Variables

- **stream_control_id:** (in) control ID of last stream started
- **stop_stream_result:** (out) success | failed

#### Examples

##### JSON, stop last stream

```json
"stop_stream"
```

##### YAML, stop last stream

```yaml
- stop_stream
```

##### JSON, stop a specific stream

```json
{ "stop_stream": "my-stream-id" }
```

##### YAML, stop a specific stream

```yaml
- stop_stream: my-stream-id
```

---

### transcribe

Start transcription of call audio. Transcription results are delivered via `status_url` webhook.

#### Parameters

- **control_id:** String, Optional. Identifier for this transcription to use with `transcribe_stop`. Default is generated and stored in `transcribe_control_id` variable.
- **status_url:** String, Optional. HTTP or HTTPS URL to deliver transcription events.

#### Variables

- **transcribe_control_id:** (out) control ID of this transcription
- **transcribe_result:** (out) success | failed

#### Examples

##### JSON, start transcription with status callback

```json
{
  "transcribe": {
    "status_url": "https://example.com/transcription-events"
  }
}
```

##### YAML, start transcription

```yaml
- transcribe:
    status_url: https://example.com/transcription-events
```

---

### transcribe_stop (control_id)

Stop an active transcription.

#### Parameters

- **control_id:** String, Optional. ID of the transcription to stop. If not set, the last transcription started will be stopped.

#### Variables

- **transcribe_control_id:** (in) control ID of last transcription started
- **transcribe_stop_result:** (out) success | failed

#### Examples

##### JSON, stop last transcription

```json
"transcribe_stop"
```

##### YAML, stop last transcription

```yaml
- transcribe_stop
```

##### JSON, stop a specific transcription

```json
{ "transcribe_stop": "my-transcription-id" }
```

##### YAML, stop a specific transcription

```yaml
- transcribe_stop: my-transcription-id
```

---

### receive_fax

Receive a fax being delivered to this call

#### Parameters

- **status_url:** String, Optional. http or https URL to deliver RELAY fax status events

#### Variables

- **receive_fax_document:** (out) URL of received document
- **receive_fax_identity:** (out) identity of this fax station
- **receive_fax_remote_identity:** (out) identity of the sending fax station
- **receive_fax_pages:** (out) number of pages received
- **receive_fax_result_code:** (out) fax status code
- **receive_fax_result_text:** (out) description of fax status code
- **receive_fax_result:** (out) success | failed

#### Examples

##### JSON, receive a fax and post a result to webhook

```json
{
  "main": ["receive_fax", { "execute": "https://example.com/handle_incoming_fax" }]
}
```

##### YAML, receive a fax and post a result to webhook

```yaml
- receive_fax
- execute: https://example.com/handle_incoming_fax
```

---

### send_fax (document)

Send a fax

#### Parameters

- **document:** String, Required URL of PDF document to fax
- **header_info:** String, Optional text to add to fax header. Default is not set.
- **identity:** String, Optional station identity to report. Default is caller ID number.
- **status_url:** String, Optional. http or https URL to deliver RELAY fax status events

#### Variables

- **send_fax_document:** (out) URL of sent document
- **send_fax_identity:** (out) identity of this fax station
- **send_fax_remote_identity:** (out) identity of the receiving fax station
- **send_fax_pages:** (out) number of pages sent
- **send_fax_result_code:** (out) fax status code
- **send_fax_result_text:** (out) description of fax status code
- **send_fax_result:** (out) success | failed

#### Examples

##### JSON, send a fax and post a result to webhook

```json
{
  "main": [
    { "send_fax": "https//example.com/fax_to_send.pdf" },
    { "execute": "https://example.com/handle_outgoing_fax_result" }
  ]
}
```

##### YAML, receive a fax and post a result to webhook

```yaml
- send_fax: https://example.com/fax_to_send.pdf
- execute: https://example.com/handle_outgoing_fax_result
```

---

### sip_refer (to_uri)

Send SIP REFER to SIP call

#### Parameters

- **to_uri:** Required SIP URI to REFER to (instead of `to_uri` it's possible to use `to`. At least one of `to_uri` or `to` must be provided)
- **result:** Optional `switch` on `return_value` when object `{}` or `cond` when array `[]` - **NOT IMPLEMENTED**
- **username:** String, Optional. Username to use if REFER request requires authentication.
- **password:** String, Optional. Password to use if REFER request requires authentication.
- **status_url:** String, Optional. http or https URL to deliver RELAY refer status events

#### Variables

- **sip_refer_to:** (out) The SIP URI the recipient is to INVITE
- **sip_refer_result:** (out) Overall SIP REFER result
- **sip_refer_response_code:** (out) recipient response to the REFER request
- **sip_refer_to_response_code:** (out) INVITE response to the recipient
- **return_value:** (out) `sip_refer_result`

#### Examples

##### Send SIP REFER and post result

```json
{
  "sections": {
    "main": [
      {
        "label": "retry_refer",
        "sip_refer": {
          "to_uri": "sip:alice@example.com",
          "result": {
            "case": {
              "success": [ ]
            },
            "default": [
              { "goto": { "label": "retry_refer", "max": 2 } }
            ]
          }
        }
      },
      { "execute": "https://example.com/handle_sip_refer_result" },
      "hangup"
    ]
  }
}
```

##### YAML, Send SIP REFER and post result

```yaml
sections:
  main:
  - label: retry_refer
    sip_refer:
      to_uri: sip:alice@example.com
      result:
        - when: "vars.return_value != 'success'"
          then:
          - goto:
              label: retry_refer
              max: 2
  - execute: https://example.com/handle_sip_refer_result
```

---

### connect ()

Dial SIP URI or phone number

#### Destination

A destination is the object to dial with:

- **to:** String, Required. Phone number, SIP URI, queue, or stream to dial. Use `queue:queue_name` to connect to the first caller in a queue. Use `stream:wss://...` to connect to a bidirectional WebSocket stream.
- **from:** String, Optional. Caller ID number. Default is calling party caller ID number.
- **from_name:** String, Optional. Caller ID name for SIP calls. Has no effect on calls to phone numbers. Default is not set.
- **encryption:** String, Optional. Media encryption (SRTP). Possible values: `mandatory`, `optional`, `forbidden`. Has no effect on calls to phone numbers.
- **username:** String, Optional. Username for SIP authentication. Has no effect on calls to phone numbers. Default is not set.
- **password:** String, Optional. Password for SIP authentication. Has no effect on calls to phone numbers. Default is not set.
- **timeout:** Integer, Optional. Seconds. Maximum time to wait for destination to answer. Default is 60.
- **call_state_url:** String, Optional - webhook url (http(s)) to which send call state change notifications
- **call_state_events:** String, Optional - an array of call state event names to be notified about. Allowed event names: `created`, `ringing`, `answered`, `ended`. Defaults to `["ended"]`
- **confirm:** SWML URL or inline compact SWML script, Optional - script to execute on destination when answered. Parent and destination are bridged after script completes execution. If more than one destination is dialed in parallel, the destination that completes the script first wins and is bridged to the parent.
- **confirm_timeout:** Integer, Optional. The amount of seconds it takes for `confirm` to successfully execute. 
- **status_url:** String, Optional. http or https URL to deliver RELAY connect status events

##### Stream-specific parameters (when using `stream:` prefix)

- **name:** String, Optional. Stream name identifier.
- **codec:** String, Optional. Audio codec. Default is PCMU. Supported: PCMU, PCMA, G722, L16. Codec can include rate/ptime modifiers (e.g., `PCMU@40i`, `L16@24000h@40i`).
- **status_url:** String, Optional. Webhook URL (https://) for stream status notifications.
- **status_url_method:** String, Optional. HTTP method for status webhook (GET or POST). Default is POST.
- **realtime:** Boolean, Optional. Enable realtime mode for bidirectional audio. Default is false.
- **authorization_bearer_token:** String, Optional. Bearer token sent as Authorization header in WebSocket handshake.
- **custom_parameters:** Object, Optional. Custom key-value pairs sent in WebSocket start message.

#### Parameters

- **from:** String, Optional. Caller ID number. Default is calling party caller ID number. Can be overwritten on each [destination](#destination).
- **from_name:** String, Optional. Caller ID name for SIP calls. Has no effect on calls to phone numbers. Default is not set. Can be overwritten on each [destination](#destination).
- **headers:** Object, Optional. Custom SIP headers to add to INVITE. Has no effect on calls to phone numbers. Default is not set.
- **codecs:** String, Optional. Comma-separated string of codecs to offer. Has no effect on calls to phone numbers. Default is picked by SignalWire.
- **webrtc_media:** Boolean, Optional. If true, WebRTC media is offered to the SIP endpoint. Has no effect on calls to phone numbers. Default is false.
- **encryption:** String, Optional. Media encryption (SRTP). Possible values: `mandatory`, `optional`, `forbidden`. Has no effect on calls to phone numbers. Can be overwritten on each [destination](#destination).
- **session_timeout:** Integer, Optional. Seconds. If >0, SIP `Session-Expires` header in INVITE is set to this value. Has no effect on calls to phone numbers. Default is picked by SignalWire.
- **timeout:** Integer, Optional. Seconds. Maximum time to wait for destination to answer. Default is 60.
- **call_state_url:** String, Optional - webhook url (http(s)) to which send call state change notifications for all legs (in case of parallel/serial). Can be overwritten on each [destination](#destination).
- **call_state_events:** String, Optional - an array of call state event names to be notified about. Allowed event names: `created`, `ringing`, `answered`, `ended`. Defaults to `["ended"]`. Can be overwritten on each [destination](#destination).
- - **confirm:** (Expected 9/4/2023) SWML URL or inline compact SWML script, Optional - script to execute on destination when answered. Parent and destination are bridged after script completes execution. If more than one destination is dialed in parallel, the destination that completes the script first wins and is bridged to the parent. Can be overwritten on each [destination](#destination)
- **ringback:** Array of `play` URIs to play as ringback tone. Default will play audio from the provider.
- **max_duration:** Integer, Optional. Maximum duration in seconds.
- **answer_on_bridge:** Delay answer until the b-leg answers. Default is false
- one of the following dial algorithms
  - **serial_parallel:** Array of arrays. Inner arrays contain [destinations](#destination) to dial simultaneously. Outer array attempts each parallel group in order.
  - **serial:** Array of [destinations](#destination) to dial in order.
  - **parallel:** Array of [destinations](#destination) to dial simultaneously.
  - **to:** Single destination to dial.
- **result:** Optional `switch` on `return_value` when object `{}` or `cond` when array `[]`

#### Variables

- **connect_result:** (out) connected | failed
- **connect_failed_reason:** (out) detailed reason for failure
- **return_value:** (out) `connect_result`

#### Allowed SWML methods in confirm (Expected 9/4/2023) 
The following subset of SWML methods are allowed in confirm:
  - cond
  - if
  - switch
  - set
  - unset
  - hangup
  - play
  - prompt
  - record
  - record_call
  - stop_record_call
  - tap
  - stop_tap
  - send_digits
  - send_sms
  - denoise
  - stop_denoise

#### Examples

##### JSON, Dial single phone number

```json
{ "connect": { "from": "+15553214321", "to": "+15551231234" } }
```

##### YAML, Dial single phone number

```yaml
- connect:
    to: "+15551231234"
```

##### JSON, Dial single phone number, go to voicemail on failure

```json
{ "connect": {
    "from": "+15553214321",
    "to": "+15551231234",
    "result": {
      "case": {
        "connected": [ "hangup" ]
      },
      "default": [
        { "execute": "voicemail" },
        "hangup"
      ]
    }
  }
}
```

##### YAML, Dial single phone number, go to voicemail on failure

```yaml
- connect:
    from: "+15553214321"
    to: "+15551231234"
    result:
      case:
        connected:
        - hangup
      default:
      - execute: "voicemail"
      - hangup
```


##### JSON, Dial numbers in parallel, one number with shorter timeout than default 60 seconds

```json
{
  "connect": {
    "parallel": [{ "to": "+15551231234", "timeout": 20 }, { "to": "+15553214321" }]
  }
}
```

##### YAML, Dial numbers in parallel, one number with shorter timeout than default 60 seconds

```yaml
-connect:
  parallel:
    - to: +15551231234
      timeout: 20
    - to: +15553214321
```

##### JSON, Dial SIP serially, with timeout of 20 seconds applied to all numbers

```json
{
  "connect": {
    "timeout": 20,
    "serial": [
      { "from": "sip:chris@example.com", "to": "sip:alice@example.com" },
      { "codecs": "PCMU", "to": "sip:bob@example.com" }
    ]
  }
}
```

##### YAML, Dial SIP serially, with timeout of 20 seconds applied to all numbers

```yaml
- connect:
    timeout: 20
    serial:
      - from: "sip:chris@example.com"
        to: "sip:alice@example.com"
      - to: "sip:bob@example.com"
        codecs: PCMU
```

##### JSON, Dial numbers in parallel. Called party must press one to accept the call.

```json
{
  "connect": {
    "timeout": 20,
    "confirm": [
      {
        "prompt": "say:Press one to accept this call."
      },
      {
        "cond": [
          { "when": "%{var.prompt_value != '1'}", "then": [ "hangup" ] },
          { "else": [ { "play": "say:Thank you" } ] }
        ]
      }
    ],
    "parallel": [{ "to": "+15551231234" }, { "to": "+15553214321" }]
  }
}
```

##### JSON, Connect to first caller in queue

```json
{ "connect": { "to": "queue:my_queue" } }
```

##### YAML, Connect to first caller in queue

```yaml
- connect:
    to: "queue:my_queue"
```

##### JSON, Connect to bidirectional WebSocket stream

```json
{
  "connect": {
    "to": "stream:wss://example.com/audio",
    "codec": "PCMU",
    "realtime": true,
    "authorization_bearer_token": "my-secret-token",
    "custom_parameters": {
      "user_id": "12345",
      "session_type": "voice"
    }
  }
}
```

##### YAML, Connect to bidirectional WebSocket stream

```yaml
- connect:
    to: "stream:wss://example.com/audio"
    codec: PCMU
    realtime: true
    authorization_bearer_token: "my-secret-token"
    custom_parameters:
      user_id: "12345"
      session_type: "voice"
```

---

### dial

(EXPERIMENTAL)

Initiate an outbound call without bridging to the current call. Similar to `connect` but creates a new independent call leg that can execute its own SWML script.

#### Parameters

Same parameters as [connect](#connect-) with the addition of:

- **dest_swml:** String or Object, Optional. SWML script URL or inline SWML to execute on the dialed call. Can be a URL string or inline SWML object/array.

#### Variables

- **dial_result:** (out) `success` | `failed` | `error`
- **dial_result_code:** (out) HTTP-style result code
- **dial_result_message:** (out) Result message
- **dial_id:** (out) Identifier for the dialed call if successful

#### Examples

##### JSON, Dial with inline SWML for destination

```json
{
  "dial": {
    "to": "+15551231234",
    "from": "+15553214321",
    "dest_swml": {
      "version": "1.0.0",
      "sections": {
        "main": [
          "answer",
          { "play": "say:Hello from the dialed call" },
          "hangup"
        ]
      }
    }
  }
}
```

##### JSON, Dial with SWML URL

```json
{
  "dial": {
    "to": "+15551231234",
    "dest_swml": "https://example.com/outbound-script.json"
  }
}
```

---

### tap (uri, direction)

Start background call tap. Media is streamed over Websocket or RTP to customer controlled URI.

#### Parameters

- **uri:** String, Required destination of tap media stream. `rtp://IP:port`, `ws://example.com`, or `wss://example.com`
- **control_id:** String, Optional identifier for this tap to use with `stop_tap`. Default is generated and stored in tap_control_id variable.
- **direction:** String, Optional. Direction of audio to tap. speak = what party says. hear = what party hears. both = what party hears and says. Default = both.
- **codec:** String, Optional. `PCMU` or `PCMA`. Default `PCMU`
- **rtp_ptime:** Integer, Optional. Milliseconds. If `rtp://` URI, this parameter can set the packetization time of the media. Default 20 ms.
- **`status_url:`** optional - the URL to make requests to for each status change during tap.

#### Variables

- **tap_uri:** (out) the destination URI of the newly started tap
- **tap_result:** (out) success | failed
- **tap_control_id:** (out) control ID of this tap
- **tap_rtp_src_addr:** (out) if RTP, source address of the tap stream
- **tap_rtp_src_port:** (out) if RTP, source port of the tap stream
- **tap_ptime:** (out) packetization time of the tap stream
- **tap_codec:** (out) codec in the tap stream
- **tap_rate:** (out) sample rate in the tap stream

#### Examples

##### JSON, start wss tap

```json
{ "tap": { "uri": "wss://example.com/tap" } }
```

##### YAML, start wss tap

```yaml
- tap: wss://example.com/tap
```

---

### stop_tap (control_id)

Stop an active tap stream

#### Parameters

- **control_id:** String, Optional ID of the tap to stop. If not set, the last tap started will be stopped.

#### Variables

- **tap_control_id:** (in) control ID of last tap stream started
- **stop_tap_result:** (out) success or failed

#### Examples

##### JSON, stop last call tap

```json
"stop_tap"
```

##### YAML, stop last call tap

```yaml
- stop_tap
```

##### JSON, stop a specific call tap

```json
{ "stop_tap": "my-tap-id" }
```

##### JSON, stop a specific call tap

```yaml
- stop_tap: my-tap-id
```

---

### send_digits (digits)

Send digit presses.

#### Parameters

- **digits:** String, Required. The digits to send. Valid values are `0123456789*#ABCDWw` `W` is a 1 second delay. `w` is a 500 ms delay.

#### Variables

- **send_digits_result:** (out) success | failed

#### Examples

##### JSON, send digits

```json
{ "send_digits": "012345" }
```

##### YAML, send digits

```yaml
- send_digits: "012345"
```

---

### sleep (duration)

Sleep for duration in ms

#### Parameters

- **duration:** Required positive integer or -1

#### Examples

##### JSON, sleep forever

```json
{ "sleep": -1 }
```

##### JSON, sleep for 2 seconds

```json
{ "sleep": 2000 }
```

##### JSON, verbose sleep

```json
{ "sleep": { "duration": -1 } }
```

---

### send_sms

Send a text message to a PSTN phone number.

#### Parameters

- **to_number**: String, Required - phone number in e.164 format.
- **from_number**: String, Required - phone number in e.164 format.
- **body**: String, Required (optional if media is present) - body of the message.
- **media**: Array(String), Required (optional if body is present) - an array of URLs to send in the message.
- **tags**: Array(String), Optional - an array of tags to associate to the message, to help when searching in the UI. Default is not set.
- **region**: String, Optional - region of the world to originate the call from. Default picked based on account prefs or device location.

#### Variables

- **send_sms_result:** (out) success | failed

#### Examples

##### JSON, send SMS

```json
{
  "send_sms": {
    "tags": ["Custom", "client", "data"],
    "region": "us",
    "from_number": "+155512312345",
    "to_number": "+15555554321",
    "body": "Message Body",
    "media": ["url1", "url2"]
  }
}
```

##### YAML, send SMS

```yaml
- send_sms:
    tags:
      - Custom
      - data
    region: us
    from_number: "+155512312345"
    to_number: "+15555554321"
    body: Message Body
    media:
      - url1
      - url2
```

---

### ai (agent)

Connect to an AI Agent.

#### Parameters

- **agent**: string, optional - ID of the agent to connect to

  OR - detailed configuration for agent defined below

- **engine**: (DEPRECATED) string, optional - the TTS engine the AI agent will use. Default is picked by SignalWire.
- **voice**: (DEPRECATED) string, optional - the TTS voice the AI agent will use. Default is picked by SignalWire.
- **post_prompt_url**: string, optional - url to send status callbacks/reports to. Default is not set.
- **post_prompt_auth_user**: string, optional - auth username for post_url endpoint. Default is not set.
- **post_prompt_auth_password**: string, optional - auth password for post_url endpoint. Default is not set.
- **global_data**: object, optional - global data accessible to all SWAIG functions.
- **pronounce**: array, optional - global pronunciation rules.
- **hints**: optional - an array of hints (as strings) to provide context to the dialogue. Default is not set.
- **result:** Optional `switch` on `return_value` when object `{}` or `cond` when array `[]`

#### prompt

- **prompt.text**: optional - the initial message/instruction-set to send to the agent. Default is not set.
- **prompt.top_p**: optional - probability cutoff. Closer to 0 is less random (alternative to temperature). Float 0.0-1.0 (default 1.0).
- **prompt.temperature**: optional - variance/randomness. Float 0.0-2.0 (default 1.0).
- **prompt.confidence**: optional - ASR speech-detect confidence threshold. Float 0.0-1.0 (default 0.6).
- **prompt.barge_confidence**: optional - confidence for speech detection during exchange. Float 0.0-1.0. Default is picked by SignalWire.
- **prompt.presence_penalty**: optional - aversion to staying on topic. Float -2.0 to 2.0 (default 0).
- **prompt.frequency_penalty**: optional - aversion to repeating lines. Float -2.0 to 2.0 (default 0).
- **prompt.model**: optional - specific LLM model for this prompt.

#### post_prompt

- **post_prompt.text**: optional - the final message/instruction set to send to the agent. Default is not set.
- **post_prompt.top_p**: optional - probability cutoff. Float 0.0-1.0 (default 1.0).
- **post_prompt.temperature**: optional - variance/randomness. Float 0.0-2.0 (default 1.0).
- **post_prompt.barge_confidence**: optional - confidence for speech detection during exchange. Float 0.0-1.0.
- **post_prompt.presence_penalty**: optional - aversion to staying on topic. Float -2.0 to 2.0 (default 0).
- **post_prompt.frequency_penalty**: optional - aversion to repeating lines. Float -2.0 to 2.0 (default 0).
- **post_prompt.model**: optional - specific LLM model for post-prompt summarization.

#### languages

Array of language objects defining supported languages. Default is en-US. Activated by `params.languages_enabled`.

| Field | Type | Description |
|---|---|---|
| `name` | string | Language display name |
| `code` / `listen_language` | string | Language code(s) for ASR |
| `voice` | string | TTS voice ID. Supports `engine.voice:model` syntax |
| `engine` | string | Optional TTS engine override |
| `model` | string | Optional speech model override |
| `params` | object | Language-specific parameter overrides |
| `fillers` | array | Filler phrases for this language |
| `speech_fillers` | array | Speech-specific fillers |
| `function_fillers` | array | Function execution fillers |
| `pronounce` | object | Pronunciation rules for this language |
| `auto_emotion` | boolean | Enable automatic emotion (supported engines) |
| `auto_speed` | boolean | Enable automatic speed adjustment (supported engines) |

#### SWAIG

The `SWAIG` object configures user-defined functions that can be called during the AI dialogue.

##### SWAIG Structure

| Field | Type | Description |
|---|---|---|
| `defaults` | object | Default settings for all functions |
| `functions` | array | Function definitions |
| `includes` | array | Remote function signature URLs |
| `native_functions` | array | Built-in function names to enable |
| `hooks` | array | Webhook hooks |
| `internal_fillers` | array | Filler phrases during processing |

##### SWAIG Defaults

| Field | Type | Description |
|---|---|---|
| `web_hook_url` | string | Default webhook URL for all functions |
| `web_hook_auth_user` | string | Default auth user |
| `web_hook_auth_password` | string | Default auth password |
| `meta_data_token` | string | Default metadata token |
| `meta_data` | object | Default metadata sent with all function calls |

##### SWAIG Function Definition

| Field | Type | Description |
|---|---|---|
| `function` | string | Function name the LLM calls |
| `purpose` / `description` | string | Description shown to LLM so it knows when to call this function |
| `argument` / `parameters` | object | JSON Schema defining function parameters |
| `web_hook_url` | string | Webhook endpoint (overrides default) |
| `web_hook_auth_user` | string | Auth user (overrides default) |
| `web_hook_auth_pass` | string | Auth password (overrides default) |
| `meta_data` | object | Metadata sent with function calls |
| `meta_data_token` | string | Token for metadata validation |
| `data_map` | object | Data mapping configuration for transforming webhook responses |
| `fillers` | array | Filler phrases while this function executes |
| `wait_file` | string | Audio file to play while waiting for webhook response |
| `wait_file_loops` | integer | Loop count for wait file |
| `wait_for_fillers` | boolean | Synchronize fillers with function execution |
| `skip_fillers` | boolean | Skip filler phrases for this function |
| `active` | boolean | Enable/disable this function at runtime |

##### SWAIG Includes

| Field | Type | Description |
|---|---|---|
| `url` | string | URL to fetch remote function signatures |
| `auth_user` | string | Auth user for the URL |
| `auth_password` | string | Auth password for the URL |
| `functions` | array | Specific function names to include from the remote |
| `meta_data` | object | Metadata applied to all included functions |

#### params

The `params` object provides detailed control over AI agent behavior.

##### Speech Recognition (ASR)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `openai_asr_engine` | string | global default | ASR engine (e.g. `"gcloud_speech_v2_async"`, `"deepgram"`) |
| `asr_smart_format` | boolean | false | Enable smart formatting in ASR output (punctuation, capitalization) |
| `asr_diarize` | boolean | false | Enable speaker diarization |
| `asr_speaker_affinity` | boolean | false | Track speaker identity across turns. Requires `asr_diarize` |
| `llm_diarize_aware` | boolean | false | Include speaker labels in LLM context. Requires `asr_diarize` |
| `confidence` | float | 1.0 | Minimum ASR confidence threshold. Results below this are discarded |
| `energy_level` | float | 0.0–100.0 | VAD energy threshold in decibels. Higher = less sensitive to quiet speech |

##### Turn Detection & Timing

| Parameter | Type | Default | Range | Description |
|---|---|---|---|---|
| `enable_turn_detection` | boolean | false | — | Enable model-assisted turn detection |
| `turn_detection_timeout` | integer (ms) | — | 0–10000 | Time after speech ends before turn is complete |
| `turn_detection_min_length` | integer (ms) | — | 0–1000 | Minimum utterance length to count as a turn |
| `end_of_speech_timeout` | integer (ms) | — | 0–10000 | Silence duration after speech to trigger end-of-speech |
| `first_word_timeout` | integer (ms) | — | 0–10000 | Max wait for the first word of a new utterance |
| `speech_event_timeout` | integer (ms) | — | 0–10000 | Timeout for individual speech events |
| `attention_timeout` | integer (ms) | — | 0 or 10000–600000 | User silence before agent prompts. 0 = disabled |
| `attention_timeout_prompt` | string | built-in | — | Custom prompt when attention timeout fires |
| `outbound_attention_timeout` | integer (ms) | — | 10000–600000 | Attention timeout for outbound calls |
| `inactivity_timeout` | integer (ms) | — | 0 or 10000–3600000 | Total inactivity before session ends. 0 = disabled |
| `input_poll_freq` | integer (ms) | — | 1000–10000 | How often to poll for user input |

##### VAD (Voice Activity Detection)

| Parameter | Type | Description |
|---|---|---|
| `vad_config` | string | Silero VAD config string. Parsed for `silero_thresh` and `silero_frame_ms` |

##### Barge-in (Interruption)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `enable_barge` | string | — | ASR events that trigger barge-in: `"complete"`, `"partial"`, `"all"` / `true`, `false` |
| `transparent_barge` | boolean | false | Barge stops TTS but interrupted text still added to conversation history |
| `transparent_barge_max_time` | integer (ms) | — | 0–60000. Max elapsed time for transparent barge. After this, normal barge resumes |
| `acknowledge_interruptions` | int/bool | — | Times AI acknowledges being interrupted. `true` = 1, integer = count |
| `interrupt_prompt` | string | — | Custom prompt sent to LLM after user barges in |
| `interrupt_on_noise` | int/bool | — | Interrupt on non-speech noise. `true` = 50ms, integer = custom ms |
| `barge_min_words` | integer | — | 1–99. Minimum words before barge-in is allowed |
| `barge_match_string` | string | — | Only barge if speech contains this string. Disables `transparent_barge` |
| `barge_confidence` | float | — | Minimum ASR confidence required for barge to trigger |
| `barge_functions` | boolean | false | Allow barge-in during function execution |
| `function_wait_for_talking` | boolean | false | Wait for AI to finish speaking before executing function results |

##### Text-to-Speech (TTS)

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ai_volume` | integer | 0 | AI speech volume adjustment. Range: -50 to +50 |
| `tts_number_format` | string | — | How numbers are spoken: `"national"`, `"international"`, `"generic"` |
| `speech_gen_quick_stops` | integer | — | 0–10. Quick-stop points in generated speech for faster barge response |

**ElevenLabs:**

| Parameter | Type | Description |
|---|---|---|
| `eleven_labs_key` | string | ElevenLabs API key (overrides global) |
| `eleven_labs_model` | string | Model: `"multilingual"`, `"eleven_english_v2"`, `"eleven_turbo_v2"`, `"eleven_turbo_v2_5"`, `"eleven_multilingual_v1"`, `"eleven_multilingual_v2"`, `"eleven_flash_v2_5"`, `"eleven_v3"` |
| `eleven_labs_stability` | float | Voice stability 0.0–1.0 |
| `eleven_labs_similarity` | float | Voice clarity/similarity 0.0–1.0 |
| `eleven_labs_stream_first` | boolean | Stream the first utterance instead of downloading |

**Azure TTS:**

| Parameter | Type | Description |
|---|---|---|
| `azure_tts_key` | string | Azure Speech Services key |
| `azure_stream_first` | boolean | Stream the first utterance |

**OpenAI TTS:**

| Parameter | Type | Description |
|---|---|---|
| `openai_stream_first` | boolean | Stream the first utterance |

**Cartesia:**

| Parameter | Type | Description |
|---|---|---|
| `cartesia_key` | string | Cartesia API key |
| `cartesia_model` | string | Model: `"sonic-english"`, `"sonic-multilingual"` |
| `max_emotion` | integer | Maximum emotion intensity. Range: 1–30 |

**Rime:**

| Parameter | Type | Description |
|---|---|---|
| `rime_model` | string | Model: `"mist"`, `"mistv2"`, `"arcana"` |
| `rime_stream_first` | boolean | Stream the first utterance |

**MiniMax:**

| Parameter | Type | Description |
|---|---|---|
| `minimax_model` | string | Model: `"speech-2.6-hd"`, `"speech-2.6-turbo"`, `"speech-02-hd"`, `"speech-02-turbo"`, `"speech-01-hd"`, `"speech-01-turbo"` |

**Inworld:**

| Parameter | Type | Description |
|---|---|---|
| `inworld_model` | string | Inworld TTS model ID |

**Deepgram TTS:**

| Parameter | Type | Description |
|---|---|---|
| `deepgram_stream_first` | boolean | Stream the first utterance |

##### LLM Configuration

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ai_model` | string | — | LLM model to use for the AI agent |
| `thinking_model` | string | — | Override model for extended thinking/reasoning |
| `inner_dialog_model` | string | — | Override model for inner dialog reasoning |
| `vision_model` | string | — | Override model for image/vision processing |
| `utility_model` | string | — | Override model for utility operations (auto_correct, summaries) |
| `max_response_tokens` | integer | — | 1–16384. Maximum tokens the LLM may generate per response |
| `conversation_sliding_window` | integer | — | Conversation turns to keep in context before older ones drop |

##### AI Behavior

| Parameter | Type | Default | Description |
|---|---|---|---|
| `conscience` / `strict_mode` | string/bool | model default | AI safety. `false` = disabled, `true` = model built-in, string = custom prompt |
| `enable_thinking` | boolean | false | Enable extended thinking — LLM reasons step-by-step before responding |
| `enable_inner_dialog` | boolean | false | Enable inner dialog reasoning pass before spoken response |
| `inner_dialog_prompt` | string | built-in | Custom system prompt for the inner dialog model |
| `auto_correct` | boolean | false | Use LLM to clean up ASR text. Mutually exclusive with `enable_text_normalization` |
| `redact_prompt` | string | — | Prompt for redacting sensitive information from logs |
| `functions_on_no_response` | boolean | false | Allow function execution when user hasn't responded |

##### Text Normalization (NeMo FST)

FST-based text normalization using NeMo grammars (~18ms, no LLM needed). Mutually exclusive with `auto_correct`.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `enable_text_normalization` | string | — | `"heard"` = spoken→written ("twenty three dollars" → "$23"). `"spoken"` = written→spoken ("$23.50" → "twenty three dollars fifty cents"). `"both"` = both |
| `text_normalization_far_dir` | string | `/usr/share/freeswitch/grammar/nemo` | Base directory for NeMo FAR grammar files |

##### Call Control

| Parameter | Type | Default | Description |
|---|---|---|---|
| `direction` | string | — | `"inbound"` or `"outbound"`. Affects timeout behavior and greeting logic |
| `wait_for_user` | boolean | false | Wait for user to speak first |
| `static_greeting` | string | — | Fixed greeting spoken immediately (skips LLM for first utterance) |
| `static_greeting_no_barge` | boolean | false | Prevent user from interrupting the static greeting |
| `hard_stop_time` | string | — | Absolute or relative time when session must end (e.g. `"1h30m"`, `"14:30"`) |
| `hard_stop_prompt` | string | built-in | Prompt sent to LLM when hard stop time is reached |
| `initial_sleep_ms` | integer | 0 | 0–300000. Delay in ms before AI starts processing |
| `hold_music` | string | — | Audio file for hold music. Prefix with `tone:` for tone_stream |
| `hold_on_process` | boolean | false | Play hold music while processing LLM responses |
| `background_file` | string | — | Background audio played during conversation |
| `background_file_loops` | integer | — | Loop count for background audio (> 0) |
| `background_file_volume` | integer | — | Volume for background audio |

##### Messaging

| Parameter | Type | Default | Description |
|---|---|---|---|
| `sms_enabled` | boolean | false | Enable SMS capabilities |
| `sms_from_number` | string | — | Origination number for SMS (E.164) |
| `call_enabled` | boolean | false | Enable outbound calling |
| `call_from_number` | string | — | Origination number for outbound calls (E.164) |

##### DTMF

| Parameter | Type | Default | Description |
|---|---|---|---|
| `digit_timeout` | integer (ms) | — | 1–29999. Time to wait for additional DTMF digits |
| `digit_terminators` | string | — | Characters that terminate DTMF collection (e.g. `"#"`) |

##### Video

| Parameter | Type | Description |
|---|---|---|
| `enable_vision` | boolean | Enable vision/image processing capabilities |
| `video_talking_file` | string | Video file displayed while AI is speaking. Requires `video_idle_file` |
| `video_idle_file` | string | Video file displayed while idle. Requires `video_talking_file` |
| `video_listening_file` | string | Video file displayed while listening |

##### Pause/Wake

| Parameter | Type | Default | Description |
|---|---|---|---|
| `enable_pause` | boolean | false | Allow the AI session to be paused and resumed |
| `start_paused` | boolean | false | Start in paused state. Implies `enable_pause` |
| `speak_when_spoken_to` | boolean | false | AI only speaks in response to user input |
| `wake_prefix` | string | — | Prefix word/phrase that must precede user speech to wake the AI |
| `ai_name` | string | — | Name the AI responds to |

##### Language

| Parameter | Type | Default | Description |
|---|---|---|---|
| `languages_enabled` | boolean | false | Enable multi-language support. Activates the top-level `languages` array |

##### Conversation & Data

| Parameter | Type | Default | Description |
|---|---|---|---|
| `conversation_id` | string | — | Unique ID for this conversation. Required for `save_conversation` |
| `save_conversation` | boolean | false | Persist conversation history. Requires `conversation_id` |
| `transfer_summary` | boolean | false | Include conversation summary when transferring calls |
| `summary_mode` | string | — | Summary format: `"string"` = plain text, `"original"` = original format |
| `persist_global_data` | boolean | false | Persist `global_data` across sessions (via `conversation_id`) |

##### SWAIG Permissions

| Parameter | Type | Default | Description |
|---|---|---|---|
| `swaig_allow_swml` | boolean | false | Allow SWAIG functions to return SWML for execution |
| `swaig_allow_settings` | boolean | false | Allow SWAIG functions to modify session settings at runtime |
| `swaig_post_conversation` | boolean | false | POST full conversation to SWAIG function webhooks |
| `swaig_set_global_data` | boolean | false | Allow SWAIG functions to set global data |
| `swaig_post_swml_vars` | object | — | SWML variables to include in SWAIG function POST data |

##### Debug & Monitoring

| Parameter | Type | Default | Description |
|---|---|---|---|
| `debug_webhook_url` | string | — | URL to receive debug event webhooks |
| `debug_webhook_level` | integer | 0 | 0–2. Verbosity of debug webhook events |

##### Miscellaneous

| Parameter | Type | Description |
|---|---|---|
| `local_tz` | string | Local timezone (e.g. `"America/New_York"`) |
| `pom_format` | string | Prompt format: `"xml"` or `"markdown"` |

#### Parameter Interactions

**Mutually Exclusive:**
- `auto_correct` and `enable_text_normalization` — cannot both be active. If both set, `auto_correct` is disabled with a warning.
- `barge_match_string` disables `transparent_barge`.

**Dependencies:**
- `save_conversation` requires `conversation_id`.
- `video_talking_file` and `video_idle_file` must both be set for video to work.
- `asr_speaker_affinity` and `llm_diarize_aware` require `asr_diarize`.
- `start_paused` implies `enable_pause`.
- `languages_enabled` activates the top-level `languages` array.
- `inner_dialog_prompt` is only used when `enable_inner_dialog` is true.

**Outbound Calls:**
- `direction` set to `"outbound"` uses `outbound_attention_timeout` instead of `attention_timeout`.

#### Variables

- **ai_result:** (out) success | failed
- **return_value:** (out) `ai_result`

#### Examples

##### JSON - pre-configured agent

```json
{ "ai": "76f5b553-7251-4b06-a2b3-bf7b6e2b8560" }
```

##### JSON - pre-configured agent with retries

```json
{
  "label": "retry_ai",
  "ai": {
    "agent": "76f5b553-7251-4b06-a2b3-bf7b6e2b8560",
    "result": {
      "case": {
        "failed": [ { "goto": { "label": "retry_ai", "max": 1 } } ]
      }
    }
  }
}
```

##### JSON - configure agent

```json
{
  "label": "retry_ai",
  "ai": {
    "engine": "gcloud",
    "voice": "en-US-Neural2-F",
    "post_prompt_url": "http://some.server.com/rest/post.cgi",
    "post_prompt_auth_user": "someuser",
    "post_prompt_auth_password": "secret",
    "result": {
      "case": {
        "failed": [ { "goto": { "label": "retry_ai", "max": 1 } } ]
      }
    },
    "params": {
      "direction": "outbound",
      "wait_for_user": true,
      "sms_from_number": "12121231234",
      "call_from_number": "12121231234",
      "call_enabled": false,
      "sms_enabled": false,
      "end_of_speech_timeout": 3000,
      "attention_timeout": 450000,
      "outbound_attention_timeout": 120000,
      "language_mode": "normal",
      "languages_enabled": true
    },
    "hints": [
      "testing",
      "hints"
    ],
    "prompt": {
      "confidence": 0.6,
      "barge_confidence": 0.1,
      "top_p": 1.0,
      "temperature": 0.3,
      "frequency_penalty": 0.1,
      "presence_penalty": 0.1,
      "text": "You name is Franklin and you are an expert at Star Wars. Introduce yourself and see if I have any questions."
    },
    "post_prompt": {
      "confidence": 0.6,
      "barge_confidence": 0.1,
      "top_p": 1.0,
      "temperature": 0.3,
      "frequency_penalty": 0,
      "presence_penalty": 0,
      "text": "Please summarize the conversation"
    },
    "languages": [
      {
        "name": "English",
        "code": "en-US",
        "voice": "en-US-Neural2-F"
      },
      {
        "name": "French",
        "code": "fr-FR",
        "voice": "fr-FR-Neural2-E"
      }
    ],
    "SWAIG": {
      "defaults": {
        "web_hook_url": "https://somewhere.com/weather.cgi",
        "web_hook_auth_user": "me",
        "web_hook_auth_pass": "secret",
        "meta_data": {
          "my_key": "some value"
        }
      },
      "functions": [
        {
          "function": "get_weather",
          "purpose": "To determine what the current weather is in a provided location.",
          "argument": "The location or name of the city to get the weather from."
        }
      ]
    }
  }
}
```


##### YAML - pre-configured agent

```yaml
 - ai: "76f5b553-7251-4b06-a2b3-bf7b6e2b8560"
```

##### YAML - pre-configured agent with retries

```yaml
- label:retry_ai
  ai:
  agent: "76f5b553-7251-4b06-a2b3-bf7b6e2b8560"
  result:
    case:
      failed:
      - goto:
          label: retry_ai
          max: 1
```


##### YAML - configure agent

```yaml
- label:retry_ai
  ai:
  engine: gcloud
  voice: "en-US-Neural2-F"
  post_prompt_url: "http://some.server.com/rest/post.cgi"
  post_prompt_auth_user: someuser
  post_prompt_password: secret
  result:
    case:
      failed:
      - goto:
          label: retry_ai
          max: 1
  params:
    direction: outbound
    wait_for_user: true
    sms_from_number: 12121231234
    call_from_number: 12121231234
    call_enabled: false
    sms_enabled: false
    end_of_speech_timeout: 3000
    attention_timeout: 450000
    outbound_attention_timeout: 120000
    language_mode: normal
    languages_enabled: true
  hints:
    - testing
    - hints
  prompt:
    confidence: 0.6
    barge_confidence: 0.1
    top_p: 1.0
    temperature: 0.3
    frequency_penalty: 0.1
    presence_penalty: 0.1
    text: "You name is Franklin and you are an expert at Star Wars. Introduce yourself and see if I have any questions."
  post_prompt:
    confidence: 0.6
    barge_confidence: 0.1
    top_p: 1.0
    temperature: 0.3
    frequency_penalty: 0
    presence_penalty: 0
    text: "Please summarize the conversation."
  languages:
    - name: "English"
      code: "en-US"
      voice: "en-US-Neural2-F"
    - name: "French"
      code: "fr-FR"
      voice: "fr-FR-Neural2-E"
  SWAIG:
    defaults:
      web_hook_url: "https://somewhere.com/weather.cgi"
      web_hook_auth_user: "me"
      web_hook_auth_pass: "secret"
      meta_data:
        my_key: "some value"
    functions:
      - function: "get_weather"
        purpose: "To determine what the current weather is in a provided location."
        argument: "The location or name of the city to get the weather from."
```

---

### amazon_bedrock

Connect to an Amazon Bedrock AI agent.

#### Parameters

Parameters are passed through to the RELAY amazon_bedrock method. See RELAY documentation for available parameters.

#### Variables

None

#### Examples

##### JSON

```json
{
  "amazon_bedrock": {
    "agent_id": "your-bedrock-agent-id",
    "agent_alias_id": "your-agent-alias"
  }
}
```

---

### user_event

(EXPERIMENTAL)

Send a custom user event on the call.

#### Parameters

Parameters are passed through to the RELAY user_event method. Any custom key-value pairs can be included.

#### Variables

None

#### Examples

##### JSON

```json
{
  "user_event": {
    "event_type": "custom_event",
    "data": {
      "key1": "value1",
      "key2": "value2"
    }
  }
}
```

---

## Webhook Callbacks Reference

This section documents all webhook/callback URLs in SWML methods, including the HTTP request details and payload structures.

### Common Request Headers

All webhook requests from SignalWire include the following headers:

| Header | Value |
|--------|-------|
| `Content-Type` | `application/json` |
| `User-Agent` | `SignalWire` |
| `X-SignalWire-Signature` | HMAC signature for request verification (external webhooks only) |

### Document Fetching Webhook

When SWML fetches a document from a URL (initial call routing, `transfer`, or `execute` with URL destination), SignalWire sends an HTTP POST request.

#### Request

- **Method:** `POST`
- **Content-Type:** `application/json`

#### Request Body

**Example: Phone call**

```json
{
  "call": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "segment_id": "c3d4e5f6-a7b8-9012-cdef-123456789abc",
    "project_id": "project-uuid",
    "space_id": "space-uuid",
    "call_state": "created",
    "direction": "inbound",
    "type": "phone",
    "from": "+15551231234",
    "to": "+15553214321",
    "from_number": "+15551231234",
    "to_number": "+15553214321",
    "headers": []
  },
  "vars": {
    "custom_var_1": "value1",
    "custom_var_2": "value2"
  },
  "envs": {
    "user_defined_env": "env_value"
  },
  "params": {
    "subroutine_param_1": "param_value"
  }
}
```

**Example: SIP call**

```json
{
  "call": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "segment_id": "c3d4e5f6-a7b8-9012-cdef-123456789abc",
    "project_id": "project-uuid",
    "space_id": "space-uuid",
    "call_state": "created",
    "direction": "inbound",
    "type": "sip",
    "from": "sip:user@example.com",
    "to": "sip:destination@yourdomain.com",
    "headers": [
      { "name": "X-Custom-Header", "value": "header-value" }
    ],
    "sip_data": {
      "sip_req_user": "destination",
      "sip_req_uri": "destination@yourdomain.com",
      "sip_req_host": "yourdomain.com",
      "sip_from_user": "user",
      "sip_from_uri": "user@example.com",
      "sip_from_host": "example.com",
      "sip_to_user": "destination",
      "sip_to_uri": "destination@yourdomain.com",
      "sip_to_host": "yourdomain.com",
      "sip_contact_user": "user",
      "sip_contact_port": "5060",
      "sip_contact_uri": "user@192.168.1.100:5060",
      "sip_contact_host": "192.168.1.100",
      "sip_contact_params": {},
      "sip_from_params": {},
      "sip_to_params": {},
      "sip_req_params": {}
    }
  },
  "vars": {
    "user_selection": "1"
  },
  "envs": {
    "api_key": "your-api-key"
  },
  "params": {
    "department": "sales"
  }
}
```

#### Call Object Fields

| Field | Type | Description |
|-------|------|-------------|
| `call_id` | String | Unique identifier for the call |
| `node_id` | String | Identifier for the SignalWire node handling the call |
| `segment_id` | String | Identifier for the call segment |
| `project_id` | String | Your SignalWire project UUID |
| `space_id` | String | Your SignalWire space UUID |
| `call_state` | String | Current call state: `created`, `ringing`, `answered`, `ending`, `ended` |
| `direction` | String | Call direction: `inbound` or `outbound` |
| `type` | String | Call type: `phone`, `sip`, `webrtc` |
| `from` | String | Caller ID (phone number or SIP URI) |
| `to` | String | Destination (phone number or SIP URI) |
| `from_number` | String | Caller phone number (phone calls only) |
| `to_number` | String | Destination phone number (phone calls only) |
| `headers` | Array | Custom SIP headers as `{name, value}` objects |
| `tag` | String | Optional tag associated with the call |
| `parent` | Object | Parent call info: `{call_id, node_id, device_type}` (if applicable) |
| `peer` | Object | Peer call info: `{call_id, node_id}` (if applicable) |
| `end_reason` | String | Reason call ended (when call ends): `busy`, `cancel`, `hangup`, `error`, etc. |
| `end_source` | String | Source of call end (when call ends): `caller`, `callee`, `system` |

#### SIP Data Fields (type="sip" only)

| Field | Type | Description |
|-------|------|-------------|
| `sip_req_user` | String | Request URI user part |
| `sip_req_uri` | String | Full request URI |
| `sip_req_host` | String | Request URI host |
| `sip_from_user` | String | From header user part |
| `sip_from_uri` | String | Full From URI |
| `sip_from_host` | String | From header host |
| `sip_to_user` | String | To header user part |
| `sip_to_uri` | String | Full To URI |
| `sip_to_host` | String | To header host |
| `sip_contact_user` | String | Contact header user part |
| `sip_contact_port` | String | Contact header port |
| `sip_contact_uri` | String | Full Contact URI |
| `sip_contact_host` | String | Contact header host |
| `sip_contact_params` | Object | Contact URI parameters |
| `sip_from_params` | Object | From URI parameters |
| `sip_to_params` | Object | To URI parameters |
| `sip_req_params` | Object | Request URI parameters |

#### Expected Response

Your server should respond with a SWML document:

- **Content-Type:** `application/json`, `application/yaml`, or `text/x-yaml`
- **HTTP Status:** 2xx for success

```json
{
  "version": "1.0.0",
  "sections": {
    "main": [
      "answer",
      { "play": "say:Hello, welcome to our service." },
      "hangup"
    ]
  }
}
```

---

### play - status_url

Receives status events during and after audio playback.

#### Request

- **Method:** `POST`
- **Triggered:** When play starts, pauses, resumes, finishes, or encounters an error

#### Request Body

```json
{
  "event_type": "calling.call.play",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "control_id": "play-control-id",
    "state": "finished"
  }
}
```

#### State Values

| State | Description |
|-------|-------------|
| `playing` | Playback has started |
| `paused` | Playback is paused |
| `finished` | Playback completed successfully |
| `error` | An error occurred during playback |

---

### prompt - status_url

Receives status events for prompt (play and collect) operations.

#### Request

- **Method:** `POST`
- **Triggered:** When prompt starts, receives input, or completes

#### Request Body

```json
{
  "event_type": "calling.call.collect",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "control_id": "prompt-control-id",
    "result": {
      "type": "digit",
      "params": {
        "digits": "1234",
        "terminator": "#"
      }
    }
  }
}
```

#### Result Types

| Type | Description |
|------|-------------|
| `digit` | DTMF digits were collected |
| `speech` | Speech was recognized |
| `no_input` | No input was received within timeout |
| `error` | An error occurred |

---

### record / record_call - status_url

Receives status events for recording operations.

#### Request

- **Method:** `POST`
- **Triggered:** When recording starts, pauses, resumes, or finishes

#### Request Body

```json
{
  "event_type": "calling.call.record",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "control_id": "record-control-id",
    "state": "finished",
    "record": {
      "url": "https://your-space.signalwire.com/api/v1/recordings/rec-uuid/download",
      "duration": 15.5,
      "size": 248320,
      "format": "wav"
    }
  }
}
```

#### State Values

| State | Description |
|-------|-------------|
| `recording` | Recording has started |
| `paused` | Recording is paused |
| `finished` | Recording completed successfully |
| `no_input` | No audio was detected |
| `error` | An error occurred |

#### Record Object Fields

| Field | Type | Description |
|-------|------|-------------|
| `url` | String | URL to download the recording |
| `duration` | Number | Recording duration in seconds |
| `size` | Number | Recording file size in bytes |
| `format` | String | Recording format: `wav` or `mp3` |

---

### connect / dial - call_state_url

Receives call state change notifications for outbound call legs.

#### Request

- **Method:** `POST`
- **Triggered:** Based on `call_state_events` setting (default: `ended`)

#### Request Body

```json
{
  "event_type": "calling.call.state",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "call_state": "answered",
    "direction": "outbound",
    "device": {
      "type": "phone",
      "params": {
        "from_number": "+15551231234",
        "to_number": "+15553214321"
      }
    },
    "end_reason": null
  }
}
```

#### Call State Values

| State | Description |
|-------|-------------|
| `created` | Call leg was created |
| `ringing` | Remote party is ringing |
| `answered` | Remote party answered |
| `ended` | Call leg ended |

#### End Reason Values (when `call_state` is `ended`)

| Reason | Description |
|--------|-------------|
| `hangup` | Normal hangup |
| `busy` | Remote party is busy |
| `no_answer` | Remote party did not answer |
| `cancel` | Call was canceled |
| `declined` | Call was declined |
| `error` | An error occurred |

---

### connect / dial - status_url

Receives overall connect/dial operation status.

#### Request

- **Method:** `POST`
- **Triggered:** When the connect/dial operation completes

#### Request Body

```json
{
  "event_type": "calling.call.connect",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "connect_state": "connected",
    "peer": {
      "call_id": "peer-call-id",
      "node_id": "peer-node-id",
      "device": {
        "type": "phone",
        "params": {
          "from_number": "+15551231234",
          "to_number": "+15553214321"
        }
      }
    }
  }
}
```

#### Connect State Values

| State | Description |
|-------|-------------|
| `connecting` | Attempting to connect |
| `connected` | Successfully connected |
| `failed` | Connection failed |
| `disconnected` | Connection ended |

---

### tap - status_url

Receives status events for media tap/stream operations.

#### Request

- **Method:** `POST`
- **Triggered:** When tap starts or ends

#### Request Body

```json
{
  "event_type": "calling.call.tap",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "control_id": "tap-control-id",
    "state": "tapping",
    "tap": {
      "type": "audio",
      "params": {
        "direction": "both",
        "codec": "PCMU",
        "rate": 8000,
        "ptime": 20
      }
    },
    "device": {
      "type": "rtp",
      "params": {
        "addr": "192.168.1.100",
        "port": 12345
      }
    }
  }
}
```

#### State Values

| State | Description |
|-------|-------------|
| `tapping` | Tap has started |
| `finished` | Tap has ended |

---

### detect_machine - status_url

Receives detection results for answering machine detection.

#### Request

- **Method:** `POST`
- **Triggered:** When detection completes

#### Request Body

```json
{
  "event_type": "calling.call.detect",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "control_id": "detect-control-id",
    "detect": {
      "type": "machine",
      "params": {
        "event": "MACHINE",
        "beep": true
      }
    }
  }
}
```

#### Detection Event Values

| Event | Description |
|-------|-------------|
| `HUMAN` | Human voice detected |
| `MACHINE` | Answering machine detected |
| `UNKNOWN` | Unable to determine |
| `READY` | Machine greeting ended (if `detect_message_end` is true) |
| `NOT_READY` | Detection timeout |

---

### receive_fax / send_fax - status_url

Receives status events for fax operations.

#### Request

- **Method:** `POST`
- **Triggered:** When fax operation starts, progresses, or completes

#### Request Body

```json
{
  "event_type": "calling.call.fax",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "control_id": "fax-control-id",
    "fax": {
      "type": "received",
      "params": {
        "direction": "receive",
        "identity": "+15551231234",
        "remote_identity": "+15553214321",
        "document": "https://your-space.signalwire.com/api/v1/faxes/fax-uuid/download",
        "pages": 3,
        "success": true,
        "result": "success",
        "result_text": "OK"
      }
    }
  }
}
```

---

### sip_refer - status_url

Receives status events for SIP REFER operations.

#### Request

- **Method:** `POST`
- **Triggered:** When REFER is sent, progress is received, or operation completes

#### Request Body

```json
{
  "event_type": "calling.call.refer",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "refer_to": "sip:alice@example.com",
    "refer_result": "success",
    "refer_response_code": 202,
    "refer_to_response_code": 200
  }
}
```

---

### enter_queue - status_url

Receives status events for queue operations.

#### Request

- **Method:** `POST`
- **Triggered:** When caller enters queue, position changes, or leaves queue

#### Request Body

```json
{
  "event_type": "calling.call.queue",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "node_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "control_id": "queue-control-id",
    "queue_name": "support",
    "queue_id": "queue-uuid",
    "queue_state": "enqueue",
    "position": 3,
    "size": 5,
    "avg_time": 120,
    "enqueue_ts": 1640000000000,
    "dequeue_ts": null,
    "leave_ts": null
  }
}
```

#### Queue State Values

| State | Description |
|-------|-------------|
| `enqueue` | Caller entered the queue |
| `position` | Caller's position changed |
| `dequeue` | Caller was connected to an agent |
| `leave` | Caller left the queue (timeout, hangup, etc.) |

---

### enter_queue - wait_url

Fetches SWML to play while the caller waits in the queue.

#### Request

- **Method:** `POST`
- **Triggered:** When caller enters queue (loops until dequeued)

#### Request Body

Same as [Document Fetching Webhook](#document-fetching-webhook) with additional queue information in `vars`:

```json
{
  "call": { ... },
  "vars": {
    "queue_name": "support",
    "queue_position": 3,
    "queue_size": 5,
    "queue_avg_time": 120
  }
}
```

#### Expected Response

A SWML document with audio to play:

```json
{
  "version": "1.0.0",
  "sections": {
    "main": [
      { "play": "say:You are number %{vars.queue_position} in the queue. Estimated wait time is %{vars.queue_avg_time} seconds." },
      { "play": "https://example.com/hold_music.mp3" }
    ]
  }
}
```

---

### join_conference - status_callback

Receives status events for conference participation.

#### Request

- **Method:** `POST` (configurable via `status_callback_method`)
- **Triggered:** Based on `status_callback_event` setting

#### Request Body

```json
{
  "event_type": "calling.call.conference",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "conference_name": "my_conference",
    "conference_id": "conf-uuid",
    "conference_event": "join",
    "participant_count": 3
  }
}
```

#### Conference Event Values

| Event | Description |
|-------|-------------|
| `start` | Conference started |
| `end` | Conference ended |
| `join` | Participant joined |
| `leave` | Participant left |
| `mute` | Participant muted/unmuted |
| `hold` | Participant put on hold |
| `speaker` | Active speaker changed |

---

### join_conference - recording_status_callback

Receives status events for conference recording.

#### Request

- **Method:** `POST` (configurable via `recording_status_callback_method`)
- **Triggered:** Based on `recording_status_callback_event` setting (default: `completed`)

#### Request Body

```json
{
  "event_type": "calling.call.conference.recording",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "conference_name": "my_conference",
    "conference_id": "conf-uuid",
    "recording_state": "completed",
    "recording_url": "https://your-space.signalwire.com/api/v1/recordings/rec-uuid/download",
    "duration": 300,
    "size": 4800000
  }
}
```

#### Recording State Values

| State | Description |
|-------|-------------|
| `in-progress` | Recording has started |
| `completed` | Recording finished |
| `absent` | No recording was created |

---

### join_conference - wait_url

Fetches SWML to play while waiting for the conference to start.

#### Request

- **Method:** `POST` (configurable via `wait_method`)
- **Triggered:** When participant joins before `start_on_enter` participant

#### Request Body

Same as [Document Fetching Webhook](#document-fetching-webhook).

#### Expected Response

A SWML document with audio to play while waiting.

---

### ai - post_prompt_url

Receives AI agent completion data after the conversation ends.

#### Request

- **Method:** `POST`
- **Triggered:** When AI agent completes (call ends or agent transfers)
- **Authentication:** Basic auth if `post_prompt_auth_user` and `post_prompt_auth_password` are set

#### Request Body

```json
{
  "event_type": "calling.call.ai.complete",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "ai_session_id": "ai-session-uuid",
    "summary": "Customer called to inquire about their order status. Provided tracking information and estimated delivery date.",
    "post_prompt_result": "Summary of the conversation as requested in post_prompt",
    "end_reason": "user_hangup",
    "conversation": [
      {
        "role": "assistant",
        "content": "Hello! How can I help you today?"
      },
      {
        "role": "user",
        "content": "I want to check my order status"
      }
    ],
    "function_calls": [
      {
        "function": "get_order_status",
        "params": { "order_id": "12345" },
        "result": { "status": "shipped", "eta": "2024-01-15" }
      }
    ]
  }
}
```

---

### ai - SWAIG Includes Signature Discovery (get_signature)

When using `SWAIG.includes` to dynamically discover functions from external endpoints, SignalWire sends a signature discovery request to fetch available function definitions.

#### Configuration

```yaml
ai:
  SWAIG:
    includes:
      - url: "https://example.com/swaig"
        functions:
          - get_weather
          - check_inventory
      - url: "https://other-server.com/functions"
        functions:
          - send_notification
```

You can also use the defaults URL if `url` is omitted in an include block:

```yaml
ai:
  SWAIG:
    defaults:
      web_hook_url: "https://example.com/swaig"
    includes:
      - functions:
          - get_weather
          - check_inventory
```

#### Request

- **Method:** `POST`
- **Content-Type:** `application/json`
- **Triggered:** At AI agent initialization, before the conversation starts
- **Authentication:** Basic auth if credentials are embedded in URL or configured in defaults

#### Request Body

```json
{
  "version": "2.0",
  "action": "get_signature",
  "content_type": "text/swaig",
  "content_disposition": "function signature request",
  "functions": ["get_weather", "check_inventory"]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `version` | String | SWAIG protocol version, always `"2.0"` |
| `action` | String | Always `"get_signature"` for signature discovery |
| `content_type` | String | Content type identifier: `"text/swaig"` |
| `content_disposition` | String | Request description: `"function signature request"` |
| `functions` | Array | List of function names to discover |

#### Expected Response

Your server should respond with an array of function signature objects:

```json
[
  {
    "function": "get_weather",
    "purpose": "Get the current weather for a specified location",
    "argument": {
      "type": "object",
      "properties": {
        "location": {
          "type": "string",
          "description": "The city or location to get weather for"
        },
        "units": {
          "type": "string",
          "enum": ["celsius", "fahrenheit"],
          "description": "Temperature units"
        }
      },
      "required": ["location"]
    },
    "web_hook_url": "https://example.com/swaig/get_weather"
  },
  {
    "function": "check_inventory",
    "purpose": "Check product availability in inventory",
    "argument": {
      "type": "object",
      "properties": {
        "product_id": {
          "type": "string",
          "description": "The product ID to check"
        }
      },
      "required": ["product_id"]
    },
    "web_hook_url": "https://example.com/swaig/check_inventory",
    "web_hook_auth_user": "api_user",
    "web_hook_auth_password": "secret"
  }
]
```

#### Function Signature Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `function` | String | Yes | The function name that the AI will call |
| `purpose` | String | Yes | Description of what the function does (used by AI to decide when to call it) |
| `argument` | Object | Yes | JSON Schema defining the function's parameters |
| `web_hook_url` | String | Yes | URL to call when the AI invokes this function |
| `web_hook_auth_user` | String | No | Basic auth username for function invocation |
| `web_hook_auth_password` | String | No | Basic auth password for function invocation |
| `meta_data_token` | String | No | Token to include in function call metadata |

#### Argument Schema Format

The `argument` field uses JSON Schema to define function parameters:

```json
{
  "type": "object",
  "properties": {
    "param_name": {
      "type": "string|number|boolean|array|object",
      "description": "Description for the AI",
      "enum": ["option1", "option2"],
      "default": "default_value"
    }
  },
  "required": ["param_name"]
}
```

---

### ai - SWAIG Function Webhook (web_hook_url)

When the AI agent determines it needs to call a user-defined function (configured in the `SWAIG.functions` array or discovered via `SWAIG.includes`), it sends an HTTP POST request to the function's `web_hook_url`.

#### Request

- **Method:** `POST`
- **Content-Type:** `application/json`
- **Triggered:** When the AI decides to invoke the function based on conversation context
- **Authentication:** Basic auth if `web_hook_auth_user` and `web_hook_auth_pass` are configured

#### Request Body

```json
{
  "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
  "ai_session_id": "ai-session-uuid",
  "app_name": "swml",
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "action": "get_weather",
  "function": "get_weather",
  "argument": {
    "parsed": [
      {
        "location": "New York"
      }
    ],
    "raw": "New York",
    "substituted": "New York"
  },
  "meta_data": {
    "my_key": "some value"
  },
  "conversation_id": "conversation-uuid",
  "content_type": "text/swaig",
  "version": "2.0"
}
```

#### Request Fields

| Field | Type | Description |
|-------|------|-------------|
| `call_id` | String | Unique identifier for the call |
| `ai_session_id` | String | Identifier for the AI session |
| `project_id` | String | Your SignalWire project UUID |
| `space_id` | String | Your SignalWire space UUID |
| `action` | String | The function name being invoked |
| `function` | String | The function name (same as action) |
| `argument` | Object | The arguments extracted by the AI |
| `argument.parsed` | Array | Structured argument data as JSON objects |
| `argument.raw` | String | Raw argument string from AI |
| `argument.substituted` | String | Argument with variable substitutions applied |
| `meta_data` | Object | Custom metadata from `SWAIG.defaults.meta_data` |
| `conversation_id` | String | Identifier for the conversation thread |

#### Expected Response

Your webhook should respond with the function result that the AI will use to continue the conversation:

```json
{
  "response": "The current weather in New York is 72°F with partly cloudy skies.",
  "action": [
    {
      "say_greeting": "Let me check the weather for you."
    }
  ]
}
```

#### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `response` | String | The result text that the AI will use to formulate its reply |
| `action` | Array | Optional array of actions for the AI to perform |

#### Action Types

The `action` array can contain various directives for the AI:

**Speech & Audio Actions:**

| Action | Description | Example |
|--------|-------------|---------|
| `say` | Text for the AI to speak immediately | `{"say": "Please hold while I transfer you."}` |
| `playback_bg` | Play audio file in background | `{"playback_bg": {"file": "hold_music.mp3", "wait": true}}` |
| `stop_playback_bg` | Stop background audio playback | `{"stop_playback_bg": true}` |

**Function Control Actions:**

| Action | Description | Example |
|--------|-------------|---------|
| `toggle_functions` | Enable/disable specific functions | `{"toggle_functions": [{"function": "get_balance", "active": false}]}` |
| `back_to_back_functions` | Allow consecutive function calls without user input | `{"back_to_back_functions": true}` or `"forever"` |
| `functions_on_speaker_timeout` | Trigger functions when speaker times out | `{"functions_on_speaker_timeout": true}` |

**Data & Metadata Actions:**

| Action | Description | Example |
|--------|-------------|---------|
| `set_meta_data` | Set session metadata (scoped by token) | `{"set_meta_data": {"order_id": "12345"}}` |
| `unset_meta_data` | Remove session metadata | `{"unset_meta_data": "order_id"}` or `["key1", "key2"]` |
| `set_global_data` | Set global data (requires permission) | `{"set_global_data": {"customer_tier": "premium"}}` |
| `unset_global_data` | Remove global data | `{"unset_global_data": "customer_tier"}` |

**Context & Prompt Actions:**

| Action | Description | Example |
|--------|-------------|---------|
| `context_switch` | Switch to new context/prompt | `{"context_switch": {"system_prompt": "You are now a billing agent.", "user_prompt": "Help with billing."}}` |
| `user_input` | Inject text as user input | `{"user_input": "I need help with my order"}` |

**Call Control Actions:**

| Action | Description | Example |
|--------|-------------|---------|
| `transfer` | Transfer the call to destination | `{"transfer": {"dest": "sip:agent@example.com", "summarize": true}}` |
| `hangup` | End the call | `{"hangup": true}` |
| `hold` | Put call on hold | `{"hold": 60}` or `{"hold": {"timeout": 120}}` |
| `stop` | Stop the AI agent | `{"stop": true}` |

**SWML Execution Actions:**

| Action | Description | Example |
|--------|-------------|---------|
| `SWML` | Execute SWML document | `{"SWML": {"version": "1.0.0", "sections": {...}}, "transfer": true}` |
| `user_event` | Fire custom user event | `{"user_event": {"event_type": "custom", "data": {...}}}` |

**Timing & Behavior Actions:**

| Action | Description | Example |
|--------|-------------|---------|
| `wait_for_user` | Wait for user to speak first | `{"wait_for_user": true}` or `"answer_first"` |
| `end_of_speech_timeout` | Change end of speech timeout (ms) | `{"end_of_speech_timeout": 2000}` |
| `speech_event_timeout` | Change speech event timeout (ms) | `{"speech_event_timeout": 5000}` |
| `settings` | Modify AI settings (requires permission) | `{"settings": {"temperature": 0.7}}` |
| `extensive_data` | Enable extensive data mode | `{"extensive_data": true}` |

**Step/Context Navigation:**

| Action | Description | Example |
|--------|-------------|---------|
| `change_context` | Change to named context | `{"change_context": "billing_flow"}` |
| `change_step` | Change to named step | `{"change_step": "collect_payment"}` |

#### Complete Action Example

```json
{
  "response": "I found your order. It shipped yesterday and should arrive by Friday.",
  "action": [
    {
      "set_meta_data": {
        "order_status": "shipped",
        "tracking_number": "1Z999AA10123456784"
      }
    },
    {
      "toggle_functions": [
        {
          "function": "check_order_status",
          "active": false
        }
      ]
    }
  ]
}
```

---

### ai - SWAIG Data Map Webhook

For functions configured with `data_map.webhooks`, the AI can fetch dynamic data during function execution.

#### Request

- **Method:** Configurable (`GET` or `POST`)
- **Triggered:** When function uses `data_map.webhooks` configuration

#### Request Body (POST method)

```json
{
  "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
  "vars": {
    "location": "New York"
  }
}
```

#### Configuration Example

```json
{
  "SWAIG": {
    "functions": [
      {
        "function": "get_weather",
        "purpose": "Get current weather for a location",
        "argument": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "City name"
            }
          }
        },
        "data_map": {
          "webhooks": [
            {
              "url": "https://api.example.com/weather?city=%{args.location}",
              "method": "GET",
              "output": {
                "response": "The weather in %{args.location} is %{temp}°F with %{conditions}."
              }
            }
          ]
        }
      }
    ]
  }
}
```

---

### pay - status_url

Receives status events during payment processing.

#### Request

- **Method:** `POST`
- **Triggered:** When payment process starts, progresses, or completes

#### Request Body

```json
{
  "event_type": "calling.call.pay",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "pay_state": "collecting_card_number",
    "attempts": 1
  }
}
```

#### Pay State Values

| State | Description |
|-------|-------------|
| `collecting_card_number` | Collecting card number |
| `collecting_expiration_date` | Collecting expiration date |
| `collecting_security_code` | Collecting CVV |
| `collecting_postal_code` | Collecting postal code |
| `processing` | Processing payment |
| `completed` | Payment completed |
| `failed` | Payment failed |
| `canceled` | Payment canceled |

---

### pay - payment_connector_url

Receives payment data for processing by your payment gateway.

#### Request

- **Method:** `POST`
- **Triggered:** When all payment details have been collected

#### Request Body

```json
{
  "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
  "payment_method": "credit-card",
  "card_type": "visa",
  "card_number": "4111111111111111",
  "expiration_date": "12/25",
  "security_code": "123",
  "postal_code": "12345",
  "charge_amount": "10.55",
  "currency": "usd",
  "token_type": "one-time",
  "description": "Monthly subscription",
  "parameters": [
    { "name": "customer_id", "value": "cust-12345" }
  ]
}
```

#### Expected Response

Your payment connector should respond with the result:

```json
{
  "charge_id": "ch_1234567890",
  "payment_confirmation_code": "ABC123",
  "result": "success",
  "error_code": null,
  "error_message": null
}
```

---

### Stream Device - status_url (in connect)

When connecting to a bidirectional WebSocket stream device, receives status events.

#### Request

- **Method:** `POST` (configurable via `status_url_method`)
- **Triggered:** When stream starts or ends

#### Request Body

```json
{
  "event_type": "calling.call.stream",
  "event_channel": "signalwire",
  "timestamp": 1640000000.123,
  "project_id": "project-uuid",
  "space_id": "space-uuid",
  "params": {
    "call_id": "b5f2c3d4-e6f7-8901-a2b3-c4d5e6f78901",
    "stream_name": "my-stream",
    "stream_state": "connected",
    "stream_url": "wss://example.com/audio"
  }
}
```

#### Stream State Values

| State | Description |
|-------|-------------|
| `connecting` | Stream is connecting |
| `connected` | Stream is active |
| `disconnected` | Stream has ended |
| `error` | Connection error |

---

### Request Verification

For external webhooks (non-SignalWire URLs), requests include an HMAC signature in the `X-SignalWire-Signature` header that you can use to verify the request authenticity.

To verify:
1. Concatenate the timestamp and request body
2. Compute HMAC-SHA256 using your API token as the key
3. Compare with the signature header value

---
