# Opatomic RPC protocol

This protocol utilizes the [Opatomic serialization format](serialization.md). Refer to that
document to understand how to serialize individual objects.


## Request

    [asyncid, command, arg1, arg2, ... argn]

The client sends a request to the server. It is an array containing an asyncid, command,
and any number of optional parameters. The asyncid can be any type except undefined or sortmax.
The request array size must be greater than or equal to 2. The asyncid has special meaning if
it is null or false (see next sections).

Examples:

    [null,  "PING"]
    [false, "PING"]  <--- will not receive a response from server
    [987,   "PING"]  <--- response can be sent/received out of order
    [null,  "ECHO", "hello"]
    [null,  "INCR", "key", -123.45]

### asyncid

The asyncid is used to receive responses out of order so that they do not block other responses.
Every request includes an asyncid and the server must respond with the exact same asyncid. The
asyncid should be unique (unless it is null). If the asyncid is a string then it should not start
with an underscore character "\_" (used by server to send messages). If the asyncid is false then
the server will not send a response for the request. Some commands (such as SUBSCRIBE) can send
more than 1 response. These commands must use a non-null async id. The server can send an
unsolicited message to the client by using an asyncid that is a string starting with an
underscore "\_" character.

#### null async id

Clients can pipeline multiple null asyncid requests and the server must respond to all of the
null asyncid requests in the order that they were received. Only non-null asyncid responses can
be sent out of order.


## Response

    [asyncid, result, error]

The server's response is an array containing the request's asyncid, a result, and an
error (optional). The result must be null if there is an error; otherwise error must not be
present or be null. The async id cannot be sent back from server as a canonical equivalent - it must
be the exact same bytes (ie, cannot normalize a string or convert an int to a bigint, or shift
the exponent of a dec, etc). Requests can receive more than 1 response (depends on the command).
All responses that have a null asyncid must be sent in the order the requests were received.

Examples:

    [null, "PONG"]
    [null, "PONG", null]
    [987,  "PONG", null]   <--- response to a request that contained an asyncid
    [null, null, -349]                          <--- error code
    [null, null, [-349, "An error occurred!"]]  <--- error code + message
    [457,  null, [-349, "An error occurred!", "some application-specific junk describing the error context"]]

### error

If non-null, must be an integer or an array. An integer represents the error code and must be
within the range of a signed 32-bit integer but not 0 (also must be serialized as a positive or
negative varint). If error is an array then the first item must be an integer (error code), the
second item must be a string (error message), and the third item (if present) can be any object
that represents extra error information. The error array length must be 2 or 3. The error string
should be 1 concise sentence.

Error code reserved range:

    1-63 - reserved for definition by this protocol

Defined error codes:

    1 - unknown command
    2 - invalid param count for specified command
    3 - invalid param
    4 - internal server error
    5 - async id required for specified command
    6 - request parse error (server should send this error code response and terminate connection)
    7 - request too big

