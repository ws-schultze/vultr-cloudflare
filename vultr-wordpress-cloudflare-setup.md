# Setting Up a Vultr Server with WordPress, NGINX, and Cloudflare

**Note:** for this entire walkthrough, let’s assume that the website we are making is called example.com

## Deploy the Server

1. Deploy an Ubuntu server with WordPress pre-installed.

2. Wait for DNS propagation; check it at [whatsmydns.net](http://whatsmydns.net).

3. Go to the server’s IP address (e.g. 111.222.333.44) and see if the default WordPress site is visible.

## Manage DNS Records Within Your Domain Name Registrar Account

**`🔔 Note`** The following two steps can be skipped if you would rather add the necessary A records from within Cloudflare. Once you set up Cloudflare as a proxy, the DNS settings in your registrar account no longer are used.

1. Add the following two A records:

   1. Host: @, Answer: IP of the server you just deployed, TTL: Auto

   2. Host: www, Answer: IP of the server you just deployed, TTL: Auto

2. Check that both [example.com](http://example.com) and [www.example.com](http://www.example.com) both go to the same place.

## Add the Website to Cloudflare

1. Inside your Cloudflare account, navigate to the websites tab and click "+ Add a site”.

2. Select the free plan for now.

3. Follow the instructions:

   1. Make sure the DNS records look as they do in your registrar account.

   2. Change the name servers in your registrar account to the Cloudflare name servers.

   3. Wait for the name server changes to take effect.

   4. Ping [example.com](http://example.com) and check that Cloudflare is masking the server’s original IP:

      - Open a terminal and enter the following command:

        ```bash
        ping example.com -c 4
        ```

      - You should get a response with something like:

        ```bash
        PING www.example.com (123.45.67.890): 56 data bytes
        64 bytes from 123.45.67.890: icmp_seq=0 ttl=49 time=25.937 ms
        64 bytes from 123.45.67.890: icmp_seq=1 ttl=49 time=44.770 ms
        64 bytes from 123.45.67.890: icmp_seq=2 ttl=49 time=58.170 ms
        64 bytes from 123.45.67.890: icmp_seq=3 ttl=49 time=72.352 ms
        ```

      **Note:** that the IP address should be different from the server’s IP address; however, this change could take a day to propagate.

At this point, you (the client) should have a secure connection to Cloudflare's proxied connection to the server (origin). To verify this, go to [https://example.com](https://example.com) and see if the connection is secure.

## Configure Cloudflare to Redirect HTTP to HTTPS

1. Go to the Edge Certificates tab and enable the Always Use HTTPS option.

2. Go to the Rules tab and add a rule to always use HTTPS.

   1. Click "+ Create rule".

   2. Rule name: Always Use HTTPS

   3. In the "If.... When incoming requests match..." field, select "All incoming requests".

   4. In the "Then the settings are..." field, select "+ Add" button next to "Automatic HTTPS Rewrites" then toggle it on.

   5. Go to the bottom of the page and click "Deploy".

Now go to [http://example.com](http://example.com) and see if the connection is redirected to [https://example.com](https://example.com).

## Change the WordPress Address and Site Address to HTTPS

Notice that the images (if you have any) are probably not loading. This is likely because WordPress is still using HTTP. That just won't do, so let's change that.

1. Go to the WordPress admin panel and login with the credentials you can find on your Vultr server's information, then navigate to Settings > General.

2. Change the WordPress Address (URL) and Site Address (URL) to [https://example.com](https://example.com).

   **_`🔔 Alert`_** The whole point of using Cloudflare to proxy the connection is to mask the server's actual address, so do yourself a favor and make sure both of the aforementioned URLs point to the domain name, not the server IP.

3. Save the changes.

4. Go to [https://example.com](https://example.com) and see if the images are loading.

   Whew, that was a lot of work, but you are almost there. Now let's make sure the connection between Cloudflare and the server is secure.

## Configure Nginx to Use Cloudflare Origin CA Certificates

Your SSL/TLS encryption mode is set to "Flexible" by default in Cloudflare, which means that the connection between the client and Cloudflare is secure, but the connection between Cloudflare and the server is not. Because this site will only be accessed through Cloudflare's proxy, we can use Cloudflare’s Origin CA to generate certificates for your server. Let's make the server's connection secure.

1. Go to the SSL/TLS tab in Cloudflare and set the encryption mode to **Full (strict)**.

2. Go to [example.com](https://example.com) and see the lovely error code **526 "Invalid SSL certificate"**. This is because the server is not using a certificate that Cloudflare recognizes.

3. Go to the **SSL/TLS > Origin Server** tab in Cloudflare.

4. Click on **Create Certificate**.

5. Leave the details at default settings unless you want to customize them.

6. Copy and paste the certificate and private key into separate files locally. These will be copied onto the server in a moment.

7. SSH into the server. Doing this in VSCode makes things easier if you are not very comfortable in the terminal with vi or nano.

8. Copy the certificate and key that you got from Cloudflare a moment ago into the directory.

   ```bash
   sudo cp ~/path/to/origin_certificate.pem /etc/nginx/ssl/origin_certificate.pem
   sudo cp ~/path/to/private_key.pem /etc/nginx/ssl/private_key.pem
   ```

   Some server configurations require the full chain certificate, so let's create that now. For more information, see the [Nginx documentation](https://nginx.org/en/docs/http/configuring_https_servers.html#chains) and [Cloudflare documentation](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/).

9. Get the Cloudflare Origin CA root certificate and put it in the same directory as the other certificates.

   ```bash
   sudo wget https://developers.cloudflare.com/ssl/static/origin_ca_rsa_root.pem -O /etc/nginx/ssl/origin_ca_rsa_root.pem
   ```

10. Combine the Cloudflare Origin CA root certificate and the origin certificate into a single file. The order of the certificates in `full_chain.pem` is important. Ensure that the origin certificate is first and the root certificate is second.

    ```bash
    sudo cat /etc/nginx/ssl/origin_certificate.pem /etc/nginx/ssl/origin_ca_rsa_root.pem > /etc/nginx/ssl/full_chain.pem
    ```

## Configure Nginx to Use the Certificates

1. Open the Nginx configuration file:

   ```bash
   sudo nano /etc/nginx/sites-available/your-site-name.conf
   ```

   Preinstalled WordPress will put two .conf files for your site in the `/etc/nginx/sites-available` directory, one is for HTTP and the other is for HTTPS. You will need to edit them both or combine them into one file, and edit that, your call. The directory could look something like this:

   ```text
   cockpit.conf
   default.conf
   your-site-name_http.conf
   your-site-name_https.conf
   ```

2. Edit the HTTP file to include the following server properties.

   ```nginx
   server {
      listen 80;
      server_name example.com www.example.com; # Point the server to the domain name
      return 301 https://$host$request_uri; # Redirect all HTTP traffic to HTTPS

      # The rest of the stuff goes 👇
   }
   ```

3. Save the file and exit the editor.

4. Edit the HTTPS and any other non-HTTP file (e.g., cockpit.conf) to include the following server properties.

   ```nginx
   server {
      listen 443 ssl http2; # Listen on port 443 (HTTPS) and use HTTP/2
      server_name example.com www.example.com; # Point the server to the domain name

      # SSL certificates and private key
      ssl_certificate /etc/nginx/ssl/full_chain.pem;
      ssl_certificate_key /etc/nginx/ssl/private_key.pem;
      ssl_trusted_certificate /etc/nginx/ssl/origin_certificate.pem;

      # More SSL settings that also get configured on Cloudflare to match the server
      ssl_protocols TLSv1.2 TLSv1.3; # Only enable the most secure protocols TSLv1.2 and TSLv1.3
      ssl_prefer_server_ciphers on; # Prefer the server's ciphers over the client's
      ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256'; # Ciphers to use
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always; # Enable HSTS (HTTP Strict Transport Security) -- only communicate with HTTPS

      # The rest of the stuff goes 👇
   }
   ```

   **`🔔 Important`** The `ssl_trusted_certificate` directive is used to specify the certificate that the server will trust. This is the Cloudflare Origin CA certificate. This is important because the server will only trust the certificate that Cloudflare uses to connect to the server.

5. Save the file(s) and exit the editor.

6. Make sure to have symbolic links to the sites-enabled directory for all the sites you want to enable:

   ```bash
   sudo ln -s /etc/nginx/sites-available/your-site-name.conf /etc/nginx/sites-enabled/your-site-name.conf
   ```

7. Test the Nginx configuration:

   ```bash
   sudo nginx -t
   ```

8. If the test is successful, reload Nginx:

   ```bash
   sudo systemctl reload nginx
   ```

Now there are some settings that need to be changed in Cloudflare to match the server settings.

## Configure Cloudflare to Match the Server Settings

1. Go to the Edge Certificates tab and make sure the following are set:

   1. Enable the Always Use HTTPS option.
   2. Enable HTTP Strict Transport Security (HSTS).
      1. Enable HSTS (Strict-Transport-Security).
      2. Max Age: 6 months.
      3. Apply HSTS to subdomains.
      4. Preload HSTS.
      5. Ignore no-sniff headers (if you want).
      6. Completing the above five steps will raise the grade of your site on SSL Labs from an A to an A+. High five!
   3. Set the minimum TLS version to 1.2.
   4. Enable Opportunistic Encryption.
   5. Enable TLS 1.3.
   6. Enable Universal SSL.

Go to [SSL Labs](https://www.ssllabs.com/ssltest/index.html) and check the grade of your site. It should be an A+ at this point. Just to make sure everything is working as expected, go to [https://example.com](https://example.com) and see if the site is loading correctly. If something is not working, restart the server, maybe go take a walk, and try again.

## One More Thing

Turn on Authenticated Origin Pulls so the server will only respond to requests from Cloudflare. Doing this will also make root access to the server more secure. You will no longer be able to access the server directly, only through Cloudflare's proxy connection.

1. Go to the SSL/TLS > Origin Server tab in Cloudflare and enable Authenticated Origin Pulls.

## Conclusion

The connection between the client and Cloudflare is secure, and the connection between Cloudflare and the server is also secure. The site should be loading correctly and should have a grade of A+ on SSL Labs. Have a good day!
