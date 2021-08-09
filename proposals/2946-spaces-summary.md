# MSC2946: Spaces Summary

This MSC depends on [MSC1772](https://github.com/matrix-org/matrix-doc/pull/1772), which
describes why a Space is useful:

> Collecting rooms together into groups is useful for a number of purposes. Examples include:
>
> * Allowing users to discover different rooms related to a particular topic: for example "official matrix.org rooms".
> * Allowing administrators to manage permissions across a number of rooms: for example "a new employee has joined my company and needs access to all of our rooms".
> * Letting users classify their rooms: for example, separating "work" from "personal" rooms.
>
> We refer to such collections of rooms as "spaces".

This MSC attempts to solve how a member of a space discovers rooms in that space. This
is useful for quickly exposing a user to many aspects of an entire community, using the
examples above, joining the "official matrix.org rooms" space might suggest joining a few
rooms:

* A room to discuss development of the Matrix Spec.
* An announcements room for news related to matrix.org.
* An off-topic room for members of the space.

## Proposal

A new client-server API (and corresponding server-server API) is added which allows
for querying for the rooms and spaces contained within a space. This allows a client
to efficiently display a hierarchy of rooms to a user (i.e. without having
to walk the full state of each room).

### Client-server API

An endpoint is provided to walk the space tree, starting at the provided room ID
("the root room"), and visiting other rooms/spaces found via `m.space.child`
events. It recurses into the children and into their children, etc.

Any child room that the user is joined or is potentially joinable (per
[MSC3173](https://github.com/matrix-org/matrix-doc/pull/3173)) is included in
the response. When a room with a `type` of `m.space` is found, it is searched
for valid `m.space.child` events to recurse into.

In order to provide a consistent experience, the space tree should be walked in
a depth-first manner, e.g. whenever a space is found it should be recursed into
by sorting the children rooms and iterating through them.

There could be loops in the returned child events; clients and servers should
handle this gracefully. Similarly, note that a child room might appear multiple
times (e.g. also be a grandchild). Clients and servers should handle this
appropriately.

This endpoint requires authentication and is not subject to rate-limiting.

#### Request format

```text
GET /_matrix/client/r0/rooms/{roomID}/hierarchy
```

Query Parameters:

* **`suggested_only`**: Optional. If `true`, return only child events and rooms
  where the `m.space.child` event has `suggested: true`.  Must be a  boolean,
  defaults to `false`.

  This applies transitively, i.e. if a `suggested_only` is `true` and a space is
  not suggested then it should not be searched for children. The inverse is also
  true, if a space is suggested, but a child of that space is not then the child
  should not be included.
* **`limit`**: Optional: a client-defined limit to the maximum
  number of rooms to return per page. Must be a non-negative integer.

  Server implementations should impose a maximum value to avoid resource
  exhaustion.
* **`max_depth`**: Optional: The maximum depth in the tree (from the root room)
  to return. The deepest depth returned will not include children events. Defaults
  to no-limit. Must be a non-negative integer.

  Server implementations may wish to impose a maximum value to avoid resource
  exhaustion.
* **`from`**: Optional. Pagination token given to retrieve the next set of rooms.

#### Response Format

* **`rooms`**: `[object]` For each room/space, starting with the root room, a
  summary of that room. The fields are the same as those returned by
  `/publicRooms` (see
  [spec](https://matrix.org/docs/spec/client_server/r0.6.0#post-matrix-client-r0-publicrooms)),
  with the addition of:
  * **`room_type`**: the value of the `m.type` field from the room's
    `m.room.create` event, if any.
  * **`children_state`**: The `m.space.child` events of the room. For each event,
    only the following fields are included<sup id="a1">[1](#f1)</sup>:
    `type`, `state_key`, `content`, `room_id`, `sender`,  with the addition of:
    * **`origin_server_ts`**: This is required for sorting of rooms as specified
      below.
* **`next_token`**: Optional `string`. The token to supply in the `from` param
  of the next `/spaces` request in order to request more rooms. If this is absent,
  there are no more results.

#### Example request:

```text
GET /_matrix/client/r0/rooms/{roomID}/hierarchy?
    limit=30&
    suggested_only=true&
    max_depth=4
```

#### Example response:

```jsonc
{
    "rooms": [
        {
            "room_id": "!ol19s:bleecker.street",
            "avatar_url": "mxc://bleecker.street/CHEDDARandBRIE",
            "guest_can_join": false,
            "name": "CHEESE",
            "num_joined_members": 37,
            "topic": "Tasty tasty cheese",
            "world_readable": true,
            "join_rules": "public",
            "room_type": "m.space",
            "children_state": [
                {
                    "type": "m.space.child",
                    "state_key": "!efgh:example.com",
                    "content": {
                        "via": ["example.com"],
                        "suggested": true
                    },
                    "room_id": "!ol19s:bleecker.street",
                    "sender": "@alice:bleecker.street",
                    "creation_ts": 1432735824653
                },
                { ... }
            ]
        },
        { ... }
    ],
    "next_token": "abcdef"
}
```

#### Errors:

An HTTP response with a status code of 403 and an error code of `M_FORBIDDEN`
should be returned if the user doesn't have permission to view/peek the root room.
This should also be returned if that room does not exist, which matches the
behavior of other room endpoints (e.g.
[`/_matrix/client/r0/rooms/{roomID}/aliases`](https://matrix.org/docs/spec/client_server/latest#get-matrix-client-r0-rooms-roomid-aliases))
to not divulge that a room exists which the user doesn't have permission to view.

An HTTP response with a status code of 400 and an error code of `M_INVALID_PARAM`
should be returned if the `from` token provided is unknown to the server.

#### Client behaviour

TODO

#### Server behaviour

The server should generate the response as discussed above, by doing a depth-first
search (starting at the "root" room) for any `m.space.child` events. Any
`m.space.child` with an invalid `via` are discarded (invalid is defined as in
[MSC1772](https://github.com/matrix-org/matrix-doc/pull/1772): missing, not an
array or an empty array).

In the case of the homeserver not having access to the state of a room, the
server-server API (see below) can be used to query for this information over
federation from one of the servers provided in the `via` key of the
`m.space.child` event. It is recommended to cache the federation response for a
period of time and to prefer local data over data returned over federation.

When the current response page is full, the current state should be persisted
and a pagination token should be generated (if there is more data to return).
If the client does not request the next page after a short period of time the
persisted data may be discarded to limit resource usage. It maybe possible to
generate reusable pagination tokens (i.e. sharable across users), but this is
left as an implementation specific detail.

The persisted state will generally include:

* Any processed rooms (and whether the requesting user is able to join them).
* A queue of rooms to process (in depth-first order with rooms at the same level
  ordered according to below).
* Pending information from the latest federation response.

### Server-server API

The Server-Server API has a similar interface to the Client-Server API. It is
used when a homeserver does not have the state of a room to include in the summary.

The main difference is that it does *not* recurse into spaces and does not support
pagination. This is somewhat equivalent to a Client-Server request with a `max_depth=1`.

If the requesting server wishes to explore a sub-space an additional federation
request can be made for any returned spaces. This should allow for trivially caching
responses.

Since the server-server API does not know the requesting user, the response should
divulge the information if any member of the requesting server could join the room.
The requesting server is trusted to properly filter this information.

If the target server is not a member of some children rooms (so would have to send
another request over federation to inspect them), no attempt is made to recurse
into them. They are simply omitted from the `rooms` key of the response.
(Although they will still appear in the `children_state`key of another room).

Similarly, if a server set limit on the size of the response is reached, additional
rooms are not added to the response and can be queried individually.

#### Request format

```text
GET /_matrix/federation/v1/hierarchy/{roomID}
```

Query Parameters:

* **`suggested_only`**: The same as the Client-Server API.

#### Response format

The response format is similar to the Client-Server API:

* **`room`**: `[obejct]` The summary of the requested room.
* **`children`**: `[object]` For each room/space, a summary of that room. The fields
  are the same as those returned by `/publicRooms` (see
  [spec](https://matrix.org/docs/spec/client_server/r0.6.0#post-matrix-client-r0-publicrooms)),
  with the addition of:
  * **`room_type`**: the value of the `m.type` field from the room's
    `m.room.create` event, if any.
  * **`allowed_room_ids`**: A list of room IDs which give access to this room per
    [MSC3083](https://github.com/matrix-org/matrix-doc/pull/3083).<sup id="a2">[2](#f2)</sup>* **`next_token`**: Optional `string`. The token to supply in the `from` param
  of the next `/spaces` request in order to request more rooms. If this is absent,
  there are no more results.
* **`inaccessible_children`**: Optional `[string]`. A list of room IDs which are
  children of the requested room, but are inaccessible to the requesting server.
  The requesting server should not attempt to request information about them
  from other servers.
  
  This is used to differentiate between rooms which the requesting server does
  not have access to from those that the target server cannot include in the
  response (which will simply be missing in the response).

#### Example request:

```jsonc
GET /_matrix/federation/v1/hierarchy/{roomID}?
    suggested_only=true
```

#### Errors:

An HTTP response with a status code of 404 is returned if the target server is
not a member of the requested room or the requesting server is not allowed to
access the room.

### MSC1772 Ordering

[MSC1772](https://github.com/matrix-org/matrix-doc/pull/1772) defines the ordering
of "default ordering of siblings in the room list" using the `order` key:

> Rooms are sorted based on a lexicographic ordering of the Unicode codepoints
> of the characters in `order` values. Rooms with no `order` come last, in
> ascending numeric order of the `origin_server_ts` of their `m.room.create`
> events, or ascending lexicographic order of their `room_id`s in case of equal
> `origin_server_ts`. `order`s which are not strings, or do not consist solely
> of ascii characters in the range `\x20` (space) to `\x7F` (~), or consist of
> more than 50 characters, are forbidden and the field should be ignored if
> received.

Unfortunately there are situations when a homeserver comes across a reference to
a child room that is unknown to it and must decide the ordering. Without being
able to see the `m.room.create` event (which it might not have permission to see)
no proper ordering can be given.

Consider the following case of a space with 3 child rooms:

```
         Space A
           |
  +--------+--------+
  |        |        |
Room B   Room C   Room D
```

Space A, Room B, and Room C are on HS1, while Room D is on HS2. HS1 has no users
in Room D (and thus has no state from it). Room B, C, and D do not have an
`order` field set (and default to using the ordering rules above).

When a user asks HS1 for the space summary with a `max_rooms_per_space` equal to
`2` it cannot fulfill this request since it is unsure how to order Room B, Room
C, and Room D, but it can only return 2 of them. It *can* reach out over
federation to HS2 and request a space summary for Room D, but this is undesirable:

* HS1 might not have the permissions to know any  of the state of Room D, so might
  receive a 403 error.
* If we expand the example above to many rooms than this becomes expensive to
  query a remote server simply for ordering.

This proposes changing the ordering rules from MSC1772 to the following:

> Rooms are sorted based on a lexicographic ordering of the Unicode codepoints
> of the characters in `order` values. Rooms with no `order` come last, in
> ascending numeric order of the `origin_server_ts` of their `m.space.child`
> events, or ascending lexicographic order of their `room_id`s in case of equal
> `origin_server_ts`. `order`s which are not strings, or do not consist solely
> of ascii characters in the range `\x20` (space) to `\x7E` (~), or consist of
> more than 50 characters, are forbidden and the field should be ignored if
> received.

This modifies the clauses for calculating the `origin_server_ts` of the
`m.room.create` event to refer to the `m.space.child` event instead. This allows
for a defined sorting of siblings based purely on the information available in
the `m.space.child` event while still allowing for a natural ordering due to the
age of the relationship.

## Potential issues

A large flat space (a single room with many `m.space.child` events) could cause
a large federation response

## Alternatives

Peeking to explore the room state could be used to build the tree of rooms/spaces,
but this would be significantly more expensive for both clients and servers. It
would also require peeking over federation (which is explored in
[MSC2444](https://github.com/matrix-org/matrix-doc/pull/2444)).

## Security considerations

A space with many sub-spaces and rooms on different homeservers could cause
a large number of federation requests. A carefully crafted space with inadequate
server enforced limits could be used in a denial of service attack. Generally
this is mitigated by enforcing server limits and caching of responses.

The requesting server over federation is trusted to filter the response for the
requesting user. The alternative, where the requesting server sends the requesting
`user_id`, and the target server does the filtering, is unattractive because it
rules out a caching of the result. This does not decrease security since a server
could lie and make a request on behalf of a user in the proper space to see the
given information. I.e. the calling server must be trusted anyway.

## Unstable prefix

During development of this feature it will be available at unstable endpoints.

The client-server API will be:
`/_matrix/client/unstable/org.matrix.msc2946/rooms/{roomID}/hierarchy`

The server-server API will be:
`/_matrix/federation/unstable/org.matrix.msc2946/hierarchy/{roomID}`

## Footnotes

<a id="f1"/>[1]: The rationale for including stripped events here is to reduce
potential dataleaks (e.g. timestamps, `prev_content`, etc.) and to ensure that
clients do not treat any of this data as authoritative (e.g. if it came back
over federation). The data should not be persisted as actual events.[↩](#a1)

<a id="f2"/>[2]: As a worked example, in the context of
[MSC3083](https://github.com/matrix-org/matrix-doc/pull/3083), consider that Alice
and Bob share a server; Alice is a member of a space, but Bob is not. A remote
server will not know whether the request is on behalf of Alice or Bob (and hence
whether it should share details of restricted rooms within that space).

Consider if the space is modified to include a restricted room on a different server
which allows access from the space. When summarizing the space, the homeserver must make
a request over federation for information on the room. The response should include
the room (since Alice is able to join it). Without additional information the
calling server does not know *why* they received the room and cannot properly
filter the returned results.

Note that there are still potential situations where each server individually
doesn't have enough information to properly return the full summary, but these
do not seem reasonable in what is considered a normal structure of spaces. (E.g.
in the above example, if the remote server is not in the space and does not know
whether the server is in the space or not it cannot return the room.)[↩](#a2)