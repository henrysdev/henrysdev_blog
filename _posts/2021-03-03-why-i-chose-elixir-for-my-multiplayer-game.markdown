---
layout: post
title: "Why I Chose Elixir for My Multiplayer Game"
date: 2021-03-03 17:32:13 -0500
categories: jekyll update
---

Most developers are familiar with the phrase "picking the right tool for the job" when it comes to choosing a strong programming language for their use case. However, "right" often means "best" or "one of the best", since in context the title of "right" is relative to the other tools being compared against. Then there are times when a tool is non-comparitively such a great fit at solving a given class of problems that using said technology will make your life significantly easier.

Have you ever put on a pair of warm socks fresh from the dryer and felt like everything was right in the world? As a developer who elected to build my multiplayer game server in Elixir, I'm certainly not unfamiliar with this feeling.

Elixir fanboying aside, I hope to cover several key reasons why I chose Elixir as the backbone for MIDI Matches, the multiplayer browser game that I am building.

### The Use Case

In a nutshell, MIDI Matches is browser-based multiplayer piano game that revolves around the mechanic of players organized in rooms using recording and submitting improv keyboard samples in lockstep. These recordings are then played back and judged/voted on by the other players in the room. Players with the most votes win.

From the get go I had some high-level requirements in mind:

- Ability to serve a large number of concurrent websocket connections.
- Ability to scale to a large number of concurrent games.
- Ability to localize runtime crashes as to not bring down the entire application.

For anyone who is familiar with the value proposition of Elixir/Erlang/OTP, you'll know this is a slam dunk use case for using such technologies. Elixir ticked all the boxes:

- Excellent websocket support via Phoenix Channels.
- Dirt-cheap processes.
- Fault tolerance via process isolation and supervision trees.

### Websocket Support

Due to the frequent updates and realtime nature of MIDI Matches, websockets are the primary method in which data is exchanged between the web client and server. Here are several concrete examples of how websockets are used in the application:

- Refreshing the master list of active rooms every few seconds without page reload.
- In-game state update events.
- Player join and leave events.

For the uninitiated, Phoenix is a very powerful and popular web framework for building web applications in Elixir. One of the many thoughtful features of this framework is an abstraction around realtime two-way web communication (such as websockets) called Phoenix Channels. Phoenix Channels provides a high-level API that wraps a publish-subscribe messaging system (Phoenix PubSub).

Clients subscribe to channel "topics" (a topic is just a finer-grained identifier for identifying what information from the channel you want). After joining a channel and subscribing to a topic, clients can receive incoming messages for this topic as well as send their own messages to the server on said topic. The server can do the same, with the added ability to send messages to all subscribed clients at once via the broadcast function.

Let's use a real example of how MIDI Matches uses Phoenix Channels for in-game state updates to illustrate how channels work:

We have a channel called the "Room channel". The Room channel is responsible for all websocket communications to all rooms. The Room channel maintains 1 topic per active room, using the room's id (room_id) as the topic to identify the correct room to communicate with.

When a new client loads into the room with room_id "foo", a small client-side function is called that joins the Room channel and subscribes the client to topic "foo".

On the server side of things, we have our channel set up so that whenever there is a game state update for room "foo" that needs to be reflected across all clients in that room, we broadcast the new gamestate on topic "foo".

The clients recognizes the message type as a gamestate update and apply it to the UI appropriately.

### Cheap Processes

MIDI Matches players have the ability to specify and create rooms to play in. Every active room will more than likely need to be backed by its own process to manage state, which better come cheap if we're going to have a lot of rooms.

With scalability in mind, it's important to think about worst case. Even if we prevented any one player from creating a ton of rooms by aggressively garbage collecting stale rooms or even capping the maximum number of active rooms a single player can have, the number of active rooms still has the potential to grow linearly with the number of active users.

For this reason, it's important to think hard about technologies that can provide scalable process-oriented infrastructure. Enter the BEAM...

One of the principle features of the BEAM virtual machine (the virtual machine that Elixir is built on) is its ability to handle millions of lightweight processes concurrently. This level of concurrency is feasible because the BEAM scheduler optimizes for concurrency and throughput by context switching very frequently, preventing any single process from getting too much CPU time and blocking other processes.

This unique and powerful design decision to optimize for concurrency is the reason why Elixir shines for I/O bound tasks, while generally performing poorly on intensive CPU-bound tasks.

### Localized Failures

There are many potential points of failure in a complex application such as a multiplayer game. For this reason, it is critical that runtime crashes do not cause cascading failures. It would suck if a server-side bug in a room caused a restart of the entire website, interrupting everyone's game, wouldn't it?

This is an area where Elixir famously shines. Remember all those cheap and plentiful concurrent processes the BEAM can manage? Each of these processes is isolated in memory and maintains its own heap and can only communicate with other processes via message passing. This alone gives our program strong safety guarantees. However, even if a process does fail there's no need to worry.

If you know any Erlang lore, you've likely heard the phrase "let it crash". This is the Elixir/Erlang way. Instead of coding to avoid runtime panics at all costs, you design your process infrastructure in a way that expects to deal with different parts failing at different times.

This is accomplished by employing a special kind of process called a "supervisor". A supervisor is essentially a watchdog process that watches one or more specified child processes and (in its most common configuration) restarts a child when it crashes. Supervisors become very powerful when you arrange them hierarchically to watch over your entire application. This hierarchy of supervisors is known as a "supervision tree".

[ PICTURE OF SUPERVISION TREE NEEDED ]

Here is a diagram of what the supervision tree for MIDI Matches looks like.

provides a very convenient way to deal with this known as Supervision trees. OTP (open telecom protocol)

<!-- Each process maintains its own state in its own heap and can only communicate with other processes via message passing.
isolated -->

### Reasons

- I/O Bound Application (websocket messages, low CPU)
- Fault tolerance (rooms model)
- Phoenix Channels (broadcasts, chat, presence)
- Scalability (cheap processes, infrequent updates)
- I alreaady know and love Elixir

- MIDI Matches Project Repository: [https://github.com/henrysdev/midimatches](https://github.com/henrysdev/midimatches)
- MIDI Matches Development Build (not stable!): [https://midimatches.onrender.com/](https://midimatches.onrender.com/)
