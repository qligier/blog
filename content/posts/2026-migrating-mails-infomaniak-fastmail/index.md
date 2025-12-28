---
title: "How I migrated my emails from Infomaniak to Fastmail"
date: 2025-12-28
draft: false
tags: ['Email']
description: "The blog post is describing how I migrated my emails from Infomaniak to Fastmail, alongside the calendar
and contacts."
---

The reason for migrating is that I've been a bit disappointed by Infomaniak's email service lately.
While parts of the service have improved, like the Android app that used to have really frequent updates, which would
log you out of your account on each update at the time.
But the webmail interface is still quite clunky, with some unnecessary nagging to set it as the default mailbox in
the browser (I haven't seen such nagging in Gmail, for example).

The spam filtering is not very good either, with quite a few spam emails making it to the inbox.
While I report spam and phishing emails, I haven't seen any improvement over time: simply changing the sender 
address for a similar email will make it pass the spam filter.
There are also some missing features that I would expect from a mature service, like a calendar containing contacts'
birthdays.

## Changing the DNS

Emails use DNS records to know where to deliver emails for a given domain.
I previously configured them to point to Infomaniak's mail servers for my domain, and need now to change them.
Fastmail provides a nice guide to set up the DNS records for your domain, and it gives you preconfigured records to
add to your DNS provider (in my case, Dynadot).
You can even delegate the DNS management to Fastmail, if you don't feel confident doing it yourself; I prefer to 
keep control of my DNS records.

![Fastmail instructions for DNS configuration](dns_instructions.png "Fastmail instructions for DNS configuration")

These DNS records are split into three categories:

- MX records: these records indicate the mail servers responsible for receiving emails for your domain.
  Two records are provided for redundancy, with different priorities; this is a nice improvement over Infomaniak, which
  only provided one mail server.
- DKIM records (stored as CNAME records): these records are used to sign your emails cryptographically, to prove that they
  were sent by you and not forged by a spammer.
  Fastmail provides two DKIM keys, which is nice for key rotation.
- SPF record (stored as a TXT record): this record indicates which mail servers are allowed to send emails for your domain.
  This helps prevent spammers from sending emails that appear to come from your domain.

DNS propagation can take some time, depending on how they are set up and how the other mail servers cache them.
I continued to receive emails on Infomaniak for a couple of hours after changing the MX records, which is expected 
and coherent with my DNS Time-To-Live (TTL) setting (2 hours).

Something that I didn't notice while changing the MX records is that the old record for Infomaniak had a priority of 
5, while the new Fastmail records have priorities of 10 and 20 (meaning they have a lower priority). This could have
increased the time needed for the change to be effective, because a mail server that cached the old record would still
choose it over the new ones.

## Migrating the emails

Once all new emails are being delivered to Fastmail, I can start migrating the old emails.
Fastmail provides a nice guide to migrate emails from other providers, with custom integrations for the most common 
providers (Gmail, Outlook/Office 365, iCloud, Yahoo, etc.), upload of MBOX files, or generic IMAP migration.
Although the import started slowly, it completed successfully after 10 minutes, having migrated 50 folders and 
19'402 messages (around 1 GiB of data in total).

Infomaniak doesn't provide any email export functionality, so I had to use the generic IMAP migration.
That surely is a point where they could improve their service: generating an MBOX or PST file export of your emails
would greatly facilitate backup and migration.

## Migrating the contacts and calendar

Although Fastmail can import contacts and calendars from other providers using CardDAV and CalDAV protocols, Infomaniak
provides an export functionality that produces standard VCard and VCal files.
Importing them in Fastmail is only a matter of drag-and-dropping; much easier than finding the Infomaniak 
configuration for these (their CalDAV integration uses a different user and password than the email account, for
some reason).

## Updating the email/calendar/contacts apps

On Windows, I simply replaced Thunderbird with the Fastmail application, which is a native wrapper around the webmail
with a proper offline support and good OS integration (unlike many Electron apps).

On Android, the Fastmail app replaced the Infomaniak Mail one (a fork of KMail), and DavX⁵ replaced Infomaniak kSync to
synchronize both contacts and calendar. The configuration of the latter is straightforward, as the setup wizard has a
dedicated flow for Fastmail accounts.

## Wrapping up

There was not much more to do to complete the migration, it was quite straightforward and fast (only a few hours to
complete it).
Everything worked perfectly well, without any hiccups on the way.

It's now a few weeks since I migrated, and I'm quite happy with Fastmail so far.
All interfaces (the webmail and the different applications) are much more enjoyable to me than the previous ones,
and the spam filtering seems to be more effective as I've received much less spam emails since the migration.
All in all, it was definitely worth the effort!
