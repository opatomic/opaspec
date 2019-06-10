# Opatomic RPC protocol

This protocol utilizes the [Opatomic serialization format](serialization.md). Refer to that
document to understand how to serialize individual objects.


## Request

    [command, params, asyncid]

The client sends a request to the server. It is an array containing a command (required),
params array (optional), and asyncid (optional). If the request includes params, it must be
an array or null. If the command includes an asyncid, it can be any type except undefined
or sortmax.

Examples:

    ["PING"]
    ["PING", null]
    ["PING", null, null] <--- will not receive a response from server
    ["PING", null, 987]  <--- response can be received out of order
    ["PING", []]
    ["ECHO", ["hello"]]
    ["INCR", ["key", -123.45]]

### asyncid

The asyncid is used to receive responses out of order so that they do not block other responses.
Any request can include an asyncid and the server must respond with the exact same asyncid. The
asyncid should be unique. If the asyncid is a string then it should not start with an underscore
character "_" (used by server to send messages). If the asyncid is null then the server will
not send a response for the request. Some commands (such as SUBSCRIBE) can send more than 1
response. The server can send an unsolicited message to the client by using an asyncid that is
a string starting with an underscore "_" character.


## Response

    [result, error, asyncid]

The server's response is an array containing the result, an error (optional), and the request's
asyncid (optional). The result must be null if there is an error; otherwise error must not be
present or be null. If a request contains an asyncid then the response must include the exact
same asyncid. The async id cannot be sent back from server as a canonical equivalent - it must
be the exact same bytes (ie, cannot normalize a string or convert an int to a bigint, or shift
the exponent of a dec, etc). Requests can receive more than 1 response if they include an asyncid
(depends on the command). All responses that do not contain an asyncid must be sent in the order
the requests were received.

Examples:

    ["PONG"]
    ["PONG", null]
    ["PONG", null, 987]                   <--- response to a request that contained an asyncid
    [null, -349]                          <--- error code
    [null, [-349, "An error occurred!"]]  <--- error code + message
    [null, [-349, "An error occurred!", "some application-specific junk describing the error context"], 457]

### error

If present, must be an integer, or an array. An integer represents the error code and must be
within the range of a signed 32-bit integer but not 0 (also must be serialized as a positive or
negative varint). If error is an array then the first item must be an integer (error code), the
second item must be a string (error message), and the third item (if present) can be any object
that represents extra error information. The error array length must be 2 or 3. The error string
should be 1 concise sentence.

Reserved ranges:

    1-63 - reserved for definition by this protocol

Defined error codes:

    1 - unknown command
    2 - invalid param count for specified command
    3 - invalid param
    4 - internal server error
    5 - async id required for specified command
    6 - request parse error (server should send this error code response and terminate connection)
    7 - request too big

