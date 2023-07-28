# Nginx Digest Authentication
## Description
This implementation of [digest authentication](https://en.wikipedia.org/wiki/Digest_access_authentication) is derived from the [existing digest module](https://github.com/atomx/nginx-http-auth-digest) with a few new directives for our authentication purposes.

### Copyright Info/History:
 * copyright © Daktronics Inc. 2023-presesnt
 * fork from nginx-http-auth-digest © Erik Dubbelboer
 * fork from nginx-http-auth-digest © samizdat drafting co.
 * derived from http_auth_basic © igor sysoev

## Changes from other forks
Bug fixes
[1](https://github.com/samizdatco/nginx-http-auth-digest/commit/9d77dcc58420d5afb8aa5a8138b1bf22a1933dd6), 
[2](https://github.com/samizdatco/nginx-http-auth-digest/commit/b98725d3d0506c895f6a9f9d38f9168d499275fc),
[3](https://github.com/samizdatco/nginx-http-auth-digest/commit/47d5bac13cf071b4dbe81048b0f12a742ba512ae>)

[Added log message for invalid login attempts](https://github.com/samizdatco/nginx-http-auth-digest/commit/9a402045082291c1f2f0a432ac24475277e2d176)

## Example
You can password-protect a directory tree by adding the following lines into
a ``server`` section in your Nginx_ configuration file:

    auth_digest_user_file /opt/httpd/conf/passwd.digest; # a file created with htdigest
    location /private {
        auth_digest 'this is not for you'; # set the realm for this location block
    }


The other directives control the lifespan defaults for the authentication session. The 
following is equivalent to the previous example but demonstrates all the directives:

    auth_digest_user_file /opt/httpd/conf/passwd.digest;
    auth_digest_shm_size 4m;   # the storage space allocated for tracking active sessions

    location /private {
        auth_digest 'this is not for you';
        auth_digest_timeout 60s; # allow users to wait 1 minute between receiving the
                             # challenge and hitting send in the browser dialog box
        auth_digest_expires 10s; # after a successful challenge/response,
                            # let the client continue to use the same nonce for
                            # additional requests for 10 seconds 
                            # before generating a new challenge
        auth_digest_replays 20;  # also generate a new challenge if the client uses the
                                # same nonce more than 20 times before the
                                # expire time limit
    }

Adding digest authentication to a location will affect any uris that match that block. To
disable authentication for specific sub-branches off a uri, set ``auth_digest`` to ``off``:

    location / {
        auth_digest 'this is not for you';
        location /pub {
            auth_digest off; # this sub-tree will be accessible without authentication
        }
    }

## Directives
These are the directives specific to the Digest Authentication


### auth_digest
|   |   |
|-------|------|
| Syntax: | ``auth_digest`` [*realm-name* \| ``off``] |
| Default: | ``off`` |
| Context: | server, location |
| Description: | Enable or disable digest authentication for a server or location block. The realm name should correspond to a realm used in the user file. Any user within that realm will be able to access files after authenticating. To selectively disable authentication within a protected uri hierarchy, set ``auth_digest`` to “``off``” within a more-specific location block (see example).
|  
  
### auth_digest_user_file
|   |   |
|-------|------|
| Syntax: | ``auth_digest_user_file`` */path/to/passwd/file* |
| Default: | *unset* |
| Context: | server, location |
| Description: | The password file should be of the form created by the apache ``htdigest`` command (or the included `htdigest.py`_ script). Each line of the file is a colon-separated list composed of a username, realm, and md5 hash combining name, realm, and password. For example: ``joi:enfield:ef25e85b34208c246cfd09ab76b01db7`` This file needs to be readable by your nginx user!
|

### auth_digest_timeout
|   |   |
|-------|------|
| Syntax: | ``auth_digest_timeout`` *delay-time* |
| Default: | ``60s`` |
| Context: | server, location |
| Description: | When a client first requests a protected page, the server returns a 401 status code along with a challenge in the ``www-authenticate`` header. At this point most browsers will present a dialog box to the user prompting them to log in. This directive defines how long challenges will remain valid. If the user waits longer than this time before submitting their name and password, the challenge will be considered ‘stale’ and they will be prompted to log in again.
|

### auth_digest_expires
|   |   |
|-------|------|
| Syntax: | ``auth_digest_expires`` *lifetime-in-seconds* |
| Default: | ``10s`` |
| Context: | server, location |
| Description: | Once a digest challenge has been successfully answered by the client, subsequent requests will attempt to re-use the ‘nonce’ value from the original challenge. To complicate [Man-In-The-Middle](http://en.wikipedia.org/wiki/Man-in-the-middle_attack) attacks, it's best to limit the number of times a cached nonce will be accepted. This directive sets the duration for this re-use period after the first successful authentication.
|

### auth_digest_replays
|   |   |
|-------|------|
| Syntax: | ``auth_digest_replays`` *number-of-uses* |
| Default: | ``20`` |
| Context: | server, location |
| Description: | Nonce re-use should also be limited to a fixed number of requests. Note that increasing this value will cause a proportional increase in memory usage and the shm_size may have to be adjusted to keep up with heavy traffic within the digest-protected location blocks.
|

### auth_digest_evasion_time
|   |   |
|-------|------|
| Syntax: | ``auth_digest_evasion_time`` *time-in-seconds* |
| Default: | ``300s`` |
| Context: | server, location |
| Description: | The amount of time for which the server will ignore authentication requests from a client address once the number of failed authentications from that client reaches ``auth_digest_maxtries``.
|

### auth_digest_maxtries
|   |   |
|-------|------|
| Syntax: | ``auth_digest_maxtries`` *number-of-attempts* |
| Default: | ``5`` |
| Context: | server, location |
| Description: | The number of failed authentication attempts from a client address before the module enters evasive tactics. For evasion purposes, only network clients are tracked, and only by address (not including port number).  A successful authentication clears the counters.
|

### auth_digest_shm_size
|   |   |
|-------|------|
| Syntax: | ``auth_digest_shm_size`` *size-in-bytes* |
| Default: | ``4096k`` |
| Context: | server |
| Description: | The module maintains a fixed-size cache of active digest sessions to save state between authenticated requests. Once this cache is full, no further authentication will be possible until active sessions expire. As a result, choosing the proper size is a little tricky since it depends upon the values set in the expiration-related directives. Each stored challenge takes up ``48 + ceil(replays/8)`` bytes and will live for up to ``auth_digest_timeout + auth_digest_expires`` seconds. When using the default module settings this translates into allowing around 82k non-replay requests every 70 seconds.
|

*Here is where our custom directives begin:

### auth_digest_allow_localhost
|   |   |
|-------|------|
| Syntax: | ``auth_digest_allow_localhost`` [``on`` \| ``off``] |
| Default: | ``off`` |
| Context: | server, location |
| Description: | When enabled, this directive will disable authentication for ``localhost`` and ``127.0.0.1`` HTTP requests. This is to give us greater control of our authentication, especially where our internal networks are concerned. It is disabled by default for better security.
|

### auth_digest_use_basic
|   |   |
|-------|------|
| Syntax: | ``auth_digest_use_basic`` [``on`` \| ``off``] |
| Default: | ``off`` |
| Context: | server, location |
| Description: | This directive enables the use of Basic Authentication to validate against a specified user in the operating system registry. 
|

### user_agents_allow_basic
|   |   |
|-------|------|
| Syntax: | ``user_agents_allow_basic`` *comma-separated list of user agents* |
| Default: | *unset* |
| Context: | server, location |
| Description: | This directive specifies certain user-agents as users of the basic/OS authentication scheme described above. Any user-agent in the list will be prompted to log in via Basic Authentication versus Digest Authentication.
|