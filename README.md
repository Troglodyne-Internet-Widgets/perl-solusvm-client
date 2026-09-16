# NAME

SolusVM::Client - Client for the SolusVM 2 API, built from its own specification

# VERSION

version 0.001

# SYNOPSIS

```perl
use SolusVM::Client ();

my $solus = SolusVM::Client->new(
    host     => 'solus.example.tld',
    email    => 'you@example.tld',
    password => $ENV{SOLUSVM_PASSWORD},
);

my $servers = $solus->get_list_of_servers( 'filter[status]' => 'started' );
print "$_->{name}\n" for @{ $servers->{data} };

my $made = $solus->create_a_new_server(
    name      => 'web01.example.tld',
    plan      => 3,
    location  => 1,
    os        => 17,
    user_data => "#cloud-config\npackages:\n  - rsync\n",
);

$solus->delete_server( id => $made->{data}{id} );
```

# DESCRIPTION

Every method of this client comes out of the OpenAPI document SolusVM publishes
for version 2 of its API.  All 309 operations are here, and they are named after
the `operationId` the vendor gave them, which is why they read the way they do:
`get_list_of_servers`, `create_a_new_server`, `get_an_existing_api_token`.
Nothing is hand-maintained, so nothing drifts from the published reference, and
an endpoint added upstream arrives by regenerating
[SolusVM::Client::Specification](https://metacpan.org/pod/SolusVM%3A%3AClient%3A%3ASpecification) rather than by writing a method.

["catalog"](#catalog) is how you find the one you want.

## Calling an operation

Named arguments, in one flat list.  Anything the path declares as a placeholder
fills the placeholder:

```perl
$solus->get_an_existing_server( id => 42 );        # GET /servers/42
```

Everything left over goes into the request body if the operation declares one,
and into the query string if it is a listing.  That rule is unambiguous because
in the whole of the SolusVM API no operation takes both -- there is a test in
this distribution that fails if a future specification changes that.

The sixty-four write operations that declare no body -- every `delete_*`, and
the actions like `server_start` that are entirely described by their path --
take nothing else, and say so rather than quietly hanging an argument off the end
of the URL where the API will ignore it.

```perl
$solus->get_list_of_servers( page => 2, 'filter[status]' => 'started' );
$solus->server_start( id => 42 );
```

The decoded JSON document comes back whole, so a listing is
`$result->{data}` and its pagination is `$result->{meta}`.  A failure
dies.

# METHODS

## new

```perl
SolusVM::Client->new( host => 'solus.example.tld', token => $token );
```

Takes:

- `host` -- required, the management node's hostname
- `token`, or `email` and `password` -- see ["AUTHENTICATION"](#authentication)
- `expires_at` -- when a `token` you passed runs out, as the API spells it
- `scheme`, `port`, `prefix` -- default `https`, none, and `/api/v1`
- `timeout`, `verify_SSL` -- handed to [HTTP::Tiny](https://metacpan.org/pod/HTTP%3A%3ATiny); verification is on
- `ua` -- your own [HTTP::Tiny](https://metacpan.org/pod/HTTP%3A%3ATiny), if you have one
- `spec_file` -- a spec written by ["fetch" in SolusVM::Client::Specification](https://metacpan.org/pod/SolusVM%3A%3AClient%3A%3ASpecification#fetch), to run against an API newer than this release
- `debug` -- print each request and its response to STDERR, passwords redacted

## host

The management node this client talks to.

## base\_url

Everything an operation's URI hangs off: scheme, host, port if there is one, and
the API prefix.

## token

The bearer token in use, logging in first if there is not one yet.  Returns it,
which is what you want if the reason you passed credentials was to mint a token
to put somewhere else.

## catalog

Returns a printable listing of the operations, one per line, with the verb, the
path, what a listing can be filtered on, and the vendor's own summary:

```perl
print $solus->catalog( like => qr/snapshot/ );
```

`like` filters on the name, the path and the summary together.  With no
arguments you get all 309, which is a lot, and still the fastest way to find out
what the API is called.

## login

Exchanges the email and password for a token and remembers it, along with when it
expires.  Returns the credentials hashref the API answered with.  Called for you
when it is needed; call it yourself to find out now rather than later whether the
credentials are any good.

## paginate

```perl
my @servers = $solus->paginate( 'get_list_of_servers', 'filter[status]' => 'started' );
```

Walks a listing to its last page and returns every `data` element from all of
them.  The generated methods deliberately return one page: walking a listing of
unknown length is a decision, not something to have happen by surprise in the
middle of something else.

## last\_meta

## last\_links

## last\_response

The `meta` and `links` of the most recent answer, and the raw [HTTP::Tiny](https://metacpan.org/pod/HTTP%3A%3ATiny)
response hash behind it.  For the cases where a listing's totals matter, or where
a caller wants to look at a status code this client only turned into a message.

# AUTHENTICATION

The API takes a bearer token.  Give the constructor one you already have:

```perl
SolusVM::Client->new( host => ..., token => $token );
```

or give it the credentials to get one with, in which case it logs in when it
first needs to, renews the token before it expires, and renews it again if the
API rejects it anyway:

```perl
SolusVM::Client->new( host => ..., email => ..., password => ... );
```

Passing both uses the token and keeps the credentials for when it expires.  Where
those arguments come from is the caller's business: this distribution reads no
configuration file and no environment variable.

# AUTHORS

Current Maintainers:

- George S. Baugh <george@troglodyne.net>

# COPYRIGHT AND LICENSE

Copyright (c) 2026 Troglodyne LLC

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
