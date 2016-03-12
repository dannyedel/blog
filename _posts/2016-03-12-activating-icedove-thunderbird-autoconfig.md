---
title: Activating Icedove/Thunderbird autoconfiguration
---

For a while now, Icedove/Thunderbird has the feature
to be configured only by entering
the user's *Full Name*, *e-mail address* and *password*.

This method is fully documented in the
[upstream wiki page about Autoconfiguration][upwiki],
so please read this for the full story.

I'm using `nginx` as a web server,
so adapt this as needed.


## Default config for the whole domain

If the whole domain is owned by one user account
(catch-all method) then a simple xml file will suffice.

The file name thunderbird will check is

```
/.well-known/autoconfig/mail/config-v1.1.xml
```

Upload a file like similar to the following ([download xml][xml]) and it will just work.
My example uses IMAP/TLS (port 993) and
Mail Submission with STARTTLS (port 587).
These ports are allowed to be accessed from most networks,
especially the *eduroam* system that pretty much
every universitiy around here uses.

{% gist dannyedel/489ae2a03a8fb2503520 %}

## Overriding this config for specific mail addresses

Icedove/Thunderbird will also check a file containing the
full mail address, URL encoded.
Example for "myaddress@example.org":

```
/.well-known/autoconfig/mail/config-v1.1.xml?emailaddress=myaddress%40example.org
```

So, just upload an additional file with this exact filename.

Try to access it, and the server will most likely just serve the
`config-v1.1.xml` file, ignoring the part after the `?`.

### Configure nginx to serve the file containing a question mark

nginx has the `try_files` configuration keyword,
where you can tell it to try various files in order.

What we want it to do is

1. the full file `config-v1.1.xml?emailaddress=somestring`
1. fall back to `config-v1.1.xml`
1. Answer with 404 if neither exists

Also, if we do this, make sure to serve these files -- *which
do no longer end in .xml* - with Content-type `text/xml`.
The default Content-type of `application/octet-stream` will
be ignored by Icedove/Thunderbird.

```
location /.well-known/autoconfig/mail {
	default_type text/xml ;
	try_files $request_uri $uri$is_args$args $uri =404;
}
```

This will first try an exact match on `$request_uri`,
then fall back to constructing `$uri`, the `$is_args` variable
(being a questionmark) and then `$args`.
Also this sets the default mime type to text/xml, but only
for that folder.

Now check your webserver with `curl`:

```bash
curl -v http://danny-edel.de/.well-known/autoconfig/mail/config-v1.1.xml?emailaddress=doesntexist%40danny-edel.de
# This should answer with the catch-all
curl -v http://danny-edel.de/.well-known/autoconfig/mail/config-v1.1.xml?emailaddress=actuallyexists%40danny-edel.de
# This should answer with the specific xml
```

Make sure to check the `Content-type` returned from
the web server.  It must be `text/xml` for both.


[upwiki]: https://developer.mozilla.org/en-US/docs/Mozilla/Thunderbird/Autoconfiguration
[xml]: /assets/downloads/config-v1.1.xml
