---
layout: post
title: "Publishing SPF Records"
date: 2014-02-23 23:07:01 +0100
comments: true
categories: 
---

The [Sender Policy Framework][spf] (SPF) is today used by many mail servers or
SPAM filters to determine if an email sent as originating from
<code>user&#64;example.net</code> really originates from
<code>example.net</code> or if it was highjacked by a spammer. An SPF-enabled
receiving mail server checks both, the fully qualified domain name (FQDN) as
specified in the HELO or EHLO announce message sent by the mail client, and the
domain of the envelop email address (MAIL FROM) against the SPF record(s)
stored in DNS for that domain.

## Example

As an example, I own two domains, <code>ntecs.de</code> and
<code>michaelonroad.de</code>. I run a SMTP server on the domain
<code>mail.ntecs.de</code>. As such, I need two SPF DNS records for domain <code>ntecs.de</code>:

```
# DNS zone for domain ntecs.de
@ IN TXT "v=spf1 mx -all"
mail IN TXT "v=spf1 a -all"
```

The first line tells the SPF enabled mail server that all email in the form of
<code>user&#64;ntecs.de</code> MUST exclusively be sent by one of the servers
listed as a Mail Exchange (MX) record of the same domain. The MX record
resolves to an IP address which is then checked against the IP address of the
connection. The second line is used to check against the EHLO (or HELO)
announcement. For example my SMTP server announces itself with a <code>EHLO
mail.ntecs.de</code>. Only the IP address of the <code>mail.ntecs.de</code>
server is accepted here in this case (or more specifically the IP address it's
<code>A</code> record resolves to).

For my second domain <code>michaelonroad.de</code> I would only need one SPF record:

```
# DNS zone for domain michaelonroad.de
@ IN TXT "v=spf1 mx -all"
```

This is because I use <code>mail.ntecs.de</code> even for sending mail from
<code>user&#64;michaelonroad.de</code>. Also note that in my case I only run
one server for both incoming and outgoing mail (for all domains).

[SPF]: http://www.openspf.org/
