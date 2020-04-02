# Touhou 12.3 Protocol Documentation

This repository contains detailed documentation about the Touhou 12.3 network protocol.

## Terms

- Game client: an instance of Touhou 12.3 running the game
- Host: the single game client that initially hosts a game by using "Host" in the network menu
- Client: the single game client that initially connects to a game to play
- Parent of a game client: the game client that is connected by that other game client
- Child of a game client: the game client that connects to that game client
- Peer of a game client: any game client that is connected to another game client, either a parent or its child 
- Player: a game client that intends to play a game: either the host or the client
- Spectator: a game client that intends to spectate a game: any game client but the host or the client

## Network topology

The network topology is a tree of game clients: each node is a game client which has 0 or more child game clients. When a game client connects to a node, either that node accepts the connection, or it redirects the game client to one of its children nodes. What is a commonly called the *spectating chain* is actually a tree, not a chain.

The host can only have at most one child, which is the client.

The client can only have at most one child, which is the first spectator.

All spectators can have at most 4 children, which are all spectators.

When a redirects a game client to one of its children, it always redirects it to the one with the fewest nodes in it subtree, and it multiple nodes are identical it redirects it to the older one.

A typical game tree can look like this (host, client, spectator shortened to H, C, S):
```
               H
               +
               |
               v
               C   
               +
               |
               v
               S1    
               |
 +-------+-----+--+--------+       
 |       |        |        |       
 v       v        v        v      
 S2      S3       S4       S5       
 |       |        |        |
 |       |        |        |
 +---+   +---+    +---+    |
 |   |   |   |    |   |    |
 v   v   v   v    v   v    v
 S6  S10 S7  S11  S8  S12  S9
```

The tree is formed like this:
- H is the root of the tree
- C connects to H and H accepts the connection
- S1 connects to H but H already has one child, so H redirects to C which accepts the connection
- S2 connects to H, H redsirects to C, which also has one child and redirects to S1, which accepts the connection
- S3, S4, S5 connect to H, C, S1, and S1 accepts the connection because it can have at most 4 children
- S6 connects to H, C, S1, but S1 is already full so it redirects it to its child with the less nodes in its subtree, but S2, S3, S4, S5, all have only 1 spectator in their subtree, so S6 is redirected to S2, which is the oldest one
- S7 connects to H, C, S1; S2 has a subtree node count of 2 (both itself and S6), so S7 is redirected to S3
- S8 and S9 connect to S4 and S5 in the same manner
- S10 connects to H, C, S1; S2, S3, S4, S5 all have a subtree node count of 2, so S10 is redirected to S2
- S11 and S12 connect to S3 and S4 in the same manner

## Session example

*TODO: show an example of a session for both playing and spectating, explain match ids, ...*

## Packets

The game clients exchange data using UDP datagrams. Each game packet is a UDP datagram with a one-byte header identifying its type, followed by the packet data:
```
packet = <type> <data>
```

### Core game packets

#### Packet 0x01: HELLO

HELLO packets are sent in two different cases:
- when connecting to a game client
- when asking a game client to help establishing connectivity to another peer using hole-punching

```
hello = 0x01 <peer_address(sockaddr_in)> <target_address(sockaddr_in)> <stuff>
sockaddr_in = 0x02 0x00 <port> <addr> <padding>
port = { 2 bytes: unsigned little-endian short }
addr = { 4 bytes: IPv4 address bytes in network order }
padding = { 8 bytes: padding, ignored }
stuff = 0x00 0x00 0x00 0xBC // unknown
```

`peer_address` is the address of the game client the packet is sent to.

`target_address` is the address of the game client the game client whishes to connect to.

##### HELLO for connecting to a game client

When a game client connects to a game client, it will send 5 HELLO packets every frame, with `peer_address` and `target_address` set to the same address, that of the peer the game client is connecting to.

The peer will respond with either:
- an OLLEH packet, if it accepts this connection (typically when a player joins a host with no current player)
- a REDIRECT packet, if it instead redirects this game client to one of its children, if it already has too many children nodes

##### HELLO for establishing connectivity to a target

When a game client is redirected from a peer to another target game client (with a REDIRECT), it will explicitly ask the peer to help establish connectivity between that target and the game client. Connectivity will be established with UDP hole punching, by asking the target to send UDP packets to the game client; and the game client itself will simultenously send packets to the target.

When the receiver (of address `peer_address`) receives a HELLO packet, it will send a PUNCH packet to the target (of address `target_address`). That target game client will in turn send an OLLEH packet to the game client.

#### Packet 0x02: PUNCH

PUNCH packets are sent after receiving HELLO packets when those are used for establishing connectivity to a target. Instead of sending them back to the game client that sent the HELLO packet, they are sent to another, target game client, that was specified in the HELLO packet. That target will then send an OLLEH packet back to the peer that sent the HELLO packet.
```
punch = 0x02 <address(sockaddr_in)> <stuff>
sockaddr_in = 0x02 0x00 <port> <addr> <padding>
port = { 2 bytes: unsigned little-endian short }
addr = { 4 bytes: IPv4 address bytes in network order }
padding = { 8 bytes: padding, ignored }
stuff = 0x00 0x00 0x00 0x00 // unknown
```

`address` is the address of the game client that sent the HELLO packet to that peer.

#### Packet 0x03: OLLEH 

OLLEH packets are sent in two different cases:
- in response to a HELLO packet: they are sent to the peer that sent the HELLO packet
- in response to a PUNCH packet from another peer: they are sent to the peer specified in the `address` of the PUNCH packet

In both cases, an OLLLEH packet means that the connection is accepted from the sender.

```
olleh = 0x03
```

#### Packet 0x04: CHAIN

CHAIN packets are sent every 0.5s from every game client to all its children. They are used to notify the entire tree of game clients about the number of game clients in the subtree they belong to.

```
chain = 0x04 <spectator_count>
spectator_count = { 4 bytes: unsigned little-endian }
```

The `spectator_count` is equal to: 
```
1 + 3 * { count of game clients in the subtree, including self }
```

That count is obtained by summing all the `spectator_count` of all its children, and adding 1 if the peer is itself a spectator.

It seems that when a new spectator is added, the count of its parent can be undefined data until the spectator sends its count to the parent. In other words, the packet can sometimes be:
```
chain = 0x04 <stuff>
stuff = { 4 bytes: probably garbage }
```

#### Packet 0x05: INIT_REQUEST

INIT_REQUEST packets are sent from a game client to its parent after a connection is established (after it has received an ELLOH from its peer) to initialize a proper game session. It is used to compare the game versions used, in order to prevent any incompatible game version from joining, and to declare its profile name (when playing).

This packet is used in two different cases:
- when initializing a game session for playing
- when initializing a game session for spectating

After receiving an INIT_REQUEST, the peer will either:
- not respond, if it doesn't accept the game id (for example using Sokuroll on a host without Sokuroll)
- respond with an INIT_SUCCESS if it accepts the session
- respond with an INIT_ERROR if it doesn't accept the session

```
init_request = 0x05 <game_id> <stuff> <play_request | spec_request> <padding>
game_id = { 16 bytes: identifier for the game }
stuff = 0x3B 0xAA 0x01 0x6E 0x28 0x00 0xFC 0x30 // unknown
padding = 0x??* // pad until init_request is 65 bytes long
```

The game id depends on both whether Sokuroll is used, whether Touhou 10.5 (SWR) is successfully connected to the game, and on the characters unlocked:

- For Hisoutensoku 1.10ac, with Sokuroll 1.3, with SWR linked, all characters unlocked:
```
game_id = 0x64 0x73 0x65 0xD9 0xFF 0xC4 0x6E 0x48 0x8D 0x7C 0xA1 0x92 0x31 0x34 0x72 0x95
```
- For Hisoutensoku 1.10ac, without Sokuroll, with SWR linked, all characters unlocked:
```
game_id = 0x6E 0x73 0x65 0xD9 0xFF 0xC4 0x6E 0x48 0x8D 0x7C 0xA1 0x92 0x31 0x34 0x72 0x95
```
- For Hisoutensoku 1.10ac, with or without Sokuroll, without SWR linked, all characters unlocked:
```
game_id = 0x46 0xC9 0x67 0xC8 0xAC 0xF2 0x44 0x4D 0xB8 0xB1 0xEC 0xEE 0xD4 0xD5 0x40 0x4A
```

*The game ids when not using SWR for both with and without Sokuroll are the same, and indeed a player without SWR with Sokuroll can connect to a peer without SWR without Sokuroll, and the game clients will freeze because of an incompatibility.*

##### INIT_REQUEST when starting a game session for playing

```
play_request = 0x01 <profile_name_length> <profile_name> 0x00
profile_name_length = { 1 byte: size of nick in bytes }
profile_name = { <profile_name_length> bytes }
```

`profile_name` is the profile name of the game client that is sending the INIT_REQUEST (eg `profile1p`). This string does have both a length identifier and a terminating NULL byte.

##### INIT_REQUEST when starting a game session for spectating

```
spectate_request = 0x00
```

#### Packet 0x06: INIT_SUCCESS

INIT_SUCCESS is sent in response to an INIT_REQUEST packet when the parent accepts the game session. The first time the INIT_SUCCESS packet is sent, it contains additional information about both players profile names. In subsequent INIT_SUCCESS packets, this information is dropped.

*This does mean that if the initial packet is dropped due to packet loss, this information will never be received. And indeed, with packet loss, the official game client may never receive the profile names and show an empty profile name instead.*

```
init_success = 0x06 <stuff> <data_first|data_additional>
stuff = <stuff_1> <data_size> <stuff_2>
stuff_1 = 0x00 0x00 0x00 0x00 0x10 0x00 0x00 0x00
data_size = { 1 byte }
stuff_2 = 0x00 0x00
```

`data_size` is the size in bytes of either `data_accept` or `data_additional`.

##### INIT_SUCCESS when a peer first accepts a game session

```
data_first = <host_profile(profile_name_padded)> <client_profile(profile_name_padded)> <swr_disabled>
profile_name_padded = <profile_name> 0x00 <profile_name_padding>
profile_name = { <profile_name_length> bytes }
profile_name_padding = 0x??* // pad until profile_name_padded is 32 bytes long
swr_disabled = { 4 bytes: unsigned little-endian }
```

`host_profile` and `client_profile` are the profile names of the host and the client. `profile_name_length` is the respective length in bytes of their profile names.

`swr_disabled` is 0 if SWR was successfully linked to both players and is used for the game, 1 otherwise.

##### INIT_SUCCESS in subsequent packets

```
data_additional = // these additional packets have no additional data
```

#### Packet 0x07: INIT_ERROR

INIT_ERROR packets are sent in response to INIT_REQUEST packets when the parent does not accept the game session. This can happen for several reasons:
- if a game session for spectating was requested but the game does not allow spectators *(reason **0**)*
- if a game session for spectating was requested but the game has not yet started (no player has joined the host) *(reason **1**)*
- if a game session for playing was requested but the game has already started (a player has already joined) *(reason **1**)*

```
init_error = 0x07 <reason>
reason = { 4 bytes: unsigned little-endian }
```

`reason` depends on the reason why the game session was refused; see above.


#### Packet 0x08: REDIRECT

REDIRECT packets are sent in response to HELLO packets when the receiving peer wishes to redirect the game client that sent the HELLO packet to one of its children. The game client will then try to connect to that child instead. This is used when the receiving peer already has too many children.

```
redirect = 0x08 <child_id> <target_address(sockaddr_in)> <stuff>
child_id = { 4 bytes: unsigned little-endian }
sockaddr_in = 0x02 0x00 <port> <addr> <padding>
port = { 2 bytes: unsigned little-endian short }
addr = { 4 bytes: IPv4 address bytes in network order }
padding = { 8 bytes: padding, ignored }
stuff = { 48 bytes } // unknown
```

`child_id` is an index for the child target game client the peer redirects to; if a game client has 4 children and redirects to the 2nd newer one, `child_id` will be `2`. In most cases (when there are less than 6 spectators) a peer only has 1 child and `child_id` will be `1`.

`target_address` is the address of the target game client that the game client that sent the HELLO packet should connect to.

#### Packet 0x0B: QUIT

QUIT packets are sent by a peer when it wishes to close the current game session. No further messages will be sent to that peer from its connected peer, in particular no QUIT packet is sent back.

```
quit = 0x0B
```

#### Packet 0x0D, 0x0E: HOST_GAME, CLIENT_GAME

HOST_GAME and CLIENT_GAME packets are packets containing various information about a game session. HOST_GAME is used when sending data from a game client to one of its children, and CLIENT_GAME is used when sending data from a game client to its parent.

There are several game packet sub-types, which carry various types of information. The general game packet format is as follows:

```
game = <0x0D|0x0E> <game_type> <game_data>
```

##### Game packet sub-type 0x01: GAME_LOADED

GAME_LOADED is sent every frame by a player who has finished loading a new game scene (corresponds to a screen, such as character select screen or battle screen) and is waiting for its player peer to load it as well, until the player peer replies with a GAME_LOADED_ACK packet. This can happen in several cases:
- when a player has finished loading a stage for a match (after a stage and music is selected)
- when a player has completed the post-match character dialog and replay save dialog after a match.

A player receiving a GAME_LOADED packet will either:
- not respond, if it hasn't finished loading that scene
- respond with a GAME_LOADED_ACK if it's done loading that scene

```
game_type = 0x01
game_data = <scene_id>
```

`scene_id` is the id of the scene the player has loaded and is waiting for its player peer to load.

Scenes:
| Scene | Id |
| -- | -- |
| Character select | 0x03 |
| Battle | 0x05 |

##### Game packet sub-type 0x02: GAME_LOADED_ACK

GAME_LOADED_ACK is sent in response to a GAME_LOADED game packet when the requested scene has been loaded.

```
game_type = 0x02
game_data = <scene_id>
```

`scene_id` is the id of the scene that has been loaded, and is equal to the `scene_id` of the corresponding GAME_LOADED packet.

##### Game packet sub-type 0x03: GAME_INPUT

GAME_INPUT is sent every frame by players to their player peer when a scene is loaded (typically during character select and during match gameplay (battle)). It contains the inputs pressed by that player during the last few frames (the count of frames varies). 

```
game_type = 0x03
game_data = <frame_id> <scene_id> <inputs_count> <sokuroll_inputs> <game_inputs>
inputs_count = { 1 byte }
game_inputs = { <game_inputs_count> <game_input> }
game_input = { 2 bytes: bitflag }
```

`frame_id` is the current frame id, starting at 1 and increasing by 1 on every frame.

`sokuroll_inputs` is additional data added by Sokuroll, only when sent from a parent (HOST_GAME), when using Sokuroll, during battle. Otherwise, it is empty, and the only inputs are regular game inputs.

`inputs_count` is `sokuroll_inputs_count` + `game_inputs_count`.

`game_inputs` is a list of the last `game_inputs_count` inputs of the game client that sends the packet, stored in descending frame order. The first `game_input` is the input at frame `frame_id`, the next one is the input at frame `frame_id - 1`, and so on.

`game_inputs_count` depends on whether Sokuroll is used, and on the sending side:
- `game_inputs_count` is always 1 when sent from the client (GAME_CLIENT) during character select
- `game_inputs_count` is always 1 when sent from the client (GAME_CLIENT) without Sokuroll
- `game_inputs_count` is otherwise `maximum rollback + input delay` when using Sokuroll *(beware that those are the actual maximum rollback and input delay in frames, so they must be multiplied by 2 from the Sokuroll settings)*
- `game_inputs_count` otherwise depends on the estimated connection latency, typically 2 to 30.

`scene_id` and the meaning of the bitflag in `input` depend on the current scene, either character select, or match gameplay (battle).

###### GAME_INPUT during character select

```
scene_id = 0x03
sokuroll_inputs = // there are no Sokuroll inputs during character select
sokuroll_inputs_count = 0
```

Charater select input bit flags:
| Button | Value (binary) | Value (Hex) |
| -- | -- | -- |
| Dash | 0b00000000 0b00000001 | 0x00 0x01 |
| A | 0b00000000 0b00000010 | 0x00 0x02 |
| Up | 0b00000001 0b00000000 | 0x01 0x00 |
| Down | 0b00000010 0b00000000 | 0x02 0x00 |
| Left | 0b00000100 0b00000000 | 0x04 0x00 |
| Right | 0b00001000 0b00000000 | 0x08 0x00 |
| Z / Pad 0 | 0b00010000 0b00000000 | 0x10 0x00 |
| X / Pad 1 | 0b00100000 0b00000000 | 0x20 0x00 |
| C / Pad 2 | 0b01000000 0b00000000 | 0x40 0x00 |
| Q / Pad 3 | 0b10000000 0b00000000 | 0x80 0x00 |

###### GAME_INPUT during battle

```
scene_id = 0x05
```

Battle input bit flags:
| Button | Value (binary) | Value (Hex) |
| -- | -- | -- |
| A+B | 0b00000000 0b00000001 | 0x00 0x01 |
| B+C | 0b00000000 0b00000010 | 0x00 0x02 |
| Up | 0b00000001 0b00000000 | 0x01 0x00 |
| Down | 0b00000010 0b00000000 | 0x02 0x00 |
| Left | 0b00000100 0b00000000 | 0x04 0x00 |
| Right | 0b00001000 0b00000000 | 0x08 0x00 |
| A | 0b00010000 0b00000000 | 0x10 0x00 |
| B | 0b00100000 0b00000000 | 0x20 0x00 |
| C | 0b01000000 0b00000000 | 0x40 0x00 |
| Dash | 0b10000000 0b00000000 | 0x80 0x00 |

###### GAME_INPUT during battle from a server with Sokuroll

```
sokuroll_inputs = <input_latency> <input_latency>
sokuroll_inputs_count = 2
input_latency = { 2 bytes: unsigned little-endian short }
```

For GAME_INPUT game packets sent in a HOST_GAME packet during a battle scene, Sokuroll sends the `input_latency` Sokuroll parameter twice. This parameter corresponds to the current `input_latency` Sokuroll parameter, which is half the actual input latency in frames. This parameter is written twice.

###### GAME_INPUT during battle in other cases

```
sokuroll_inputs = // there are no Sokuroll inputs in other cases
sokuroll_inputs_count = 0
``` 

##### Game packet sub-type 0x04: GAME_MATCH

GAME_MATCH contains information about the match stage, music, players deck data, ... The packet differs depending on whether it is sent to a player or a spectator. It is sent in two cases:
- when a match is starting, to its player peer
- in response to a GAME_REPLAY_REQ packet, to its spectator child

When sending GAME_MATCH to a spectator, the packet includes information about both players. When GAME_MATCH packets are exchanged between players, they contain less information, in particular they don't contain information about the other player deck, since that player will send this information in its packet.

The general format for that packet is as follows. Note that when sent to a player, this packet is slightly different.
```
game_type = 0x04
game_data = <host_data(player_match_data)> <client_data(player_match_data)> <stage_id> <music_id> <random_seed> <match_id>
player_match_data = <character_id> <skin_id> <deck_id> <deck_size> <deck> <disabled_simultaneous_buttons>
character_id = { 1 byte }
skin_id = { 1 byte }
deck_id = { 1 byte }
deck_size = { 1 byte }
deck = { <deck_size> <card_id> }
card_id = { 2 bytes: unsigned little-endian short }
disabled_simultaneous_buttons = { 1 byte }
stage_id = { 1 byte }
music_id = { 1 byte }
random_seed = { 4 bytes }
match_id = { 1 byte }
```

`character_id` is the character used by that player, according to the character table below. If it was chosen to be random, it is determined before sending the packet. `skin_id` is the palette used by that player.

`deck_id` is merely used to send the color of the deck used, but **the player send the actual cards of the deck each time at the beginning of each match, they do no store the list of decks for each character in advance**; this value is therefore probably unused.

`deck` is the list of cards in a player deck, its size `deck_size` is always 20 in vanilla game clients but could possibly be less according to the protocol. Each `card_id` is a card in that deck.

`disabled_simultaneous_buttons` is 1 if the profile of that player has disabled simultaneous buttons (in other words, if pressing A and B simultaneously won't trigger a spellcard).

`stage_id` and `music_id` are identifiers for the stage and music. If those were chosen to be random, there are determined before sending that packet, in other words there are no special values for random stages and music.

`random_seed` is the seed of the random number generator for that match.

`match_id` is an id for the current match, starting at 0 and increasing by 1 on every match, only used when sending to a spectator. 

Character table:
| Id | Name |
| -- | -- |
| Reimu | 0x00 |
| Marisa | 0x01 |
| Sakuya | 0x02 |
| Alice | 0x03 |
| Patchouli | 0x04 |
| Youmu | 0x05 |
| Remilia | 0x06 |
| Yuyuko | 0x07 |
| Yukari | 0x08 |
| Suika | 0x09 |
| Reisen | 0x0A |
| Aya | 0x0B |
| Komachi | 0x0C |
| Iku | 0x0D |
| Tenshi | 0x0E |
| Sanae | 0x0F |
| Cirno | 0x10 |
| Meiling | 0x11 |
| Utsuho | 0x12 |
| Suwako | 0x13 |

###### GAME_MATCH for a player

When sent to a player, some information about that player is dropped, since it is supposed to be provided by that player to the other player. The differences are as follows.

```
client_data = <character_id> <skin_id> <deck_id> <deck_skipped_size> <deck_skipped> <disabled_simultaneous_buttons_skipped>
deck_skipped_size = 0x00
deck_skipped = // the client deck is not sent to the client
disabled_simultaneous_buttons_skipped = 0x00 // padding instead of disabled_simultaneous_buttons
match_id = { 1 byte } // padding instead of match_id
```

In other words, the deck is sent as if it was of size 0 (only its length, 0, is written), and the two fields `disabled_simultaneous_buttons` and `match_id`, which are not relevant, are ignored padding (random garbage data).

##### Game packet sub-type 0x05: GAME_MATCH_ACK

GAME_MATCH_ACK is sent in response to a GAME_MATCH_REQUEST game packet.

```
game_type = <0x05>
game_data = // there is no additional data for this game type
```

##### Game packet sub-types 0x08: GAME_MATCH_REQUEST

GAME_MATCH_REQUEST is sent every frame by a player who is waiting for its player peer to send its match data with a GAME_MATCH game packet. This happens after a stage and music is selected, before the match starts. The player receiving the packet will reply with a GAME_MATCH_ACK.

```
game_type = 0x08
game_data = // there is no additional data for this game type
```

##### Game packet sub-type 0x09: GAME_REPLAY

GAME_REPLAY game packets are sent in resposne to GAME_REPLAY_REQUEST game packets from children spectators. They contain zlib-compressed inputs of both players of a specific match, starting at a specific frame. GAME_REPLAY game packets are always sent in HOST_GAME packets.

```
game_type = 0x09
game_data = <compressed_size> <compressed_data>
compressed_size = { 1 byte }
compressed_data = { zlib of replay_data }
replay_data = <frame_id> <end_frame_id> <match_id> <game_inputs_count> <replay_inputs>
frame_id = { 4 bytes: unsigned little-endian }
end_frame_id = { 4 bytes: unsigned little-endian }
match_id = { 1 byte }
game_inputs_count = { 1 byte }
replay_inputs_count = { 1 byte }
replay_inputs = { <replay_inputs_count> <replay_input> }
replay_input = <client_input(game_input)> <host_input(game_input)>
game_input = { 2 bytes: bitflag }
```

`frame_id` is the frame of the most recent input sent in this packet, and is usually different from the `frame_id` set in the corresponding GAME_REPLAY packet.

`end_frame_id` stores information about when a match ends: if the match has not yet ended, it is set to 0, otherwise it is set to the frame on which the match ends (which might be well after the highest frame of the inputs sent in that packet).

`match_id` is an id for the current match, starting at 0 and increasing by 1 on every match, and usually corresponds to the `match_id` set in the corresponding GAME_REPLAY packet.

`replay_inputs` is a list of the last `replay_inputs_count` inputs of both players, stored in descending frame order. The first `replay_input` is the input at frame `frame_id`, the next one is the input at frame `frame_id - 1`, and so on.

`replay_input` is a pair of a `client_input` and a `host_input`, always sent in pairs and in that order. There are `game_inputs_count` `game_input`, which means that there are actually `replay_inputs_count = game_inputs_count / 2` pairs of host and client inputs.

##### Game packet sub-type 0x0B: GAME_REPLAY_REQUEST

GAME_REPLAY_REQUEST is sent every 3 frames from spectators to their parent to request replay data, that is, inputs of both players from a specific frame offset in a specific match. GAME_REPLAY_REQUEST game packets are always sent in CLIENT_GAME packets.

The parent will respond:
- with a GAME_MATCH game packet if a newer match has started
- otherwise, with a GAME_REPLAY game packet containing some input data starting at that offset. 

```
game_type = 0x0B
game_data = <frame_id> <match_id>
frame_id = { 4 bytes: unsigned little-endian }
match_id = { 1 byte }
```

`frame_id` is the current frame of the spectator, which is the lowest frame for which the spectator has not received inputs yet.

`match_id` is an id for the current match, starting at 0 and increasing by 1 on every match. 

### Sokuroll packets

Sokuroll packets are only exchanged when the game clients use Sokuroll, and are only exchanged between player peers. Spectators do not send or receive any of these packets.

#### Packet 0x10: SOKUROLL_TIME

SOKUROLL_TIME packets are sent every 0.1s to estimate the connection latency between the two players. The general idea is that a peer sends a SOKUROLL_TIME packet, gets a corresponding SOKUROLL_TIME_ACK packet from its peer, then computes the RTT latency by subtracting the SOKUROLL_TIME_ACK receive time with the SOKUROLL_TIME send time. This time estimation is probably unused.

```
sokuroll_time = 0x10 <sequence_id> <time_stamp>
sequence_id = { 4 bytes: unsigned little-endian }
time_stamp = { 4 bytes: unsigned little-endian }
```

`sequence_id` is a sequence id for SOKUROLL_TIME packets, starting at 1 and increasing by 1 every time a SOKUROLL_TIME packet is sent from that peer.

`time_stamp` is a number of elapsed miliseconds since an arbitrary origin.

#### Packet 0x11: SOKUROLL_TIME_ACK

SOKUROLL_TIME_ACK packets are sent in response to SOKUROLL_TIME packets to help the peer that sent the SOKUROLL_TIME packets estimate the connection latency.

```
sokuroll_time_ack = 0x11 <sequence_id> <time_stamp>
sequence_id = { 4 bytes: unsigned little-endian }
time_stamp = { 4 bytes: unsigned little-endian }
```

`sequence_id` and `time_stamp` have the same value as those used in the SOKUROLL_TIME packet this packet is in response to.

#### Packet 0x12: SOKUROLL_STATE

SOKUROLL_STATE packets are sent each frame from both player peers, only when Sokuroll debug is enabled in its config file, in order to check for desyncs.

```
sokuroll_state = 0x12 <frame_id> <host_x> <host_y> <client_x> <client_y> <stuff>
frame_id = { 4 bytes: signed little-endian }
host_x = { 4 bytes: signed little-endian }
host_y = { 4 bytes: signed little-endian }
client_x = { 4 bytes: signed little-endian }
client_y = { 4 bytes: signed little-endian }
stuff = { 4 bytes } // unknown, probably a value from the game state
```

`frame_id` is the current frame id, starting at 1 and increasing by 1 on every frame.

`host_x` and `client_x` are the X positions of the host and client characters. `host_y` and `client_y` are the Y positions of the host and client characters, *or 0 if the character is touching the ground*.

#### Packet 0x13: SOKUROLL_SETTINGS

SOKUROLL_SETTINGS packets are used to send the current input delay and maximum rollback Sokuroll settings. They are sent by the host every 0.1s after game session initialization or when changing the input delay in-game, until a SOKUROLL_SETTINGS_ACK packet is received.

```
sokuroll_settings = 0x13 <maximum_rollback> <input_delay>
maximum_rollback = { 1 byte }
input_delay = { 1 byte }
```

`maximum_rollback` corresponds to `MaximumRollback` in the settings, and is **half** the maximum number of frames that the game can roll back on each pass. For example, a `maximum_rollback` of 2 means that the game can rollback 4 frames.

`input_delay` initially corresponds to `InitialDelay` in the settings and can vary during the game, and is **half** the current connection input delay in frames. For example, an `input_delay` of 2 means that the game actually has 4 frames of input delay.

#### Packet 0x14: SOKUROLL_SETTINGS_ACK

SOKUROLL_SETTINGS_ACK packets are sent in response to SOKUROLL_SETTINGS packets.

```
sokuroll_settings_ack = 0x14
```
