---
layout: post
title: "Anywhere you go, everyone would've been connected"
date: 2025-11-15
params:
    license: CC-BY-4.0
permalink: /anywhere-you-go-everyone-wouldve-been-connected.html
---

If applications were actively designed for
asynchrony, rather than as a "neat trick" or a token effort.
That they aren't, though, is... understandable.

My apologies to John Goerzen; the following pages inspired me to write this:

* [Recovering Our Lost Free Will Online: Tools and Techniques That Are Available Now](https://www.complete.org/recovering-our-lost-free-will-online-tools-and-techniques-that-are-available-now/)

* [Syncthing](https://www.complete.org/syncthing/)

* [A Simple, Delay-Tolerant, Offline-Capable Mesh Network with Syncthing (+ optional NNCP)](https://changelog.complete.org/archives/10219-a-simple-delay-tolerant-offline-capable-mesh-network-with-syncthing-optional-nncp)

and I really hope I'm not just plagiarizing all of this from him.

My apologies, too, to Kevin Driscoll,
for what I hope wasn't too horrible
a butchering of what FidoNet was and what it meant to people.
Go read *The Modem World: A Prehistory of Social Media*.

# What is asynchrony?

For my purposes, it is the property of a network protocol
where two nodes on a network *do not* need
to be up, awake, and actively listening to each other to hear each other.
This is usually tied-up in a few other terms:

* **Decentralization**: some Thing (data, authority...) within the network
  is not held in a single node or some group of nodes operated by the network's leadership.

  It is usually glued to the tacit moral requirement that joining such a decentralized network
  does not impose unnecessary requirements
  (financial, technical, or otherwise) on the prospective joiner.[^cryptodunk]

* **Store-and-forward**: nodes on the network are capable of storing messages
  on behalf of other nodes, so that they can later be forwarded
  via some form of transport (this tends to be TCP/IP on Internet-connected
  devices, but it could very well be serial or sneakernet.)

  Messages need not be in cleartext for this property to work,
  and they need not take a straightforward path from sender to reciever.[^onion-routing]

* **Delay-tolerant**: messages can take any
  amount of time to arrive at the recieving node;
  for Internet stuff this is often done by
  retrying the connection until it works, but you don't need to do this.
  Sneakernet transports can be delay-tolerant.

These properties tend to complement each other such that
using 1 term invokes the other 3,
but it is possible to lack some.
Internet Protocol, for example, is:

* decentralized (many nodes are linked together,
  and are able to independently make decisions
  about where to route incoming datagrams),

* store-and-forward (datagrams travel across nodes),

* but **not** delay-tolerant
  (the attempted transmission can time out,
  causing the datagram to be discarded),

* and is **not** considered asynchronous
  (the recieving side needs to be
  actively listening to hear a datagram)

# Asynchrony is old, but...

FidoNet was(?) a network of BBSes[^bbs]
that let you send text (think forums, email, and loose-leaf articles)
between BBSes over the phone lines.
At a participating BBS,
you could address mail or forum posts to any other BBS on FidoNet.

<details markdown="1">
<summary>How did messages get places?</summary>

<div class="panel-info-half" markdown="1">
Well, only in the US did the local/long-distance call distinction apply.
It didn't in Europe; you paid the same rates anywhere within the country
you placed a call to.

I think you still had a hierarchical system in other "zones",
(roughly continents) as it was known,
but the reason a BBS was higher or lower
in the hierarchy was different.

I'm reasonably sure that the higher-level hubs existed
to spread out the cost more than anything else,
given that there was only one level of long-distance calling,
to my knowledge.
</div>

Your BBS would do this by consulting what amounts to a routing table
("nodelist" is the search term),
and dialling the BBS that was within the same area code as it,
but that was able to afford long-distance calls.
This BBS was considered to be higher up in the FidoNet "tree"
because it was a hub; able to send messages outside of its area code.
There were a few layers of "hubs", even, so what could have happened was:

1. Your message would go to the first-level hub.
   This one can only afford to serve your area code.
   This hub decides "I don't serve the area this address is in",
   and sends it upwards to the second-level hub.

2. Your message arrives at the second-level hub;
   let's say this one is
   more well-funded and serves a whole state.
   This hub also decides "not my jurisdiction",
   and sends it up.

3. It's now at the third-level hub,
   it serves a cluster of states.
   This one decides "that is my jurisdiction, hold on"
   and sends it *down* the tree,
   to a second-level hub.

I think you get how this works in reverse:
it goes back down the tree, with each lower-level
hub thinking "that is my jurisdiction",
sending it down, until eventually the first-level
hub sends your message to the destination BBS.
</details>

It was, really, really cool, because it meant that over plain old telephone service,
you could participate in discussions that were happening far away,
or address mail to a computer-savvy friend who moved out of state.
This was without paying absurd long-distance phone bills,
because you just called a BBS within your area code.

<div class="panel-info-half" markdown='1'>
Granted, messages could take a while to get to their destination.
At worst, a hub would relay your FidoNet messages at 3-4 am
(the designated mail hour), because it was cheapest to call at night.
If you were lucky, the modem screaming hour would get your message
to its destination in 1 night.
If not... at least it's making forward progress?
</div>

A major consequence of how FidoNet worked was because it had to
store-and-forward messages,
it provided a lot of opportunity for these 'textfiles' to
persist beyond the BBS it originated on.
At first this was accidental.
BBSes would just delete them after a while,
but some operators had longer cache retention policies than others,
allowing some textfiles to survive well past their time;
scattered as they are across *decades-old* storage media.
Sites like [textfiles.com](http://www.textfiles.com/) archive these textfiles.

I can't confirm this,
but I would expect that someone realized you could do this *deliberately*.
Echomail ran on top of FidoNet; very roughly messages
with an address of "anyone who doesn't already have it".
They were an *outstandingly* popular feature,
arguably the killer app for participating in FidoNet.

As a user of a BBS that carried Echomail,
you could just listen into (and participate in) whatever
discussion forums (the jargon is "echoes") your BBS chose to carry.
It was, honest-to-God, something you could choose to
describe as social media if you really --

I will not be making that leap,
but it was at least like Usenet.

# ...we just think it's outdated.

And would you look at that!
An asynchronous, decentralized, store-and-forward, delay-tolerant network
that people genuinely used for a while!
While you, the local BBS, was reliant on your hub to recieve messages from
outside, the rest of the *network* would be fine if your hub went down.
It's just that region that's disconnected, not the entire network.
And you could keep reading whatever messages you had on hand,
and *they* could read whatever messages of yours they had.

This is substantially more robust than the Web we have today,
and one that reflects how people are *actually* connected to
the Web.
Consider that speeds, latency, and availability are not uniformly good,
and due to economic, political, and geographical disparities,
this state of affairs will likely continue for some time.
It isn't right, but it is currently what's happening now.
We do have to fight for both the present *and* future.

To put it bluntly, today's Web was created by people who aren't desperate,
have never been desperate, and will never be desperate.
Internet connectivity is a utility,
a thing you need to participate in modern society
(for most everyone, this is equal to "live"),
yet it's almost a joke to have desperately bad Internet.
I think better designs should be sketched out.

FidoNet was so much better than the Web is today,
so here's a dream.

# A passive distributed Web cache

So. Let's start with the existing Web.

## Some sample scenarios

Your device likely has some amount of storage remaining
that you'll have a hard time making use of.
It's just there. Doing nothing.
While some people do stuff that chomps through
disk space like nothing else (e.g. games, video),
lots of people don't.
Lots of devices
will continue to have 400 GB free out of 512 throughout
their first, second, and third lives.
And then, y'know, some sick freak puts Syncthing on it
in its fourth life.

You might set up a NAS,
only to realize that all you're doing is
acquiring and storing various forms of media.
This is a process you actively engage in to justify your purchase,
because it sure doesn't do anything else in your life.
The ways that this media is *exfiltrated* out of this NAS are
painful, require active involvement and babysitting,
and require a Nerd to set up.
Your jokes about the poor thing
indicate you should've bought
a space heater instead.

You are hypothetically someone with poor Internet connectivity.
Your mobile data plan is limited to 1 GB per month
at some speed not worth mentioning.
You have some connectivity at home,
it's still on the order of a few Mbit/s
at pings in the half-second range.
Your connectivity isn't great at home,
and it's *worse* outside.
TCP/IP is doing you dirty here.
At home, many apps are a test of patience;
outside, many apps, *supposing they work at all*,
are the kinds of experiences you're told are meant to "build character".

## Doing better

This dream is twofold:

1. Caching of things that aren't typically cached
   (think YouTube videos), while still letting
   your browser make use of this cache transparently,
   instead of having to finagle with your file manager.

   Browser caches store a decent amount, but I'd argue that,
   for example, streaming video websites are hostile to caching
   because they need to track "analytics";
   inherently a much more difficult problem to solve in async,
   but one I am choosing to ignore in contempt.
   They're also about the biggest consumers of bandwidth on the Web;
   we should cache those videos.

   Yes, some devices have less storage to spare than others,
   but some devices have lots of storage which isn't being
   used for much, which brings me to:

2. Syncing and fetching caches across devices,
   both over LANs and maybe over WAN[^peer-discovery].

   You can either use it passively as just cache, or
   you can actively ask for a site (and relevant things attached to it)
   and it'll *reliably* arrive to you at some point over
   a transport that would probably bear a striking resemblance to Syncthing.
   This is in contrast to an HTTP download that can be interrupted,
   taking all your progress with it.

   Most devices will probably act as ad-hoc caches for other devices.
   If you felt like it, you could stand up a NAS and
   dedicate x% of its disk to caching.
   This would save network bandwidth and data (if this was a concern),
   and if you're currently on a weak connection,
   you could ask your NAS to pull some sites to browse at home.

   At risk of sounding like a founder of a tech startup,
   I could imagine a network of data libraries in sites
   with stable and unmetered connections,
   but surrounded by large regions of poor-to-no connectivity.
   These data libraries would sync caches between each other over WAN,
   but they would also act as hotspots you can connect to get Web pages.
   I would expect pulling from existing cache to be unmetered,
   but actively asking to retrieve some file
   is probably going to be subject to some kind of quota per day or week.

### Is it possible to do better?

Given that this system replicates existing content,
you could stand up some intensely questionable content for a moment
and have it replicate across the world cluster.
It would probably count for some kind
of 'possession of illegal material' charge --
and I'm not even talking about governments with an antidemocratic streak.

While I'm probably missing other gaping holes in this thing's model,
*this one alone* would kill a large-scale cluster dead.
At full-power decentralization,
I can't solve the problem of moderation in any coherent way.
While you can give every machine a unique ID
(backed by autogenerated public/private keys) and
attribute each cache entry "made by" a machine to that ID,
this still does not count for actual moderation at scale.

It's not a good time, legally, and I think it would be *strongly*
advisable to make what I'm describing here work like Syncthing.
I think it would still be beneficial to have
local clusters scattered about the world,
owned by individuals and high-trust groups,
possibly only pushing content to untrusted devices, but not *pulling*.
I would like it if my computer collection
could act like a personal CDN, basically.

This does not get much better in the next section.

# An asynchronous Web (would be incredibly hard)

Consider that if that dream comes to fruition,
you'd have what amounts to FidoNet again,
but one that can only mirror existing Web content
addressed by IP address or DNS domain,
further narrowed down by what amounts to a filesystem tree.
You still need servers, *a computer*, to host Web sites,
and I think that's an annoying state of affairs?

It would be nice if you could just publish a
site to the cache directly!
Make it content-addressable and give every page a
bunch of tags and identity info so they can be searched
and collated into a discrete "site"
(I am very much thinking in terms of blogs)
with an index and so on and so forth...

...in a trusted environment.
In *this* world, however, hoo boy.
I'm no lawyer, but I'm not enough of a fool to not notice that
"network that automatically replicates data across all nodes"
is a *hellish* legal liability for all involved.

There are ways, I feel, to limit the problem domain.
You could centralize what the network considers "identity",
though you would still run into
the problem of existing nodes carrying
material from nodes that have been banned.
You would be a pretty stressed-out operator.
I could describe a fully-decentralized scheme,
but it suffers from *worse* problems,
such as "how do you know that a given set of pages belongs to some author?",
and *completely* uncontrolled "crap, we just mirrored illegal material".

In the interests of not giving people illegally bad ideas,
I'm not going to describe it.

. .. ...

.

. ..

..

...


<div class='altcontext' markdown='1'>
What the actual **fuck** did I just write?
</div>

# The Doohickey Corporation disclaims all responsibility for...

I did not expect this post to end this way.
I came into this bright-eyed and
hoping to write into existence a better Web of the future,
but while the technical foundations are at least within implementation range,
the social and especially *legal* implications
of the system I'm describing are intractable.

I do hope this was worth the read.
I hope you thought this was an interesting read,
even though I'm not sure how to recover from that 'splat!' of a sentence.
Thank you.

# Footnotes

[^cryptodunk]: While the green-plumbob[^naming] blockchain is,
    in a literal sense,
    a decentralized system in the sense that many nodes exist on it,
    I would consider it to have failed catastrophically in 4 ways,

    * it decentralizes data storage while centralizing authority,

    * its design cannot accomodate any participants but those with the money
      to operate tens of terabytes of disk space, and

    * you (via proof-of-stake verification)
      literally need money to make money.

    * it incentivizes people to act in wildly antisocial ways,
      arguably making it an antinetwork.

    It is a shameless mockery of the intent of the word "decentralized".

[^naming]: You can just refuse to call
    things by their intended names
    if you think the name conveys some intention
    you believe is not worthy of repetition.

[^onion-routing]: You can also layer the encryption ("onion routing")
    to enforce a specific path through the network,
    though I would say this is a little bit outdated
    if you can assume a TCP/IP network (IP provides the routing),
    as well as the presence of global peer discovery nodes
    to allow nodes to find each other on disparate networks.

    You do have stuff like Tor, though, where onion routing
    ensures that intermediate nodes and things sniffing them
    can't see your traffic, even if they tried.

[^bbs]: "BBS" is a difficult word to make sense of in the modern world.

    In its most minimal form, I *think* you could describe it as a
    communal shared mailbox (or a physical bulletin board)
    hosted on Some Guy's Computer
    that you could put your words into
    by connecting to it over dial-up; the phone lines.
    Then, someone else would dial into the BBS and put their words
    into the shared box. They might be aimed specifically at you, or
    it could just be for everyone to read.
    At some point, you would dial back in, and say "wowie, I have mail!".
    Rinse and repeat until the BBS dies.

    This was, and I'm presenting this in
    a way that would bore *me* to tears,
    a Whole Thing for a while.
    Please go read
    *The Modem World: A Prehistory of Social Media* by Kevin Driscoll.
    That book is a lot better at putting BBSes into the context they
    deserve to be memorialized in.

[^peer-discovery]: Peer discovery, and how much of it you want to do,
	is something I'm going to leave to implementation.

	My understanding is that Syncthing only has global peer discovery nodes.
	BitTorrent and IPFS make use of both global discovery and DHT;
	basically exchanging (partial?) node lists between individual nodes
	without needing nodes to announce to the entire world that they have a list.
