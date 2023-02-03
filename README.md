[![Build Status](https://travis-ci.org/powerman/perl-Mojolicious-Plugin-SecureCORS.svg?branch=master)](https://travis-ci.org/powerman/perl-Mojolicious-Plugin-SecureCORS)
[![Coverage Status](https://coveralls.io/repos/powerman/perl-Mojolicious-Plugin-SecureCORS/badge.svg?branch=master)](https://coveralls.io/r/powerman/perl-Mojolicious-Plugin-SecureCORS?branch=master)

# NAME

Mojolicious::Plugin::SecureCORS - Complete control over CORS

# VERSION

This document describes Mojolicious::Plugin::SecureCORS version v2.0.5

# SYNOPSIS

    # in Mojolicious app
    sub startup {
        my $app = shift;
        …

        # load and configure
        $app->plugin('SecureCORS');
        $app->plugin('SecureCORS', { max_age => undef });

        # set app-wide CORS defaults
        $app->routes->to('cors.credentials'=>1);

        # set default CORS options for nested routes
        $r = $r->under(…, {'cors.origin' => '*'}, …);

        # set CORS options for this route (at least "origin" option must be
        # defined to allow CORS, either here or in parent routes)
        $r->get(…, {'cors.origin' => '*'}, …);
        $r->any(…)->to('cors.origin' => '*');

        # allow non-simple (with preflight) CORS on this route
        $r->cors(…);

        # create under to protect all nested routes
        $r = $app->routes->under_strict_cors('/resource');

# DESCRIPTION

[Mojolicious::Plugin::SecureCORS](https://metacpan.org/pod/Mojolicious%3A%3APlugin%3A%3ASecureCORS) is a plugin that allow you to configure
Cross Origin Resource Sharing for routes in [Mojolicious](https://metacpan.org/pod/Mojolicious) app.

Implements this spec: [http://www.w3.org/TR/2014/REC-cors-20140116/](http://www.w3.org/TR/2014/REC-cors-20140116/).

## SECURITY

Don't use the lazy `'cors.origin'=>'*'` for resources which should be
available only for intranet or which behave differently when accessed from
intranet - otherwise malicious website opened in browser running on
workstation in intranet will get access to these resources.

Don't use the lazy `'cors.origin'=>'*'` for resources which should be
available only from some known websites - otherwise other malicious website
will be able to attack your site by injecting JavaScript into the victim's
browser.

Consider using `under_strict_cors()` - it won't "save" you but may helps.

# INTERFACE

## CORS options

To allow CORS on some route you should define relevant CORS options for
that route. These options will be processed automatically using
["after\_render" in Mojolicious](https://metacpan.org/pod/Mojolicious#after_render) hook and result in adding corresponding HTTP
headers to the response.

Options should be added into default parameters for the route or it parent
routes. Defining CORS options on parent route allow you to set some
predefined defaults for their nested routes.

- `'cors.origin' => '*'`
- `'cors.origin' => 'null'`
- `'cors.origin' => 'http://example.com'`
- `'cors.origin' => 'https://example.com http://example.com:8080 null'`
- `'cors.origin' => qr/\.local\z/ms`
- `'cors.origin' => undef` (default)

    This option is required to enable CORS support for the route.

    Only matched origins will be allowed to process returned response
    (`'*'` will match any origin).

    When set to false value no origins will match, so it effectively disable
    CORS support (may be useful if you've set this option value on parent
    route).

- `'cors.credentials' => 1`
- `'cors.credentials' => undef` (default)

    While handling preflight request true/false value will tell browser to
    send or not send credentials (cookies, http auth, SSL certificate) with
    actual request.

    While handling simple/actual request if set to false and browser has sent
    credentials will disallow to process returned response.

- `'cors.expose' => 'X-Some'`
- `'cors.expose' => 'X-Some, X-Other, Server'`
- `'cors.expose' => undef` (default)

    Allow access to these headers while processing returned response.

    These headers doesn't need to be included in this option:

        Cache-Control
        Content-Language
        Content-Type
        Expires
        Last-Modified
        Pragma

- `'cors.headers' => 'X-Requested-With'`
- `'cors.headers' => 'X-Requested-With, Content-Type, X-Some'`
- `'cors.headers' => qr/\AX-|\AContent-Type\z/msi`
- `'cors.headers' => undef` (default)

    Define headers which browser is allowed to send. Work only for non-simple
    CORS because it require preflight.

- `'cors.methods' => 'POST'`
- `'cors.methods' => 'GET, POST, PUT, DELETE'`

    This option can be used only for `cors()` route. It's needed in complex
    cases when it's impossible to automatically detect CORS option while
    handling preflight - see below for example.

## cors

    $app->routes->cors(...);

Accept same params as ["any" in Mojolicious::Routes::Route](https://metacpan.org/pod/Mojolicious%3A%3ARoutes%3A%3ARoute#any).

Add handler for preflight (OPTIONS) CORS request - it's required to allow
non-simple CORS requests on given path.

To be able to respond on preflight request this handler should know CORS
options for requested method/path. In most cases it will be able to detect
them automatically by searching for route defined for same path and HTTP
method given in CORS request. Example:

    $r->cors('/rpc');
    $r->get('/rpc', { 'cors.origin' => 'http://example.com' });
    $r->put('/rpc', { 'cors.origin' => qr/\.local\z/ms });

But in some cases target route can't be detected, for example if you've
defined several routes for same path using different conditions which
can't be checked while processing preflight request because browser didn't
sent enough information yet (like `Content-Type:` value which will be
used in actual request). In this case you should manually define all
relevant CORS options on preflight route - in addition to CORS options
defined on target routes. Because you can't know which one of defined
routes will be used to handle actual request, in case they use different
CORS options you should use combined in less restrictive way options for
preflight route. Example:

    $r->cors('/rpc')->to(
        'cors.methods'      => 'GET, POST',
        'cors.origin'       => 'http://localhost http://example.com',
        'cors.credentials'  => 1,
    );
    $r->any([qw(GET POST)] => '/rpc',
        headers => { 'Content-Type' => 'application/json-rpc' },
    )->to('jsonrpc#handler',
        'cors.origin'       => 'http://localhost',
    );
    $r->post('/rpc',
        headers => { 'Content-Type' => 'application/soap+xml' },
    )->to('soaprpc#handler',
        'cors.origin'       => 'http://example.com',
        'cors.credentials'  => 1,
    );

This route use "headers" condition, so you can add your own handler for
OPTIONS method on same path after this one, to handle non-CORS OPTIONS
requests on same path.

## under\_strict\_cors

    $route = $app->routes->under_strict_cors(...)

Accept same params as ["under" in Mojolicious::Routes::Route](https://metacpan.org/pod/Mojolicious%3A%3ARoutes%3A%3ARoute#under).

Under returned route CORS requests to any route which isn't configured
for CORS (i.e. won't have `'cors.origin'` in route's default parameters)
will be rendered as "403 Forbidden".

This feature should make it harder to attack your site by injecting
JavaScript into the victim's browser on vulnerable website. More details:
[https://code.google.com/p/html5security/wiki/CrossOriginRequestSecurity#Processing\_rogue\_COR:](https://code.google.com/p/html5security/wiki/CrossOriginRequestSecurity#Processing_rogue_COR:).

# OPTIONS

[Mojolicious::Plugin::SecureCORS](https://metacpan.org/pod/Mojolicious%3A%3APlugin%3A%3ASecureCORS) supports the following options.

## max\_age

    $app->plugin('SecureCORS', { max_age => undef });

Value for `Access-Control-Max-Age:` sent by preflight OPTIONS handler.
If set to `undef` this header will not be sent.

Default is 1800 (30 minutes).

# METHODS

[Mojolicious::Plugin::SecureCORS](https://metacpan.org/pod/Mojolicious%3A%3APlugin%3A%3ASecureCORS) inherits all methods from
[Mojolicious::Plugin](https://metacpan.org/pod/Mojolicious%3A%3APlugin) and implements the following new ones.

## register

    $plugin->register(Mojolicious->new);
    $plugin->register(Mojolicious->new, { max_age => undef });

Register hooks in [Mojolicious](https://metacpan.org/pod/Mojolicious) application.

# SEE ALSO

[Mojolicious](https://metacpan.org/pod/Mojolicious).

# SUPPORT

## Bugs / Feature Requests

Please report any bugs or feature requests through the issue tracker
at [https://github.com/powerman/perl-Mojolicious-Plugin-SecureCORS/issues](https://github.com/powerman/perl-Mojolicious-Plugin-SecureCORS/issues).
You will be notified automatically of any progress on your issue.

## Source Code

This is open source software. The code repository is available for
public review and contribution under the terms of the license.
Feel free to fork the repository and submit pull requests.

[https://github.com/powerman/perl-Mojolicious-Plugin-SecureCORS](https://github.com/powerman/perl-Mojolicious-Plugin-SecureCORS)

    git clone https://github.com/powerman/perl-Mojolicious-Plugin-SecureCORS.git

## Resources

- MetaCPAN Search

    [https://metacpan.org/search?q=Mojolicious-Plugin-SecureCORS](https://metacpan.org/search?q=Mojolicious-Plugin-SecureCORS)

- CPAN Ratings

    [http://cpanratings.perl.org/dist/Mojolicious-Plugin-SecureCORS](http://cpanratings.perl.org/dist/Mojolicious-Plugin-SecureCORS)

- AnnoCPAN: Annotated CPAN documentation

    [http://annocpan.org/dist/Mojolicious-Plugin-SecureCORS](http://annocpan.org/dist/Mojolicious-Plugin-SecureCORS)

- CPAN Testers Matrix

    [http://matrix.cpantesters.org/?dist=Mojolicious-Plugin-SecureCORS](http://matrix.cpantesters.org/?dist=Mojolicious-Plugin-SecureCORS)

- CPANTS: A CPAN Testing Service (Kwalitee)

    [http://cpants.cpanauthors.org/dist/Mojolicious-Plugin-SecureCORS](http://cpants.cpanauthors.org/dist/Mojolicious-Plugin-SecureCORS)

# AUTHOR

Alex Efros <powerman@cpan.org>

# COPYRIGHT AND LICENSE

This software is Copyright (c) 2014- by Alex Efros <powerman@cpan.org>.

This is free software, licensed under:

    The MIT (X11) License
