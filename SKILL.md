---
name: swml
description: Craft SWML (SignalWire Markup Language) documents for voice and video call control and messaging. Use when building IVR menus, AI agents, call routing, recording, conferencing, or any telephony automation.
allowed-tools: Read, Grep, Glob
license: MIT
---

# SWML (SignalWire Markup Language)

SWML is a JSON/YAML-based language for controlling voice and video calls and messages on SignalWire. A SWML document describes a call flow — answering, playing audio, collecting input, connecting calls, joining video rooms, launching AI agents, and more. Documents can be served from a webhook or defined inline.

## Document Structure

```yaml
version: 1.0.0
sections:
  main:
    - answer
    - play: "say:Hello world"
    - hangup
```

- **version**: Semantic version string (use `"1.0.0"`)
- **sections**: Map of named sections. `main` is always executed first.
- Each section is an array of method calls executed in order.
- **Compact form**: A bare array `[...]` is treated as a single `main` section (no `version`/`sections` wrapper needed).

## Expression Evaluation

All text wrapped in `%{}` is evaluated as JavaScript:

- `%{vars.my_var}` — access script variables
- `%{call.from}` — access call object fields (read-only)
- `%{call.headers.X-Custom}` — access SIP headers
- `%{params.my_param}` — access subroutine parameters

Use `set` / `unset` methods to manage variables. Variables cannot be permanently changed inside `%{}` expressions.

## Method Format

Methods can be specified in four forms:

```yaml
# 1. Bare string (no params)
- answer

# 2. Single parameter shorthand
- play: "say:Hello"

# 3. Array parameter shorthand
- play:
    - "say:Hello"
    - "silence:1"
    - "say:Goodbye"

# 4. Object with named parameters
- play:
    urls:
      - "say:Hello"
    volume: 10
    say_voice: Polly.Matthew
```

Every method call can include an optional `label` field for use with `goto`.

## Method Summary

### Script Control Flow & Variables
| Method | Description |
|--------|-------------|
| `goto` | Jump to a label (with optional `when` condition, `max` iterations) |
| `label` | Define a jump target |
| `set` | Set variable(s): `{var: value}` |
| `unset` | Delete variable(s) |
| `switch` | Branch on variable value (`case` / `default`) |
| `cond` | Conditional array: `[[expr, [actions]], ..., [default_actions]]` |
| `if` | If/then/else: `{condition, then, else}` |
| `execute` | Call a subroutine section or URL (`dest`, `params`, `result`) |
| `return` | Return a value from a subroutine |
| `transfer` | Transfer execution to a new SWML URL |

### Call Control & Bridging
| Method | Description |
|--------|-------------|
| `answer` | Answer the call (optional `max_duration` in seconds) |
| `hangup` | End the call (optional `reason`: `hangup`, `busy`, `decline`) |
| `connect` | Dial phone/SIP/queue/stream (`serial`, `parallel`, `serial_parallel`, or `to`) |
| `dial` | (EXPERIMENTAL) Similar to connect |
| `join_room` | Join a video room by name |
| `join_conference` | Join an audio conference by name |
| `enter_queue` | Place caller into a named queue |
| `sip_refer` | SIP REFER to transfer call |

### IVR (Interactive Voice Response)
| Method | Description |
|--------|-------------|
| `play` | Play audio files, TTS, ringtones, or silence |
| `prompt` | Play audio and collect digit/speech input |
| `record` | Record audio from caller |
| `record_call` | Record the full call (both legs) |
| `stop_record_call` | Stop an active call recording |
| `send_digits` | Send DTMF digits |
| `sleep` | Pause execution for N seconds |
| `echo` | Echo audio back to caller (with optional timeout) |
| `detect_machine` | Detect answering machine/voicemail |
| `pay` | Collect payment card info |
| `bind_digit` | (EXPERIMENTAL) Bind digit to trigger actions |
| `clear_digit_bindings` | (EXPERIMENTAL) Clear digit bindings |

### Media Processing
| Method | Description |
|--------|-------------|
| `denoise` | Enable noise reduction |
| `stop_denoise` | Disable noise reduction |
| `live_transcribe` | Start live transcription |
| `live_translate` | Start live translation |
| `stream` | Start media stream to WebSocket URL |
| `stop_stream` | Stop an active media stream |
| `transcribe` | Transcribe audio |
| `transcribe_stop` | Stop transcription |

### AI
| Method | Description |
|--------|-------------|
| `ai` | Connect to an AI agent (inline config or agent ID) |
| `amazon_bedrock` | Connect to Amazon Bedrock AI |
| `tap` | Tap media stream to a URI |
| `stop_tap` | Stop media tap |
| `user_event` | (EXPERIMENTAL) Send custom event |

### Faxing
| Method | Description |
|--------|-------------|
| `receive_fax` | Receive an incoming fax |
| `send_fax` | Send a fax document |

### Messaging
| Method | Description |
|--------|-------------|
| `send_sms` | Send an SMS message |

### External APIs
| Method | Description |
|--------|-------------|
| `request` | Make an HTTP request (`url`, `method`, `headers`, `body`) |
| `execute_rpc` | (EXPERIMENTAL) Execute an RPC call |

## Common Patterns

### Answer and Play TTS

```yaml
version: 1.0.0
sections:
  main:
    - answer
    - play: "say:Welcome to our service. Please hold."
    - hangup
```

### IVR Menu with Digit Input

```yaml
version: 1.0.0
sections:
  main:
    - answer
    - label: menu
      prompt:
        play:
          - "say:Press 1 for sales, 2 for support."
        max_digits: 1
        digit_timeout: 5.0
    - switch:
        variable: prompt_value
        case:
          "1":
            - transfer: "https://example.com/sales.swml"
          "2":
            - transfer: "https://example.com/support.swml"
        default:
          - play: "say:Invalid selection."
          - goto:
              label: menu
              max: 3
    - play: "say:Goodbye."
    - hangup
```

### Subroutines with execute/return

```yaml
version: 1.0.0
sections:
  main:
    - answer
    - execute:
        dest: greeting
        params:
          name: "caller"
    - hangup

  greeting:
    - play: "say:Hello %{params.name}, welcome!"
    - return: "done"
```

### AI Agent (Minimal)

```yaml
version: 1.0.0
sections:
  main:
    - answer
    - ai:
        prompt:
          text: "You are a helpful assistant for Acme Corp. Help callers with their questions."
        post_prompt:
          text: "Summarize the call."
        post_prompt_url: "https://example.com/summary"
    - hangup
```

### Connect / Dial Out

```yaml
version: 1.0.0
sections:
  main:
    - answer
    - play: "say:Please hold while we connect you."
    - connect:
        from: "+15551230000"
        to: "+15559876543"
        timeout: 30
        result:
          case:
            connected:
              - play: "say:Goodbye."
              - hangup
          default:
            - play: "say:Sorry, no one is available. Please try again later."
            - hangup
```

### HTTP Request

```yaml
version: 1.0.0
sections:
  main:
    - answer
    - request:
        url: "https://api.example.com/lookup"
        method: POST
        headers:
          Content-Type: application/json
        body:
          caller: "%{call.from}"
    - play: "say:Your account status is %{vars.request_result}"
    - hangup
```

## Reference Lookup

This skill includes two detailed reference files in its directory:

- **`swml_reference.md`** — Full prose documentation with parameter details and examples for every SWML method (~5000 lines)
- **`schema.json`** — JSON Schema with exact types, required fields, enums, and `$defs` for every SWML structure (~12000 lines)

When you need detailed parameter information, validation rules, or exact type specifications for any SWML method, use `Grep` and `Read` to search these files in the skill directory. For example:

- To find `play` method details: `Grep` for `### play` in `swml_reference.md`
- To find the AI agent schema: `Grep` for `"ai"` in `schema.json`
- To find all parameters for `connect`: `Grep` for `### connect` in `swml_reference.md`
