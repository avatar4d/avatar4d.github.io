---
layout: post
title:  "Fashionably Secure SSH in Fedora Linux"
date:   2024-06-20 20:42:52 -0400
tags: fedora linux ssh HostKeyAlgorithms PubkeyAcceptedAlgorithms RequiredRSASize ssh_config crypto-policies
categories: sysadmin linux
---
# Introduction
Despite the inherent security risks, I've found that end-of-life (EOL) enterprise switches generally outperform consumer tech. While prosumer gear like the TPLink TL-SG108E offers features like VLAN support, it can be quirky. For instance, VLANs need to be set in two places in the UI in order to work, and firmware updates require a factory reset. Yes, you read that correctly. [The device requires a reset before you can install a new firmware!](https://community.tp-link.com/en/business/forum/topic/513008). Furthermore, consumer options for 10Gb fiber are limited.

Given the high cost of brand new enterprise gear, I turn to the secondary market for used equipment. Although this hardware is insecure due to lack of support, it is really  no worse than consumer tech, which typically receives only a few updates before becoming unsupported and insecure. Despite its own shortcomings, I feel that enterprise gear far surpasses consumer alternatives. To minimize the risk of operating them, I place them behind my OpenBSD firewalls and configure them to only accept incoming connections from specific internal addresses. 

<br>

# Addressing Disabled OpenSSH Options
The intent of this post is to describe how to enable connections to OpenSSH on EOL enterprise switches. These switches run older OpenSSH versions, which lack cryptography considered secure by current standards. That presents challenges when attempting to connect because the OpenSSH team periodically disables options as they lose relevance. This requires the user to explicitly enable those deprecated mechanisms when connecting to the switches. I found that [Fedora](https://fedoraproject.org/) imposes additional restrictions, requiring even more steps from the user.

<br>

# Configuring SSH Client for Older Systems
To connect to these older systems, I've historically used the following options, which are discussed in many other places online. I suggest reading the [ssh_config manual](https://man.openbsd.org/ssh_config) for more details on these and other available options.
```
KexAlgorithms
HostKeyAlgorithms
```
<br>

For convenience, I place these options in my ~/.ssh/config file to avoid typing them out each time. I manage that and other dotfiles with [chezmoi](https://www.chezmoi.io/) so everything works as expected no matter which system I am using. Needless to say, I was initially confused when it didn't work on Fedora. 

Here’s an example ssh_config:

```
Host  10.1.x.y
    KexAlgorithms +diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa
```

<br>

Without these options, attempts to connect result in errors:
```
~> ssh  10.1.x.y
Unable to negotiate with 10.1.x.y port 22: no matching key exchange method found. Their offer: diffie-hellman-group1-sha1
```

```
~> ssh 10.1.x.y -oKexAlgorithms=+diffie-hellman-group1-sha1
Unable to negotiate with 10.1.x.y port 22: no matching host key type found. Their offer: ssh-rsa
```

<br>
# Troubleshooting on Fedora
On Mac or [Pop!_OS](https://pop.system76.com/), the following command addresses the last error, but, as mentioned, Fedora requires additional steps:

```
~> ssh 10.1.x.y -oKexAlgorithms=+diffie-hellman-group1-sha1 -oHostKeyAlgorithms=+ssh-rsa
Unable to negotiate with 10.1.x.y port 22: no matching host key type found. Their offer: ssh-rsa
```

That’s weird. Despite explicitly enabling ssh-rsa, Fedora still blocks the connection due to its system-wide crypto policies, as detailed in ssh_config(5), which refers us to the crypto_policies(7) manpage.

```
The proposed HostKeyAlgorithms during KEX are limited to the set of algorithms  that  is  defined  in  PubkeyAcceptedAlgorithms  and  therefore  they  are  indirectly  affected  by  system-wide
               crypto_policies(7).  crypto_policies(7) can not handle the list of host key algorithms directly as doing so would break the order given by the known_hosts file.
```
<br>
# Adjusting Fedora’s Crypto Policies
I discovered that Fedora/RedHat's crypto policies impact OpenSSH. According to the manual for crypto_policies, these policies are configured in:

```
  /etc/crypto-policies/back-ends
           The individual cryptographical back-end configuration files.
           Usually linked to the configuration shipped in the crypto-policies
           package unless a configuration from local.d is added.
```

Reviewing the policy for OpenSSH reveals the necessary changes:
```
~> cat /etc/crypto-policies/back-ends/openssh.config
```

I identified two critical sections that needed changing. To connect, I needed to configure both options. First, I added PubKeyAcceptedAlgorithms=+ssh-rsa, a Fedora specific restriction, but encountered another error:
```
 ~> ssh 10.1.x.y -oKexAlgorithms=+diffie-hellman-group1-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa
Bad server host key: Invalid key length
```

Fedora increased the default RequiredRSASize to 2048, requiring a smaller key length be enabled:
```
~> ssh 10.1.x.y -oKexAlgorithms=+diffie-hellman-group1-sha1 -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedAlgorithms=+ssh-rsa -oRequiredRSASize=+1024
(gp@10.1.x.y) Password:
```
<br>
<br>
**IT WORKS!**
<br>
<br>
# Conclusion
I don't really see the value that PubkeyAcceptedAlgorithms adds over and above HostKeyAlgorithms, which is already found in upstream OpenSSH. I suspect it has something to do about having a centralized place to control broader system policy in [RHEL](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux). I do see value in increasing RequiredRSASize though and happy that Fedora appears to be thinking about security.

With this setup, my ~/.ssh/config file on Fedora now looks like:

```
Host 10.1.x.y
    KexAlgorithms +diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
    RequiredRSASize 1024
```
<br>
While this issue seems specific to Fedora/RHEL for now, it's likely that the OpenBSD/OpenSSH team will adopt similar defaults for RequiredRSASize in the future. I hope this post helps others facing similar challenges. Stay secure!




[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
