unicorn-puppets
===============

If you run your puppet master with a nginx/unicorn stack you'll eventually notice a ton of "TrustedInformation expected a certificate, but none was given." warnings in your logs. The issue here is that the SSL_CLIENT_CERT environment variable is not getting set for each request. In this stack, you obviously can't do that, so you need to mangle the ssl client cert data and strip it of newlines, send it across to the rack app and use some middleware to reconstruct it.

###Setup
1. Modify your nginx puppetmaster virtualhost configuration file to contain the following code at the top of your file, outside of the server {} definition. It creates an nginx map to strip the newlines out of the client cert. This allows you to transfer the client cert through HTTP headers without violating any RFCs and avoids the nasty 400 error you'd get if you tried to send it through with newlines.

  ```
  map $ssl_client_raw_cert $a {
  "~^(-.*-\n)(?<1st>[^\n]+)\n((?<b>[^\n]+)\n)?((?<c>[^\n]+)\n)?((?<d>[^\n]+)\n)?((?<e>[^\n]+)\n)?((?<f>[^\n]+)\n)?((?<g>[^\n]+)\n)?((?<h>[^\n]+)\n)?((?<i>[^\n]+)\n)?((?<j>[^\n]+)\n)?((?<k>[^\n]+)\n)?((?<l>[^\n]+)\n)?((?<m>[^\n]+)\n)?((?<n>[^\n]+)\n)?((?<o>[^\n]+)\n)?((?<p>[^\n]+)\n)?((?<q>[^\n]+)\n)?((?<r>[^\n]+)\n)?((?<s>[^\n]+)\n)?((?<t>[^\n]+)\n)?((?<v>[^\n]+)\n)?((?<u>[^\n]+)\n)?((?<w>[^\n]+)\n)?((?<x>[^\n]+)\n)?((?<y>[^\n]+)\n)?((?<z>[^\n]+)\n)?(-.*-)$" $1st;
}
  ```
  
2. Within the server {} block, add the following line to send the client cert across.

  ```
  proxy_set_header SSL-CLIENT-CERT $a$b$c$d$e$f$g$h$i$j$k$l$m$n$o$p$q$r$s$t$v$u$w$x$y$z;
  ```
  
3. In your puppetmaster config.ru file add the following middleware code at the top of the file to convert the HTTP header back in to an environment variable for puppet to consume.

  ```
  class String
    def in_groups_of(n)
      chars.each_slice(n).map(&:join).join("\n")
    end
  end

  class SSL_Middleware
    def initialize(app, options = {})
      @app = app
      @options = options
    end

    def call(env)
      env["SSL_CLIENT_CERT"] = "-----BEGIN CERTIFICATE-----\n%s\n-----END CERTIFICATE-----" %     [env["HTTP_SSL_CLIENT_CERT"].in_groups_of(65)]
      # continue processing
      @app.call(env)
    end
  end

  use SSL_Middleware
  ```
  
4. Restart unicorn and nginx and enjoy!
