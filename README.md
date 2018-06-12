# Secure, Untraceable Mobile Guide

This guide will show you how to create a secure, untraceable mobile device that can use both encrypted messaging services, like Signal, and the normal cellphone network.

![alt text](https://github.com/y22k/secure-untraceable-mobile-guide/raw/master/diagram.jpg)

## Introduction

Burner phones are going out of fashion. With large scale deployment of voice recognition, ID requirements for buying SIM cards and baseband hacks, how do we stay secure and under the radar? The answer is simple: a non-GSM, or airplane moded device hooked up to the network through a mobile hotspot and disposable LTE SIM card that receives calls through Signal (or any other secure messaging service), can use Whatsapp, and is still compatible with the traditional phone network through a GSM-VoIP gateway.

The security principle is to hide your permanent mobile device behind a disposable hotspot and SIM card. You will still be able to use your “official” phone number to sign up for Signal/WhatsApp and receive/place calls with it over Linphone. However, an attacker can’t easily find out where you are, who you are talking to, which device you are using, and will consequently lack a target to attack.

A well resourced attacker could probably still find you by tracking all mobile routers in a suspect area or connections to your VPN of choice. This is where changing up your behavior can become helpful. If you can’t prevent your attackers from finding you, at least make it hard for them.

In this guide I will try to stay as device unspecific as possible, however, I do feel that specific instructions on hardware that I have tested can help you.

## Equiptment

General OPSEC (à la Grugq):
* Keep your mouth shut
* Guard secrets
* Need to know
* Never let anyone get into position to blackmail you

Device OPSEC
* Buy your hardware in a store with cash if at all possible
* Strong passwords
* Keep everything updated
* Maintain physical control over devices

The Mobile Device
* Hardware resistant to GSM exploits: Google Pixel 2 on airplane mode or iPod Touch
* Secure OS like CopperheadOS or iOS
* Select communication apps: email client, OpenVPN, Orbot, Signal, WhatsApp, Linphone

The Mobile Router
* A disposable mobile router, like the Huawei E5770, with WiFi and Ethernet
  * It would be preferable to own a router that supports Ethernet to minimize wireless leakage, but it could get expensive to toss every few months relative to a cheaper model. Oh well. Remember that all devices have unique identifiers so it will break your anonymity to use different SIM cards on the same hotspot.
  
The SIM Card
* High capacity (10 GB) 4G LTE SIM card. Definitely buy in cash. Toss after use.

The VPN
* Non-US Company
* OpenVPN or Wireguard
* Some good ones are: Mullvad, ProtonVPN

The SIP Server
* Self-hosted Freeswitch server 
* Located either in your private domicile (setting up behind a NAT will be difficult) or with a trustworthy hosting provider
* No logs, no crime!

The GSM VoIP Gateway
* OpenVox WGW1002G with 2 hot swap GSM Channels or similar
* Located in your private domicile or other secure location of your choice
* OpenVox is a chinese company. Are they backdooring us? Who knows. I would assume that any GSM calls are being monitored anyways. The important thing is that they can’t be traced back to your physical location because of your VPN/Tor.

## Step-by-Step

### SIP Server (with TLS+SRTP)
#### Install CentOS 7
* Get a VPS with CentOS 7. Minimal hardware specs will be fine.
 * Here’s a list of good options https://torbitcoinvps.github.io/
* SSH into your VPS with the root account
* Change root password with: `passwd`
* You might want to create a user and install sudo at this point. There are various guides online. This one is good https://plusbryan.com/my-first-5-minutes-on-a-server-or-essential-security-for-linux-servers
#### Install Freeswitch
`yum install -y http://files.freeswitch.org/freeswitch-release-1-6.noarch.rpm epel-release
yum install -y freeswitch-config-vanilla freeswitch-lang-* freeswitch-sounds-*
yum install letsencrypt nano
systemctl enable freeswitch`
#### Configure Freeswitch
* Change default extension password

`nano /etc/freeswitch/vars.xml` and change `<X-PRE-PROCESS cmd="set" data="default_password=[your extension password here]"/>`
* Change domain in vars.xml

`<X-PRE-PROCESS cmd="set" data="domain=[your FQDN here]"/>`
* Enable TLS in vars.xml

Change

`<!-- Internal SIP Profile -->
 <X-PRE-PROCESS cmd="set" data="internal_auth_calls=true"/>
 <X-PRE-PROCESS cmd="set" data="internal_sip_port=5060"/>
 <X-PRE-PROCESS cmd="set" data="internal_tls_port=5061"/>
 <X-PRE-PROCESS cmd="set" data="internal_ssl_enable=false"/>`

to

`<!-- Internal SIP Profile -->
 <X-PRE-PROCESS cmd="set" data="internal_auth_calls=true"/>
 <X-PRE-PROCESS cmd="set" data="internal_sip_port=5060"/>
 <X-PRE-PROCESS cmd="set" data="internal_tls_port=5061"/>
 <X-PRE-PROCESS cmd="set" data="internal_ssl_enable=true"/>`
* Enable outbound SRTP

`nano /etc/freeswitch/dialplan/default.xml`

and change

```<condition field="${rtp_has_crypto}" expression="^($${rtp_sdes_suites})$" break="never">
        <action application="set" data="rtp_secure_media=true"/>
        <!-- Offer SRTP on outbound legs if we have it on inbound. -->
        <!-- <action application="export" data="rtp_secure_media=true"/> -->
      </condition>

      <!--
         Since we have inbound-late-negotation on by default now the
         above behavior isn't the same so you have to do one extra step.
        -->
      <condition field="${endpoint_disposition}" expression="^(DELAYED NEGOTIATION)"/>
      <condition field="${switch_r_sdp}" expression="(AES_CM_128_HMAC_SHA1_32|AES_CM_128_HMAC_SHA1_80)" break="never">
        <action application="set" data="rtp_secure_media=true"/>
        <!-- Offer SRTP on outbound legs if we have it on inbound. -->
        <!-- <action application="export" data="rtp_secure_media=true"/> -->
      </condition>
```
to

```<condition field="${rtp_has_crypto}" expression="^($${rtp_sdes_suites})$" break="never">
        <action application="set" data="rtp_secure_media=true"/>
        <!-- Offer SRTP on outbound legs if we have it on inbound. -->
        <action application="export" data="rtp_secure_media=true"/>
      </condition>

      <!--
         Since we have inbound-late-negotation on by default now the
         above behavior isn't the same so you have to do one extra step.
        -->
      <condition field="${endpoint_disposition}" expression="^(DELAYED NEGOTIATION)"/>
      <condition field="${switch_r_sdp}" expression="(AES_CM_128_HMAC_SHA1_32|AES_CM_128_HMAC_SHA1_80)" break="never">
        <action application="set" data="rtp_secure_media=true"/>
        <!-- Offer SRTP on outbound legs if we have it on inbound. -->
        <action application="export" data="rtp_secure_media=true"/>
      </condition>
```
* Create letsencrypt certificate

`letsencrypt certonly --standalone -d [your FQDN here]`
* Make TLS directory

`mkdir /etc/freeswitch/tls`
* Copy letsencrypt certificate into TLS directory

`cat /etc/letsencrypt/live/[your FQDN here]/fullchain.pem /etc/letsencrypt/live/[your FQDN here]/privkey.pem > /etc/freeswitch/tls/wss.pem && cat /etc/letsencrypt/live/[your FQDN here]/fullchain.pem /etc/letsencrypt/live/[your FQDN here]/privkey.pem > /etc/freeswitch/tls/tls.pem`
* Change TLS directory permissions
`chown freeswitch.daemon /etc/freeswitch/tls && chown -R freeswitch.daemon /etc/freeswitch/tls`
* Disable logging
`nano /etc/freeswtich/autoload_configs/modules.conf.xml`
  * Comment out mod_logfile and mod_cdr_csv
  * Save

```fs_cli
unload mod_logfile
unload mod_cdr_csv
/quit
```
  * Delete any log files that already exist in /var/log/freeswitch
* Restart Freeswitch

`service freeswitch restart`
