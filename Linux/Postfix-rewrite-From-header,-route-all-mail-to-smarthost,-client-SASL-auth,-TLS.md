# Postfix: rewrite From header, route all mail to smarthost, client SASL auth, TLS

You have a system that you want to send e-mail from. That system is at a residential IP address, which is a problem for sending mail.
The big e-mail hosting providers have strict mail reception filtering to combat spam, phishing and impersonation. This means that for all incoming mail, they implement rDNS, SPF and DKIM checking.
Of course, your residential address doesn't have a rDNS record (or it's controlled by your ISP, not you, so you can't change it) and you can't add MX, SPF and DKIM for your dynamic IP (nor is it practical).

To solve this you relay mail through an existing e-mail server (one that you host yourself or one of a hosting provider). To relay, you need a valid e-mail account on the server. Ideally, you create one e-mail account per sending host. That way each host has its own password, so in case it gets hacked and starts sending spam, you know which it is and disable the account. Also you know which host is sending just from the From header, which makes filtering on the receive side easy. As a bonus, anyone replying to such an e-mail will reply to a valid mailbox, where mail will be delivered, for you to look at later (or forward it to your primary mail address).

There's two requirements for relaying through such mail hosting providers.
First, you have to log in with your username and password. In postfix, that's easy with SASL.
Second, the mail (both header and envelope) From address (sender) must be identical to your e-mail address. (if you could pick any arbitrary From address, you could impersonate anyone - some lesser mail providers actually allow this, but it's not good, and your mails will get filtered at the receive side for not matching the DKIM record or by other antispam measures).

## Configuration
All the files and commands below should be created or ran as root. Replace any example values as necessary (`myhost@example.com`, `smtp.example.com`, `p4ssw0rd`)

Of course, postfix should already be installed.

If you're on a Debian or Ubuntu OS, you can start by setting some basic settings by running `dpkg-reconfigure postfix`.
When asked, select "Satellite system" (for only local mail reception) or "Internet site with smarthost" (if you want to also receive and relay mails from other hosts via the network, make sure you're firewalled from the internet though).
For the smarthost, enter the hostname of the SMTP server of your mail provider, usually you want to also use port 587.

### SASL login
Install libsasl2-modules (Debian, Ubuntu).

Add to main.cf:
```
# client SASL
smtp_sasl_auth_enable = yes
smtp_sasl_tls_verified_security_options = noanonymous
smtp_sasl_mechanism_filter = plain
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
```

Create the sasl_passwd file:
```
touch /etc/postfix/sasl_passwd
chmod 600 /etc/postfix/sasl_passwd
cat > /etc/postfix/sasl_passwd <<-'EOF'
smtp.example.com:587 myhost@example.com:p4ssw0rd
EOF
postmap /etc/postfix/sasl_passwd
```

### Rewriting From
To achieve the 2nd requirement, postfix must replace all header From addresses with the login e-mail address.
This is possible by using sender_canonical_maps and smtp_header_checks.

Add to main.cf:
```
sender_canonical_maps = regexp:/etc/postfix/sender_canonical_map_rewriteheaderfrom
smtp_header_checks = regexp:/etc/postfix/header_checks_rewriteheaderfrom
```

Create the files:
```
cat > /etc/postfix/sender_canonical_map_rewriteheaderfrom <<-'EOF'
/.*/ myhost@example.com
EOF

cat > /etc/postfix/header_checks_rewriteheaderfrom <<-'EOF'
/From:.*/ REPLACE From: myhost@example.com
EOF
```

### Redirecting all mails
If you want to redirect all mails to any recipient (local or remote) to one or more remote addresses, use a regexp alias_maps:

```
# comment out alias_maps, replace it with new one
sed -i -E 's,(^\s*alias_maps\s*=)(.*),#\1\2\n\1 regexp:/etc/postfix/alias_maps_regexp,' /etc/postfix/main.cf
# comment out alias_database
sed -i -E 's,(^\s*alias_database\s*=.*),#\1,' /etc/postfix/main.cf

cat > /etc/postfix/alias_maps_regexp <<-'EOF'
/.*/ hostmaster@example.com
EOF
```

### TLS
Establishing a TLS session is easy if the relay server has a certificate trusted by the local PKI.
The benefits of using TLS should be self-evident. If you don't configure it, mails will be sent over a plaintext connection which could be intercepted.

Add to main.cf:
```
# client TLS
smtp_tls_security_level = verify
smtp_tls_loglevel = 1
#uncomment if your system only has the ca-bundle file and not individual symlinks or certs in /etc/ssl/certs (also comment smtp_tls_CApath)
#smtp_tls_CAfile = /etc/ssl/certs/ca-bundle.trust.crt
smtp_tls_CApath = /etc/ssl/certs/
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
```

If you also want this server to relay mail in the local network, listening on SMTP port 25, also enable TLS in smtpd:
```
# server TLS
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_use_tls=yes
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
```
Different OS's might have different paths to the self-signed certificate. Of course, if you have a trusted certificate for this host, configure postfix to use it!

### Complete main.cf
For reference, this is the complete main.cf. It's a combination of the one generated by dpkg-reconfigure (for a satellite system) and the settings from this guide.

```
myhostname=myhost.mydomain

smtpd_banner = $myhostname ESMTP $mail_name (Debian/GNU)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

#alias_maps = hash:/etc/aliases
#alias_database = hash:/etc/aliases
mydestination = myhost.mydomain, myhost.mydomain, localhost.mydomain, localhost
relayhost = smtp.example.com:587
mynetworks = 127.0.0.0/8
inet_interfaces = loopback-only
recipient_delimiter = +

compatibility_level = 2

myorigin = /etc/mailname
mailbox_size_limit = 51200000
inet_protocols = all

alias_maps = regexp:/etc/postfix/alias_maps_regexp
sender_canonical_maps = regexp:/etc/postfix/sender_canonical_map_rewriteheaderfrom
smtp_header_checks = regexp:/etc/postfix/header_checks_rewriteheaderfrom

# client SASL
smtp_sasl_auth_enable = yes
smtp_sasl_tls_verified_security_options = noanonymous
smtp_sasl_mechanism_filter = plain
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd

# client TLS
smtp_tls_security_level = verify
smtp_tls_loglevel = 1
smtp_tls_CApath = /etc/ssl/certs/

# server TLS
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_use_tls=yes
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
```
