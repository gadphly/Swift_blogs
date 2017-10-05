# Goal
- I have a test running on localhost `127.0.0.1`
- A self-signed cert is not enough. You need to a CA signed cert to get your tests on localhost to pass.

## Problems
The problem here is that a CA-signed cert can't be ordered for localhost (https://github.com/letsencrypt/boulder/issues/137#issuecomment-112236543)

# Solution (not secure!!!)

## Overview
1 - a real domain name should point to localhost
2 - create a cert for the domain name.

## Step by Step

1 - buy a cheap domain name. I bought one at GoDaddy (`gelareh.com`)
2 - Update DNS of your domain to point to localhost:
  - go to DNS management of your Domain manager
  - add an A record: _host_ can be a subdomain, say `ssl` which would really mean `ssl.gelareh.com` and _Points to_ 
  should be `127.0.0.1`.
  The tool might barf at you including localhost so you might have to play around with adding and editting your record to 
  get it to work. In my case, I added an A record with `@` for hostname and localhost and then I editted the record from `@` to the subdomain.

3 - Go to https://zerossl.com/free-ssl/#crt which uses Let's Encrypt under the covers.
 - _Details_ page: Add sub domain: `ssl.gelareh.com`. Check DNS verification and the TOS and SA. Press Next. Don't include www-prefix.
 - Generating CSR. Copy it. Press Next.
 - Generating account key. This is the private key for the Let's Encrypt account and will allow you to later re-generate certs whent they expire. Copy it.
 - A pop up should say OK now the key is registered.
 - _Verification_ page: go back to DNS management page of your Domain manager. Create a TXT record for `_acme-challenge.ssl` and the value they give. 
 - You should be able to test this on your terminal by: `dig _acme-challenge.ssl.gelareh.com in txt`
 - Press Next and this tool will then verify that that DNS record exists.
 - _Certificate_ page: Now you get the cert and key!

4 - Verify the certs that you have:
 - the cert you get is both the site's cert and the intermediate Let'sEncrypt cert.
 - divide up the file into two files: cert and intermediate
 - `openssl verify -CAfile /usr/local/etc/openssl/cert.pem -untrusted Intermediate.pem UserCert.pem` which uses the system's CA file to verify Let's Encrypt's cert's root singature.
