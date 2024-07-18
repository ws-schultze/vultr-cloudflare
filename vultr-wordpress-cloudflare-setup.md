# Setting Up a Vultr Server with WordPress and Cloudflare

**Note:** for this entire walkthrough, let’s assume that the website we are making is called
example.com

## Deploy the Server

1. Deploy an Ubuntu server with WordPress pre-installed.

2. Wait for DNS propagation; check it at [whatsmydns.net](http://whatsmydns.net).

3. Go to the server’s IP address (e.g. 111.222.333.44) and see if the default WordPress site is visible.

## Manage DNS Records Within Your Domain Name Registrar Account

1. Add the following two A records:

   1. Host: @, Answer: IP of the server you just deployed (e.g. 111.222.333.44), TTL: Auto

   2. Host: www, Answer: IP of the server you just deployed, TTL: Auto

2. Check that both [example.com](http://example.com) and [www.example.com](http://www.example.com) both go to the same place.

## Add the Site to Cloudflare

1. Inside your Cloudflare account, navigate to the websites tab and click "+ Add a site”.

2. Select the free plan for now

3. Follow the instructions:

   1. Make sure the DNS records look as they do in your registrar account

   2. Change the name servers in your registrar account to the Cloudflare name servers

   3. Wait for the name server changes to take effect

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

Your SSL/TLS encryption mode is set to "Flexible" by default in Cloudflare, which means that the connection between the client and Cloudflare is secure, but the connection between Cloudflare and the server is not. Let's set the encryption mode to Full(strict) to ensure that the connection between Cloudflare and the server is secure. Do this by:

1. Go to the SSL/TLS tab in Cloudflare and set the encryption mode to Full(strict).

2. Go to the Edge Certificates tab and enable the Always Use HTTPS option.

3. Go to the Rules tab and add a rule to always use HTTPS.

   1. Click "+ Create rule".

   2. Rule name: Always Use HTTPS

   3. In the "If.... When incoming requests match..." field, select "All incoming requests".

   4. In the "Then the settings are..." field, select "+ Add" button next to "Automatic HTTPS Rewrites" then toggle it on.

   5. Go to the bottom of the page an click "Deploy".

Now to to [http://example.com](http://example.com) and see if the connection is redirected to [https://example.com](https://example.com).

Notice that the images (if you have any) are probably not loading. This is likely because Wordpress is still using HTTP. To fix this:

1. Go to the WordPress admin panel and navigate to Settings > General.

2. Change the WordPress Address (URL) and Site Address (URL) to [https://example.com](https://example.com).

3. Save the changes.

Now go to [https://example.com](https://example.com) and see if the images are loading.
It could be that the images are still not loading because the server is not yet configured to have a secure connection with Cloudflare.

## Install Cloudflare Origin CA Certificate, Root Certificate and Private Key onto the Server

Because this site will only be accessed through cloudflare, we can Use Cloudflare’s Origin CA to generate certificates for your server. To do this:

1. Go to the SSL/TLS tab in Cloudflare and click on Origin Server.
2. Click on Create Certificate.
3. Fill in the details and click Next.
4. Download the certificate and key. These will be copied onto the server in a moment.
5. SSH into the server.

6. Create a directory for the certificate and key:

   ```bash
     sudo mkdir /etc/nginx/ssl
   ```

7. Copy the certificate and key that you got from Cloudflare a moment ago, into the directory:

   ```bash
     sudo cp ~/path/to/origin_certificate.pem /etc/ssl/origin_certificate.pem
     sudo cp ~/path/to/private_key.pem /etc/ssl/private_key.pem
   ```

8. Get the Cloudflare Origin CA root certificate and put it in the same directory as the other certificates:

   ```bash
   sudo wget https://developers.cloudflare.com/ssl/static/origin_ca_rsa_root.pem -O /etc/nginx/ssl/cloudflare_origin_ca.pem
   ```

9. Make the full chain certificate:

   ```bash
   sudo cat /etc/nginx/ssl/cloudflare_origin_ca.pem /etc/nginx/ssl/origin_certificate.pem > /etc/nginx/ssl/full_chain.pem
   ```

Now there should be four files in the /etc/nginx/ssl directory:

```text
   cloudflare_origin_ca.pem
   full_chain.pem
   origin_certificate.pem
   private_key.pem
```

Now it's time to configure Nginx to use these certificates.

## Configure Nginx to Use the Certificates

1. Open the Nginx configuration file:

   ```bash
   sudo nano /etc/nginx/sites-available/your-site-name.conf
   ```

   **`🔔 Tip`** Using VSCode to SSH into the server makes things easier if you are not very comfortable in the the terminal with vi or nano.

Preinstalled Wordpress will put two .conf files for your site in the `/etc/nginx/sites-available` directory, one is for HTTP and the other is for HTTPS. You will need to edit them both or combine them into one file, and edit that, your call. The directory could look something like this:

```text
   cockpit.conf
   default.conf
   your-site-name.conf
   your-site-name-ssl.conf
```

1. Edit the HTTP file to include the following server properties:

   ```nginx
   server {
      listen 80;
      server_name example.com www.example.com;
      return 301 https://$host$request_uri;

      # The rest of the stuff goes 👇
   }
   ```

2. Save the file and exit the editor.

3. Edit the HTTPS and and other non HTTP file (e.g. cockpit.conf) to include the following server properties:

   ```nginx
   server {
      listen 443 ssl http2; # Listen on port 443 (HTTPS) and use HTTP/2
      server_name example.com www.example.com; # Point the server to the domain name

      # SSL certificates and private key
      ssl_certificate /etc/nginx/ssl/full_chain.pem;
      ssl_certificate_key /etc/nginx/ssl/private_key.pem;
      ssl_trusted_certificate /etc/nginx/ssl/cloudflare_origin_ca.pem;

      # More SSL settings that also get configured on Cloudflare to match the server
      ssl_protocols TLSv1.2 TLSv1.3; # Only enable the most secure protocols TSLv1.2 and TSLv1.3
      ssl_prefer_server_ciphers on; # Prefer the server's ciphers over the client's
      ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256'; # Ciphers to use
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always; # Enable HSTS (HTTP Strict Transport Security) -- only communicate with HTTPS

      # The rest of the stuff goes 👇
   }
   ```

   **`🔔 Note`** The `ssl_trusted_certificate` directive is used to specify the certificate that the server will trust. This is the Cloudflare Origin CA certificate. This is important because the server will only trust the certificate that Cloudflare uses to connect to the server.

4. Save the file(s) and exit the editor.

5. Make sure to have symbolic links to the sites-enabled directory, for all the sites you want to enable:

   ```bash
   sudo ln -s /etc/nginx/sites-available/your-site-name.conf /etc/nginx/sites-enabled/your-site-name.conf
   ```

6. Test the Nginx configuration:

   ```bash
   sudo nginx -t
   ```

7. If the test is successful, reload Nginx:

   ```bash
   sudo systemctl reload nginx
   ```

Now there are some setting that need to be changed in Cloudflare to match the server settings.

## Configure Cloudflare to Match the Server Settings

1. Go to the SSL/TLS tab in Cloudflare and set the encryption mode to Full(strict).
2. Go to the Edge Certificates tab and make sure the following are set:

   1. Enable the Always Use HTTPS option.
   2. Enable HTTP Strict Transport Security (HSTS).
   3. Set the minimum TLS version to 1.2.
   4. Enable Opportunistic Encryption.
   5. Enable TLS 1.3.
   6. Enable Universal SSL.

3. Go to the Rules tab and add a rule to always use HTTPS.
   1. Click "+ Create rule".
   2. Rule name: Always Use HTTPS
   3. In the "If.... When incoming requests match..." field, select "All incoming requests".
   4. In the "Then the settings are..." field, select "+ Add" button next to "Automatic HTTPS Rewrites" then toggle it on.
   5. Go to the bottom of the page an click "Deploy".

Go to [SSL Labs](https://www.ssllabs.com/ssltest/index.html) and check the grade of your site. It should be an A+ at this point.

Just to make sure everything is working as expected, go to [https://example.com](https://example.com) and see if the site is loading correctly.

If something is not working, restart the server and try again.

## Conclusion

At this point, you should have a WordPress site running on a Vultr server with Cloudflare acting as a proxy. The connection between the client and Cloudflare is secure, and the connection between Cloudflare and the server is also secure. The site should be loading correctly and should have a grade of A+ on SSL Labs.
