Background
----------
Since the completion of [BitTube](https://github.com/yicui/BitTube) project, I haven't given up looking for ways to
further minimize intrusion of a P2P program. It came to my mind that the best way to hide your P2P functionality is
the Flash player itself. After all, why do you need two pieces of program to coordinate on the same job? Also since
Flash had (and still has) such a high penetration rate, if I could find a way to sneak in, then bingo, instant
world domination :) I came as close to Flash's [Socket API](http://help.adobe.com/en_US/FlashPlatform/reference/actionscript/3/flash/net/Socket.html) 
but couldn't get further. All I need is my socket able to listen on a port, but understandably this shouldn't be allowed.

Then in the early summer of 2009, my dream came true: Adobe rolled out their [RTMFP protocol](http://en.wikipedia.org/wiki/Real_Time_Media_Flow_Protocol)!
I jumped in immediately only to find out it was quite primitive: only point-to-point connection is allowed among peers.
Also instead of integrating the tracker functionality into one of their products like Media Server, Adobe kept it on one
of its own servers and you must apply for an account to use it. So everything was behind a black box and they could cut
me off any time. But this is good enough for me and I can build whatever lacking by my own code. And best of all, this
all happens inside the Flash player, exactly what I'm looking for.

About six months into my development, a new version of RTMFP was introduced, this time with the powerful 
[NetGroup](http://help.adobe.com/en_US/FlashPlatform/reference/actionscript/3/flash/net/NetGroup.html) that can do
everything: object replication in a BitTorrent-fashioned swarm, application-layer multicasting either pull- or push-based,
message broadcasting. This rendered my own development largely useless, but that's okay, we can now focus on the
application.
Rationale
---------
This time I decided to focus on live streaming: lectures, broadcasting from a studio or TV channel. The reasons? First,
hiding everything in the Flash player has a price to pay: users can shut it down anytime just by closing the webpage.
Second, the Flash sandboxed model requires the player to only access network resources, not local ones. So you can't
access data on hard drive like BitTorrent does. In lieu of the longevitity & rich accessiblity a BitTorrent client can
enjoy, I need to find something where all users are online at the same time, even for just 10 minutes. Better yet,
the more peers there are, the more bandwidth they can share, which paints a (ideally) scall-free picture for those 
who need to sleep on server capacity planning.

For those who has played with ActionScript, live streaming may sound rather trivial with the help of
[NetStream](http://help.adobe.com/en_US/FlashPlatform/reference/actionscript/3/flash/net/NetStream.html) class. All you
have to do is hook the broadcasting node with a camera, then call *publish()*, and let subscribers call *play()*, done.
While this is all true, I want my application to help in more scenarios. First, many event organizers want to save 
their broadcasts and replay later, which is not supported by the above simple operation. Second, lots of live feeds do
not come from a camera, but from some TV capturing devices or already wrapped in HTTP or RTMP stream emitted from an
FMS/Wowza/who-knows-what server. What's my chance if I ask these **functioning** legacy infrastructure to be restructured 
for some bandwidth-saving pitch? Well, I learned this the hard way: operators hate intrusion as much as users, 
if not more.
Design
------
So I decided to ignore the source of the feeds, but deal with the stream itself. Most open media containers follow
a remarkly similar structure: a file header followed by an array of audio/video/metadata blocks, or tags in 
[FLV jargons](http://download.macromedia.com/f4v/video_file_format_spec_v10_1.pdf), which are sequenced by their
timestamps. As long as I can make a server (or proxy if you prefer) to break this stream into packets (each containing
one or more tags) and throw them into the NetGroup swarm, the rest will be straightforward: peers can retrieve these 
pieces and reassemble them based on the timestamps therein, then feed to the player. As of now we support live streaming
in the following scenarios:
#### HTTP Stream
In order to parse and carve out tags from the FLV file beging streamed, we use the raw
[Socket](http://help.adobe.com/en_US/FlashPlatform/reference/actionscript/3/flash/net/Socket.html) class and implement
a mini-HTTP parser. We use the *send()* method of NetStream quite extensively to push these tags into the NetGroup swarm.
#### Local File
In order to parse and carve out tags from the FLV file beging streamed, we use the raw
