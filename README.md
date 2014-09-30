unicorn-puppets
===============

If you run your puppet master with a nginx/unicorn stack you'll eventually notice a ton of "TrustedInformation expected a certificate, but none was given." warnings in your logs. The issue here is that the SSL_CLIENT_CERT environment variable is not getting set for each request. In this stack, you obviously can't do that, so you need to mangle the ssl client cert data and strip it of newlines, send it across to the rack app and use some middleware to reconstruct it.
