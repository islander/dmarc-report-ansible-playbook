# dmarc-report-ansible-playbook

DMARC parser and report viewer ansible playbook.

Powered by [parser](https://github.com/techsneeze/dmarcts-report-parser.git) and [viewer](https://github.com/techsneeze/dmarcts-report-viewer.git).

## Mail server DMARC config

Below is example for Exim4, if you use another server, examine it's documentation.

First, generate DKIM keys:

```
# mkdir /etc/exim/dkim/
# cd /etc/exim/dkim/
# openssl genrsa -out example.com.key 2048
# openssl rsa -pubout -in example.com.key -out example.com.pub
# chown -R exim:exim /etc/exim/dkim
# chmod -R 0640 /etc/exim/dkim
```

Configure Exim:

```
# vim /etc/exim/exim.conf

DKIM_DOMAIN = ${lc:${domain:$h_from:}}
DKIM_FILE   = /etc/exim/dkim/${lc:${domain:$h_from:}}.key
DKIM_PRIVATE_KEY = ${if exists{DKIM_FILE}{DKIM_FILE}{0}}

remote_smtp:
    driver = smtp
    dkim_domain = DKIM_DOMAIN
    dkim_private_key = DKIM_PRIVATE_KEY
    dkim_selector = mail
    dkim_canon = relaxed
    dkim_strict = yes
```

Restart service:

```
# systemctl restart exim
```

## DNS configuration

Add SPF, DMIM and DMARC DNS records to your name server according to your setup.

DKIM `p=` value should be content of your `example.com.pub` without whitespaces and header and footer:

```
# grep -v PUBLIC /etc/exim/dkim/example.com.pub | tr -d '\n'
MIIEpAIBAAKCAQEAqNBflT3e8lzjNa1TzZI/y631dOXavzA+v4siHQLkyCiyMkSB..."
```

Test records with `dig` utility:

```
# dig TXT example.com +noall +answer

; <<>> DiG 9.11.3-1ubuntu1.7-Ubuntu <<>> TXT example.com +noall +answer
;; global options: +cmd
example.com.                3600    IN      TXT     "v=spf1 a mx ~all"

# dig TXT mail._domainkey.example.com +noall +answer

; <<>> DiG 9.11.3-1ubuntu1.7-Ubuntu <<>> TXT mail._domainkey.example.com +noall +answer
;; global options: +cmd
mail._domainkey.example.com. 3589	IN	TXT	"v=DKIM1; g=*; k=rsa; p=MIIEpAIBAAKCAQEAqNBflT3e8lzjNa1TzZI/y631dOXavzA+v4siHQLkyCiyMkSB..."

# dig TXT _dmarc.example.com  +noall +answer

; <<>> DiG 9.11.3-1ubuntu1.7-Ubuntu <<>> TXT _dmarc.example.com +noall +answer
;; global options: +cmd
_dmarc.example.com.         3593    IN      TXT     "v=DMARC1; p=none; rua=mailto:postmaster@example.com"
```

If everything correct, you will receive daily DMARC reports on `postmaster@example.com`. Parse them and analyze using tools in this playbook.
