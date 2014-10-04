Next-Generation HTTP
====================

With [HTTP2](http://http2.github.io/) solving problems that are not
viewed by many in the web-development community as nessecary, I believe
that there are issues more important to the future of the web than
re-inventing TCP at Layer 7:

* Better authentication
* More secure caching
* Better methods to find alternate downloads locations
* Keeping the protocol simple
* Making each request contain less information about the sender
* Improved Metadata

These issues, and not serving super heavy-weight pages slightly faster,
should be the focus of improviments made to one of the most widely used
protocols on the internet.

Better Authentication
---------------------

[Mutual Authentication](http://en.wikipedia.org/wiki/Mutual_authentication), 
including the IETF draft from 2012 for
[HTTP-Based Mutual Authentication](https://tools.ietf.org/id/draft-oiwa-http-mutualauth-12.txt),
would be a good starting point. Mutual Authentication doesn't require
the user to transmit their credentials over the wire. Combined with
support from HTML and TLS (via TLS-SRP) to provide a better interface
with fewer round-trips, respectivly, could provide a user-experince
similar to what is currently found, while increasing security.

Secure Caching
--------------

[Content-Signature](https://tools.ietf.org/html/draft-burke-content-signature-00)
with the signing key being the TLS/SSL key. This could aid in secure
caching, mirroring, and proxying. This will help ensure that proxies and
mirrors are serving up a trustworthy copy of the resource.

Whatever standard is used for the `Content-Signature` header, there should
be a way to link to the signing key, so that it can be downloaded. This
still has the problem of being able to verify that the key being
downloaded actually belongs to the claimed signer. I don't think that
this is a problem that could be solved in a post about updating HTTP.
Regardless, the standard should have a standard way of giving this basic
information.

Deprecate `Content-MD5` in favour of a `Content-Hash` header whose value
would be a space delimited "algo hash". I feel that this is a more
future-proof and flexible header. This header could be specified
multiple times, each with a different algorithm. However, it also opens
the chance that a server could give a client a hash algorithm that it is
unfamiliar with and hence can't use. I would suggest that hash
algorithms of md5, sha1, sha224, sha256, sha384, sha512, and whirlpool
(or perhaps only sha512?) be standardly supported and that if a server
decides to send a function not on this list, it should additionally send
one from the list as well.

The encoding of the hash could be hex, base32, or base64 which are
fairly well-known standards. I, however, prefer base58 because I like
some of the reasonings found in the header for it in the bitcoin
project.

```
// Why base-58 instead of standard base-64 encoding?
// -- Don't want 0OIl [ed. note: zero, oh, capital i, lowercase ell]
//   characters that look the same in some fonts and could be used to
//   create visually identical looking account numbers.
// -- A string with non-alphanumeric characters is not as easily accepted
//    as an account number.
// -- E-mail usually won't line-break if there's no punctuation to break at
// -- Doubleclicking selects the whole number as one word if it's all
//    alphanumeric.
```

Specifically, not using characters that would be reserved in URLs and
not using characters that aren't visually identical or similar in many
fonts. Both of these can (and will) aid debugging for those cases in
which the hash must be read and interacted with by a human.

Deprecate `Etag` in favour of `If-Not-Hash` and `Content-Hash` tag.
`If-Not-Hash` would use the same semantics as the Content-Hash taga.One
downside, however, is that some `ETag` implementations are done in a way
such that the contents of the file don't need to be read (e.g.: Apache).
The internal representation could be used in a dictionary as the key to
the hash, so it may not be a terrible change or overhead.

Alternate Download methods
--------------------------

The `Magnet` header would contain a [magnet
link](https://en.wikipedia.org/wiki/Magnet_URI_scheme) for the file.  I
could see this being used in conjunction with the `Content-Range` header
to give information on how to use
[BitTorrent](https://en.wikipedia.org/wiki/BitTorrent) to download the
remainder of a file, say a CD or DVD image. Some browsers, namely
[Opera](http://help.opera.com/Windows/9.00/en/bittorrent.html), already
have clients that would support BitTorrent and could support this type
of header with minimal retooling. If the client would like to download
the entire file, it can resubmit the request with a `Range` header for the
rest of the file. The Server could respond with the rest of the file or
a redirect to a mirror. `Mirror` headers served with a `Magnet` header
should be checked before resubmitting the request.

The `Mirror` header would contain the URL of a mirror of the resource.
This header could be repeated multiple times in order to specify
multiple mirrors.  In a similar manner to the `Magnet` header, if the
server would prefer the User Agent use mirrors, then it could respond
with a `Content-Range` header so that it doesn't have to deliver the
entire resource.

Sending less information
-----------------------

Many headers, such as `Date`, `Via`, `User-Agent`, `DNT`, `Pragma`,
`Server`, and `P3P` are useless and take up bandwidth. They also enable
techniques such as [PanOptiClick](https://panopticlick.eff.org/) that
enable tracking of users.

Improved Metadata
-----------------

Supporting some basic metadata, maybe starting with the [Dublin
Core](http://en.wikipedia.org/wiki/Dublin_Core) fields. Metadata as
header-information. Perhaps in the following format: Metadata: `field`
`value`. The headers would be cumulative, and not overwrite previous
headers with a different field.

example:

```
Metadata: Title HTTP NG Specification
Metadata: Creator HTTP NG WG
Metadata: Description This document aims to create an update to \
          HTTP 1.1 that is also backwards compatible, and without \
          the complexity of "HTTP2"
Metadata: Language en-US
Metadata: Rights GNU FDL
```
