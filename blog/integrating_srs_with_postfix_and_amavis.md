## Or more specifically, integrating SRS with Postfix & Amavis, without breaking SPF checks!
# Current set-up
I've always run Amavis (with Clamd and Spamassassin) on Postfix as the standard set-up article suggests - as a content-filter. Unfortunately, content-filters are utilised after queueing, and when I recently decided to integrate [postsrsd](https://github.com/roehling/postsrsd) this caused me some issues, specifically with SPF. I have SPF records on my domain, and have done for some time - these have been working fine. I've also been checking the SPF records of received mail on my MXes for some time - historically I'd always done this through the postfix-policyd-spf-python package, integrated as a check policy in Postfix. The issue with this is that it can only be used as an outright pass or fail - really I wanted to integrate it into the existing spam checking that goes on in Amavis.

# Easy fix (or so it seemed...)
I quickly worked out how to do this (Spamassassin has config that will do it by default in Debian/Ubuntu, assuming you have the right packages installed) - so I installed libmail-spf-perl and restarted Spamassasin on one of my MXes. Sure enough, it started added SPF information to the spam headers in mail it scanned - so I went ahead and disabled the policyd package.

# Some time later...

After a while, I went to poke at the logs and see what kind of an effect it was having on mail (specifically whether it was helping to stop spam) - and I noticed something worrying. The SPF checks for all mail I received were failing! After some investigation, I worked out why - the SRS rewrite was taking place before the mail was passed to Amavis, so every mail Amavis received was checking the SPF records for my domain, using the originating server for the inbound mail. At first I thought I might be able to overcome this by attaching the SRS rewrite config to the amavis re-injection service (the postfix service defined in /etc/postfix/master.cf that listens on 10025, and has no content-filter etc specified) - but that didn't work.

# The *real* fix
After taking a step back and thinking it through, I realised that the best way of solving this was to move where in the process of receiving mail Amavis runs - this led me to look at the amavisd-milter package.

My SRS config is now exactly as it was before;

    # SRS

    # rewrite the sender on outbound (forwarded) mail,
    # excluding the addresses in no_srs.cf
    sender_canonical_maps = hash:/etc/postfix/no_srs.cf,
        tcp:127.0.0.1:10001
    sender_canonical_classes = envelope_sender

    # rewrite the sender on inbound mail that was
    # forwarded and rewritten by us previously,
    # excluding the addresses in no_srs.cf
    recipient_canonical_maps = hash:/etc/postfix/no_srs.cf,
        tcp:127.0.0.1:10002
    recipient_canonical_classes = envelope_recipient

but my Amavis config has changed slightly - the amavisd-milter package utilises the amavisd-new service, so don't remove that or prevent it from starting. Simply install the milter package, and make a couple of tweaks to /etc/default/amavisd-milter. The first change is optional (I did it to prevent issues with unix sockets in chroots, and ownership of them);

    # Make amavisd-milter listen on 10026 on localhost, 
    # instead of the unix socket.
    MILTERSOCKET=inet:10026@127.0.0.1

The second change is more important - the milter package needs to only accept a number of maximum connections equal to the max_servers variable in the amavis configuration - in my case this is 2, so I added the following;

    # Make amavisd-milter only accept two simultaneous connections,
    # as that's what's in amavis' config.
    EXTRAPARAMS="-m 2"

Then you need to tell postfix to use this new service - comment out the content-filter definition in /etc/postfix/main.cf;

    #content_filter = smtp-amavis:[127.0.0.1]:10024

and add a service to your milter definitions (I already had one for DKIM signing);

    # DKIM signing & Amavisd-milter
    milter_macro_daemon_name=ORIGINATING
    smtpd_milters=inet:127.0.0.1:10005,inet:127.0.0.1:10026
    non_smtpd_milters=inet:127.0.0.1:10005,inet:127.0.0.1:10026

If you don't have any milters, then just make that;

    # Amavisd-milter
    milter_macro_daemon_name=ORIGINATING
    smtpd_milters=inet:127.0.0.1:10026
    non_smtpd_milters=inet:127.0.0.1:10026

Optionally, you can set the default action to take in the event that your milters aren't functional - bear in mind that this will allow mail to pass through without being scanned by amavis if you misconfigure it.

    # if the milters don't work, don't reject the mail
    milter_default_action = accept

Then restart amavisd-new (I don't know why, but the milter restart  on it's own didn't seem to make it listen on the right socket for me), amavisd-milter, and postfix;

    sudo service amavis-new restart
    sudo service amavis-milter restart
    sudo service postfix restart

et voila - I now have amavis performing content scanning before the mail is accepted, and then SRS only taking place once it's been processed. This has an added benefit of allowing amavis to perform rejection as part of the SMTP transaction, instead of having to bounce the mail later on. To make that happen, I just updated /etc/amavis/conf.d/50-user;

    $final_spam_destiny = D_REJECT; # reject any message that seems too spammy

Because of that last benefit, I've also installed the milter package on other MXes, even though they don't forward mail (and so don't have the SRS package installed) - it's much better to be able to reject the mail when you first get it, rather than bounce it later. A couple of downsides to this method include longer SMTP sessions when receiving mail (so you might not want to run this on a slow mail server), and a complete failure if the amavis service is stopped/broken - normally, when amavis is utilised as a content-filter, postfix would still queue the mail, but then it would get no further until the service was started/functioning. Instead, we'll now fail to accept the mail - in my opinion this is a price worth paying for the ability to SRS and reject at the time of queueing.
