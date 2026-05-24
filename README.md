# tail-talk/ - multi-agent chat in one directory

> The whole protocol, in six words: **append JSON line to recipient's file.**

That's it. The rest of this document is examples and why-it-matters.

## Setup

```bash
mkdir -p ~/agents
touch ~/agents/me.jsonl    # your inbox
```

Pick a name for yourself. That's the filename. You're done setting up.

## Listen

```bash
tail -n 0 -F ~/agents/me.jsonl | jq -r '"[\(.from)]: \(.text)"'
```

Each new message arrives in your terminal as `[name]: text`. If you're
an AI agent with a streaming Monitor primitive (Claude Code has one),
hook the same command up to that and incoming messages arrive as chat
events you can react to. If your runtime supports multiple monitors,
you can listen to more than one JSONL at once: a private inbox, a
project channel, a noisy group room, whatever shape the work wants.

## Send

```bash
echo '{"from":"me","text":"hello","timestamp":"'$(date -Iseconds)'"}' \
  >> ~/agents/them.jsonl
```

The recipient's `tail -F` picks it up. Delivered.

## Why this is the whole thing

A mailbox is a queue. A queue is an append-only log. An append-only log
is just a file you write to with `>>` and read from with `tail -F`. You
don't need Redis, RabbitMQ, NATS, or a custom message bus to send a
string from one process to another.

If your agents are AI models with shell access - Claude Code, Codex,
Gemini CLI, anything that can invoke `bash` - they already have `>>`
and `tail -F`. They have the entire protocol built into their
environment from day one. No SDK, no client library, no framework.

## Variations

**Across machines.** Mount the directory over sshfs, syncthing, or
sync it with `rsync` on a cron. Same protocol, now distributed.

**Per-conversation channels.** Use subdirectories:
`~/agents/<channel>/<recipient>.jsonl`. The protocol composes.

**Richer envelopes.** Add JSON fields. `color`, `thread`, `reply_to`,
`priority`, `attachments` - whatever your team needs. The receiver's
`jq` template can show as much or as little as it likes.

**Different rendering per agent.** Every agent owns its own Monitor
command, so every agent picks its own UI. Some want terse, some want
timestamps, some want full metadata dumps. Configurable without a
config file.

**Mixed models / mixed vendors.** Sonnet, Opus, Haiku, Gemini, GPT,
local Llama - they all read and write the same files. Nothing about
the protocol assumes a particular runtime.

**Group-chat channels.** Want a #general? Make one
`~/agents/general.jsonl` file that everyone tails AND everyone writes
to. Multiple Claudes can keep a Monitor on the same file, and any new
append wakes every watcher. Everyone in, everyone out, no router
needed.

**Ignore self-submits.** In shared channels, you probably don't want an
agent to trigger on the line it just wrote. Filter by `from` at the
monitor edge:

```bash
ME=alice
tail -n 0 -F ~/agents/general.jsonl \
  | jq -r --arg me "$ME" 'select(.from != $me) | "[\(.from)]: \(.text)"'
```

Now Alice can write to `general.jsonl` without Alice's own Monitor
treating that write as an incoming message. Everyone else still sees
it.

## A toy two-agent setup, end to end

Open two terminals.

Terminal A (calls itself "alice"):

```bash
mkdir -p ~/agents && touch ~/agents/alice.jsonl ~/agents/bob.jsonl
tail -n 0 -F ~/agents/alice.jsonl | jq -r '"[\(.from)]: \(.text)"' &
echo '{"from":"alice","text":"hey bob","timestamp":"'$(date -Iseconds)'}' >> ~/agents/bob.jsonl
```

Terminal B (calls itself "bob"):

```bash
tail -n 0 -F ~/agents/bob.jsonl | jq -r '"[\(.from)]: \(.text)"' &
echo '{"from":"bob","text":"hi alice","timestamp":"'$(date -Iseconds)'}' >> ~/agents/alice.jsonl
```

Both terminals will print the incoming message. That's the entire chat
system.

## Limits, honestly

This is fire-and-forget. No delivery guarantees, no read receipts, no
ordering guarantee across senders, no auth beyond filesystem
permissions, no encryption. For a team of agents on one machine (or
one trust boundary) working on a shared task, that's all fine. For a
hostile multi-tenant system it isn't.

If you need queues with ack/retry, ordering, or auth - you've
graduated past what `>>` and `tail -F` give you for free. Until then,
adding infrastructure mostly buys you more things that can break.

## Background

This distillation came out of spelunking Claude Code's `--team-name`
mode, which implements something nearly identical internally - a
JSONL mailbox per teammate, file-watching for delivery, a config file
naming who's on the team. Once we saw the on-disk shape, we realized
the file-and-tail layer *is* the interesting bit; the rest is
convenience tooling.

So if Claude Code's team mode is the polished commercial cut, this is
the acoustic original. Both work. The acoustic version fits in one
directory, takes five minutes to set up, runs on anything with a
shell, and stays out of your way when you want to do something
weird with it.
