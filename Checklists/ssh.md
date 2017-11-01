## Securing SSH Services

Source: https://isc.sans.edu/forums/diary/Securing+SSH+Services+Go+Blue+Team/22992/

As the world of the attacker evolves and new attacks are developed (Red Team), people in the world of defense sees a matching evolution in recommendations for securing various platforms and services (Blue Team).  It struck me as odd that we don’t see a lot of “high profile” changes in advice for SSH, so I did some digging.

The go-to document that many of us use for SSH security is NISTIR 7966 (http://nvlpubs.nist.gov/nistpubs/ir/2015/NIST.IR.7966.pdf).  That has lots of great recommendations, and is dated October 2015, so it’s 2 years old as of this post.

There are some great OPENSSH recommendations published here at mozilla.org: https://wiki.mozilla.org/Security/Guidelines/OpenSSH.  The latest commit on that project was Oct 19, so just 11 days ago – that’s about as current as it gets.

However, lots of the devices we work with don’t give us access to the OpenSSH config, so there’s a lot of translation to match up guidance to the command interface of this or that device.  Bettercrypto.org hosts a paper that does just this.  https://bettercrypto.org/static/applied-crypto-hardening.pdf

For instance, let’s look at Cisco IOS routers – mostly because the configuration is so simple, and I see so many of them that have silly configuration “misses” on the security side.  Most guidance for Cisco IOS routers suggests that you force SSHv2, add an access-list to limit access, and then stop there.  That config sequence would look like:

```
crypto key gen rsa gen mod 2048        ! generate the SSH key

login on-failure log                   ! log failed login attempts
login on-success log                   ! log successful login attempts

 

ip ssh version 2                       ! force SSHv2 (SSHv1.x is easily decrypted)

ip access-list standard ACL-MGT        ! Create an ACL to limit access

  permit ip x.x.x.0 255.255.255.0      ! permit access from management VLAN

  permit ip host x.x.x.y               ! or permit access from specific management hosts

deny ip any log                        ! log all other accesses  (you can log the successful ones too of course)

 

line vty 0 15                          ! note that some newer platforms have more than 15 VTY lines !!

transport input ssh                    ! restrict to SSH only (no telnet)

access-class ACL-MGT                   !  apply the ACL
```

However, combining advice from the various documents above, naming the key, increasing the key size and setting a client minimum all make good sense – this changes our config commads to:

```
crypto key generate rsa modulus 4096 label SSH-KEYS     ! name the keys.  This makes it easier to rotate keys without a "gap" in protection

ip ssh rsa keypair-name SSH-KEYS

ip ssh version 2

ip ssh dh min size 2048                                                                              ! set a minimum standard for a client to connect with (helps prevent downgrade attacks)

ip access-list standard ACL-MGT

 permit ip x.x.x.0 255.255.255.0             ! permit access from management VLAN

 permit ip host x.x.x.y                      ! or permit access from specific management hosts

deny ip any log                              ! log all other accesses  (you can log the successful ones too of course)

 

line vty 0 15                                ! note that some newer platforms have more than 15 VTY lines !!

transport input ssh                          ! restrict to SSH only (no telnet)

access-class ACL-MGT                                                           ! apply the ACL

A Cisco ASA gives us a bit more control, where we can set the key exchange group.

crypto key generate rsa modulus 4096

ssh version 2

ssh key-exchange group dh-group14-sha1

 

ssh x.x.x.y m.m.m.m inside            ! limits SSH access to specific hosts or subnets.
```

(I’d recommend NOT to enable ssh to any public interfaces – use a VPN connection, preferably with 2FA, then connect to the inside interface).

Rather than disable telnet on an ASA, you simply don’t grant access to it – be sure that there are no lines similar to:
```
telnet x.x.x.y m.m.m.m <interface name>
```
in your configuration.

Note that both the Cisco IOS platform and Cisco ASA support key based authentication in addition to standard userid/password authentication.  The advantage to this is that you can’t easily brute-force a key-based authentication interface – so that’s a very (very) good thing.  The disadvantage is that if an admin workstation is then compromised, the attacker is in the “all your keys are belong to us” situation – all they need is the keys, and the keys are usually helpfully named with the IP address or hostnames of the targets.  Or if you happen to be a vendor who’s put keys into a firmware image, that would mean all instances of your product everywhere are easy to pop.

This covers off some basic SSH configuration, but what about auditing this config?  (Stay tuned)

===============
Rob VandenBrink
Compugen
	Rob VandenBrink

449 Posts
ISC Handler
Reply Subscribe 	Nov 1st 2017
10 hours ago
Also, off-topic but related:

And to not forget, TLS guidance is here:
https://tools.ietf.org/html/rfc7525#page-5

this was mostly due to absolute retirement of SSL as well as deprecation of various crypto elements.
