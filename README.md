Next-Generation HTTP
====================

With [HTTP2](http://http2.github.io/) solving problems that are not
viewed by many in the web-development community as necessary, I believe
that there are issues more important to the future of the web than
re-inventing TCP at Layer 7:

* Better authentication
* More secure caching
* Better methods to find alternate downloads locations
* Keeping the protocol simple
* Making each request contain less information about the sender
* Improved Metadata

These issues, and not serving super heavy-weight pages slightly faster,
should be the focus of improvements made to one of the most widely used
protocols on the internet.

Better Authentication
---------------------

[Mutual Authentication](http://en.wikipedia.org/wiki/Mutual_authentication),
including the IETF draft from 2012 for
[HTTP-Based Mutual Authentication](https://tools.ietf.org/id/draft-oiwa-http-mutualauth-12.txt),
would be a good starting point. Mutual Authentication doesn't require
the user to transmit their credentials over the wire. Combined with
support from HTML and TLS (via TLS-SRP) to provide a better interface
with fewer round-trips, respectively, could provide a user-experince
similar to what is currently found, while increasing security.

Additionally, having browsers support a standard could allow `method="mutual-auth`
for form elements to avoid one of the main issues with HTTP auth, namely
that people consider it ugly and unusable.

Another issue that plauges HTTP auth is the inability to logout.  There
should be a clearly-defined manner in which the user-agent can be asked to
drop its credentials by either the server or client.

However, part of me feels that this should be pushed into another
protocol, perhaps just using TLS-SRP and building the infrastructure in
both the browser and server to support this. However, that wouldn't be
in the purvue of an HTTP WG. Whether a stateful authentication exchange
has a place in HTTP is to be determined. I feel that by placing it as a
lowerst-common-denominator it will become more wide-spread which may
outweigh some of the technical ugliness it would have?

One possible method would be for the `WWW-Authenticate` header to provide
an additional parameter as to where this service could be located. Example:

```
GET /protected HTTP/`.3
Host: example.com

HTTP/1.3 401 Authentication Required
WWW-Authenticate: Mutual realm="a realm" endpoint="srp://example.com:81"
```

This endpoint would perform the SRP transaction leaving both the server and
user-agent with the session key-username pair.

While this veers even further from HTTP, the authentication could conclude
with the server passing a token (e.g. customer session key-expire-hmac
token, or a [JWT](http://jwt.io/)).  Otherwise the session key should be
used as-is, preferably in an `Authentication: Token` header.

Now may also be a good time to consider
[SASL](http://tools.ietf.org/html/rfc4422) compatibility as well.

Secure Caching
--------------

[Content-Signature](https://tools.ietf.org/html/draft-burke-content-signature-00)
(with the concatenation problem fixed, i.e. concatenated fields for the
sig could be prefixed with a fixed-length length-of-field)(also,
signatures should be able to be in other encodings than hex) could aid
in secure caching, mirroring, and proxying. This will help ensure that
proxies and mirrors are serving up a trustworthy copy of the resource.

Whatever standard is used for the `Content-Signature` header, there should
be a way to link to the signing key, so that it can be downloaded. This
still has the problem of being able to verify that the key being
downloaded actually belongs to the claimed signer. I don't think that
this is a problem that could be solved in a post about updating HTTP.
Regardless, the standard should have a standard way of giving this basic
information.

Deprecate `Content-MD5` in favour of a `Content-Hash` header whose value
would be a key/value pair of algorithm name and hash. I feel that this is a more
future-proof and flexible header. This header could be specified
multiple times, each with a different algorithm. However, it also opens
the chance that a server could give a client a hash algorithm that it is
unfamiliar with and hence can't use. I would suggest that hash
algorithms of md5, sha1, sha224, sha256, sha384, sha512, and whirlpool
(or perhaps only sha512?) be standardly supported and that if a server
decides to send a function not on this list, it should additionally send
at least one from the list as well.

Advocate for the use of `If-Not-Hash` and `Content-Hash` tag.
`If-Not-Hash` would use the same semantics as the Content-Hash taga.One
downside, however, is that some `ETag` implementations are done in a way
such that the contents of the file don't need to be read (e.g.: Apache).

Alternate Download methods
--------------------------

The `Mirror` header would contain the type (via the URI schema specifier)
and URL of a mirror for the resource. This header could contain a list
in order to specify multiple mirrors and download methods
(e.g. http, bittorent (via magnet uris), ftp, or jigdo).

If a server would prefer a client use a mirror, requests without a
`Range` can respond with a correct `Content-Length` and a `Content-Range`
of 0, and one or more `Mirror` headers. Servers are encouraged to also send
`Content-Hash` and `Content-Signature` headers as well. If the client
would like to download the entire file, it can resubmit the request with
a `Range` header for the rest of the file. The Server could respond with
the rest of the file or a redirect to an HTTP mirror.

(In theory magnet urls could be entirely/mostly filled in from the other
headers.)

Example:

```
GET /hello HTTP/1.3
Host: example.com

HTTP/1.3 206 Partial Content
Content-Hash: algorithm=sha256;
              hash=2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824,
              algorithm=sha1;
              hash=aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d,
              algorithm=md5;
              hash=5d41402abc4b2a76b9719d911017c592
Content-Signature: id=https://example.com/signing_key.pub;
                   algorithm=ecdsa:secp384r1;
                   signature=3064023068985da8ab2485f30dc0ca1cd04ae330221c04755acb716d0175
                             8ba3e17225e130ea6bf0e0910ec045f19b7c53dd234602302dc29d827c5c
                             f518efe551d639b32644ebcc452197ed907933d6fdd8e565ac5a52366fbf
                             c8b1786713937d1db363058c
Content-Length: 5
Content-Rang: bytes 0
Content-Type: text/plain
Last-Modified: Tue, 01 Jan 1833 00:00:00 GMT
Mirror: magnet:&xl=5&dn=hello&xt=urn:md5:5d41402abc4b2a76b9719d911017c592,
        https://mirror-a.example.com/hello,
        https://mirror-b.example.com/hello,
        ftp://mirror-c.example.com/hello,
        jigdo://example.com/hello.jigdo

```

Sending less information
-----------------------

Many headers, such as `Date`, `Via`, `User-Agent`, `DNT`, `Pragma`,
`Server`, and `P3P` are useless and take up bandwidth. The client headers
also enable techniques such as [PanOptiClick](https://panopticlick.eff.org/)
that enable tracking of users.

MIME-Multi-part
---------------

In addition to sending fewer headers, an server so designed could send
MIME-multipart messages that would contain, for instance, images,
scripts, and stylesheets with the main content body.  One issue with
this is knowing if the clinent currently requires these assets or not
(e.g. because they are cached). Instead, the next request to the server could
contain multiple resources and their corrsponding `ETag` or `Content-Hash`.

Improved Metadata
-----------------

Supporting some basic metadata, maybe starting with the [Dublin
Core](http://en.wikipedia.org/wiki/Dublin_Core) fields. Metadata as
header-information.

Example (note the backslash-escaped comma in the description):

```
Metadata: title=HTTP NG Specification,
          creator=HTTP NG WG,
          description=This document aims to create an update to
                      HTTP 1.1 that is also backwards compatible\054 and without
                      the complexity of "HTTP2",
          rights=CC-By-SA,
          rights-url=http://creativecommons.org/licenses/by-sa/3.0/
```

Join the Discussion
-------------------

Feel free to fork and make pull requests for ideas.

We're also on freenode @ #http_ng ([WebChat](http://webchat.freenode.net/?channels=http_ng))
