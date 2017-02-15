# HowTo Configure "Send-Only" Email via an SMTP 3rd Party Relay on a Fedora Linux 25 -based System using Postfix

I want to be able to send automated alerts and whatnot so that my phone buzzes when something bad happens on a remote
system that I manage. There are a number of ways you can do this, but I found the easiest is to simply pump automated email
through a 3rd party email vendor to myself, or a group of people.

> Note, these instructions have been tested on Fedora Linux 25, but they should be easily adaptable to any unix/linux. Check
out the reference at the end for some excellent guidance on that topic.

## Prep: Burner Email and Google Group

_Note, these are all made-up examples, of course._

* A burner email account. Example, nfdasd@yandex.com
* A google group to send notifications. Example, my-awesome-notifications@googlegroups.com

Let's talk about each of those...

### Burner email account. Example, `nfdasd@yandex.com`

Using a single purpose email account is important. If for whatever reason that account is compromised, your personal email
is put at risk. For example, if my personal google account were compromised, I would be in a world of hurt. We don't want
that. Sure sing app-passwords reduces that possibility, and sure we compromise ourselves all the time by linking
applications to our account, but... don't add one more when you don't have.

I am using yandex.com as an example, but you can use gmail.com, yahoo.com, whomever. I use something like Lastpass'es password generator to cycle through random usernames until I find one I like. Or just pound randomly on the keyboard. That works too. :)

* Create a <https://yandex.com> account, for example.

Caveat: They, like many providers, now require an associated cell phone number. They let you create an account temporarily
without one, but eventually they force you to cough up your cell phone number. I don't know if we can
avoid this anymore unless we set up our own domain for this purpose. That may be a separate HowTo.

* Set up an app password.

In order to use a scripted interface with most 3rd party accounts, they will provide you some means to access the account
with a password that is not your login password. The password allows certain activities on the account but not "admin"
activities. With Yandex, you do this in `Settings > Security > App password`. Create a label, copy the generated password.
Save that for later.

### Google Group to send notifications. Example, `my-awesome-notifications@googlegroups.com`

Instead of sending alert emails directly to some 1 account, send them to a google group account. It will serve like a
message bus of events and historical record. Plus multiple accounts can be attached to it if you want to have other people
watch what is going on with a particular system. Additionally, if you want to be all passive about it, you can set it up
that you only receive daily digests, no email at all (web view only), etc.

* Browse to <https://groups.google.com>
* Create a group. Our example is, `my-awesome-notifications`
* Set permissions so that it is invite only
* Set a description
* Add `nfdasd@yandex.com` and interested admin emails to the group.
* Send a test email to <mailto:my-awesome-notifications@googlegroups.com> from one of those group members.
* Everyone should receive that email.

## Configure you Fedora Linux server to send email

Summary of next steps...

* Install postfix and mailx
* Configure postfix
* Send test email

### Install postfix and mailx

Easy peasy...

```
sudo dnf install -y postfix mailx
```

### Get your email provider's SMTP server information

Search for your email provider's smtp domain and port and any special requirements to use it. For yandex.com, for example,
I found out that...

* `smtp.yandex.com` is the server -- even though they have a lot of various web targets for their service.
* Ports 587 and 465 are supported. And a lot of the documentation mentions 465. Don't use that. Use 587.
* App password needs to be set up to use it. You can't just use your username and password. You need to set up an "app
  password" as was described earlier.


### Configure postfix

Configuring postfix is pretty easy, though not entirely straight forward.

* Edit postfix configuration file: `sudo nano /etc/postfix/main.cf`
* Change these settings, or add if they are missing...

```
relayhost = [smtp.yandex.com]:587
#relayhost = [smtp.yandex.com]:465
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options =
smtp_tls_CAfile = /etc/ssl/certs/ca-bundle.crt
# These two additional settings only used if using port 465
#smtp_tls_wrappermode = yes
#smtp_tls_security_level = encrypt
```

Other options to examine suggested by other howtos...

```
# Note: user+alias@domain subaddressing is not supported by all email providers, yandex included
recipient_delimiter = +
mailbox_size_limit = 0
```

* Edit SMTP username and password in postfix sasl file: `sudo nano /etc/postfix/sasl_passwd`    
  For example for, yandex.com it will look like this. For gmail and such, it will be similar...

```
[smtp.yandex.com]:587 nfdasd@yandex.com:kjsadkjbfsfasdfqwfq
```

That file has your password in it. Lock it down: `sudo chmod 600 /etc/postfix/sasl_passwd`

* "Compile" that password file: `sudo postmap /etc/postfix/sasl_passwd`

It should produce a file called `/etc/postfix/sasl_passwd.db`

_Note: If you see permission errors, check ownership and permissions in that directory: `ls -l /etc/postfix`_

## Crank up Postfix

If you haven't enabled it upon reboot, do it now...

```
sudo systemctl enable postfix
```

Start it up...

```
sudo systemctl start postfix # if not previously started
#sudo systemctl restart postfix # if already started
#sudo systemctl stop postfix # if you need to stop it
#sudo systemctl status postfix # is it already started?
```

Monitor it...

```
sudo journalctl -u postfix -f -n25
```

## Send a test email...

Send one to someone directly (this is sending an email to me, please use your own email address)...

```
echo "This is the body of the email. Test. Test. Test." | mail -s "Direct email test 01" -r nfdasd@yandex.com t0dd@protonmail.com 
```

If that works fine, send one to your google group...

```
echo "This is the body of the email. Test. Test. Test." | mail -s "Group email test 01" -r nfdasd@yandex.com my-awesome-notifications@googlegroups.com
```

If you are monitoring the system log with that journalctl command, you should see something like...

```
Feb 15 17:50:59 mn0 postfix/pickup[20874]: D5100DC63A: uid=1000 from=<nfdasd@yandex.com>
Feb 15 17:50:59 mn0 postfix/cleanup[21887]: D5100DC63A: message-id=<58a49503.tOGKlZXDLpKqWJQt%nfdasd@yandex.com>
Feb 15 17:50:59 mn0 postfix/qmgr[19914]: D5100DC63A: from=<nfdasd@yandex.com>, size=469, nrcpt=1 (queue active)
Feb 15 17:51:04 mn0 postfix/smtp[21889]: D5100DC63A: to=<my-awesome-notifications@googlegroups.com>, relay=smtp.yandex.com[77.88.21.38]:587, delay=4.3, delays=0.03/0.04/2.5/1.7, dsn=2.0.0, status=sent (250 2.0.0 Ok: queued on smtp3j.mail.yandex.net as 1487181064-lyDmMVZ9eP-p2c0s157)
Feb 15 17:51:04 mn0 postfix/qmgr[19914]: D5100DC63A: removed
```

## All done!

Now you have a scripting pattern for using with something like a monitoring system or cronjobed scripts, etc.

Good luck. Comments or feedback: <t0dd@protonmail.com>


## References

* [Configure Postfix to use Gmail as a Mail Relay](https://www.howtoforge.com/tutorial/configure-postfix-to-use-gmail-as-a-mail-relay/)
* [Fedora Linux Postfix Documentation](https://docs.fedoraproject.org/en-US/Fedora/25/html/System_Administrators_Guide/s1-email-mta.html) - Educational, if not overly helpful
