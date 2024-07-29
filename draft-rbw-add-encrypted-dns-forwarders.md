---
title: "Hosting Encrypted DNS Forwarders on CPEs"
abbrev: "Encrypted DNS Forwarders on CPEs"
category: info

docname: draft-rbw-add-encrypted-dns-forwarders-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Internet"
workgroup: "Adaptive DNS Discovery"
keyword:
 - DNS
 - deployment
 - Discovery

author:
 -
    fullname: Tirumaleswar Reddy
    organization: Nokia
    city: Bangalore
    region: Karnataka
    country: India
    email: "kondtir@gmail.com"
 -
    fullname: Mohamed Boucadair
    organization: Orange
    city: Rennes
    code: 35000
    country: France
    email: mohamed.boucadair@orange.com
 -
    ins: D. Wing
    name: Dan Wing
    organization: Cloud Software Group Holdings, Inc.
    abbrev: Cloud Software Group
    email: danwing@gmail.com

normative:

informative:
  openwrt:
     title: "OpenWrt Project"
     author:
        org: OpenWrt
        date: false
     target: https://openwrt.org/

  prpl:
     title: "Prpl Foundation"
     author:
        org: Prpl Foundation
        date: false
     target: https://prplfoundation.org/

  prplwrt:
     title: "Prpl WRT"
     author:
        org: Prpl Foundation
        date: false
     target: https://prplfoundation.org/project/prplwrt/

  TR-069:
     title: "CPE WAN Management Protocol"
     author:
        org: The Broadband Forum
        date: December 2018
     target: https://www.broadband-forum.org/technical/download/TR-069.pdf

  plex-https:
     title: "How Plex Is Doing HTTPS for all its Users"
     date: June 2015
     target: https://words.filippo.io/how-plex-is-doing-https-for-all-its-users

  https-local-dom:
     title: "HTTPS for Local Domains"
     author:
       org: Mozilla
       name: Martin Thomson
     date: April 2021
     target: https://docs.google.com/document/d/170rFC91jqvpFrKIqG4K8Vox8AL4LeQXzfikBQXYPmzU

--- abstract

Typical connectivity service offerings based upon on Customer Premise Equipment (CPEs)
involve DNS forwarders on the CPE for various reasons (offer
local services, control the scope/content of information in DNS, ensure better
dependability for local service, provide control to users, etc.). Upgrading DNS to use
encrypted transports introduces deployment complications as to how to sustain current
offerings with local services. Solutions are needed to ease operating
DNS forwarders in CPEs while allowing to make use of encrypted DNS  capabilities.

This document describes the problem and why some existing solutions can't be used for
these deployments. For example, Star certificates and name constraints extension suffer from
the problem of deploying a new feature to CAs, TLS clients, and servers.

The scope of this document is encrypted DNS servers deployed on managed CPEs.

--- middle

# Introduction

   Customer Premises Equipment (CPEs, also called Home Routers) are a
   critical component of the home network, and their security is
   essential to protecting the devices and data that are connected to
   them.  For example, the prpl Foundation {{prpl}} has developed a number
   of initiatives to promote home router security and hardening.  The
   prplWrt project {{prplwrt}} is an initiative in prpl Foundation that
   aims to improve the security and performance of open-source router
   firmware, such as OpenWrt {{openwrt}}.  OpenWrt is an open-source
   operating system that is designed to run on a wide range of routers
   and embedded devices.  It now includes support for containerization
   technology such as Docker, making it possible to run containerized
   applications on a home router.  Further, DNS providers have optimized
   the encrypted DNS forwarder to run in a container in home routers.

   However, upgrading DNS to use encrypted transports (e.g., DNS over HTTPS (DoH) {{?RFC8484}}, DNS over TLS (DoT) {{?RFC7858}},
   and DNS over QUIC (DoQ) {{?RFC9250}}) introduces deployment complications as to how to sustain current
offerings with local services.

   This document describes the problem encountered to host encrypted DNS resolvers
   in managed CPEs. It also discusses limitations of existing solutions.

The document does not focus on generic considerations related to deploying DNS proxies. The reader may
refer to {{?RFC5625}} for such matters.

# Conventions and Definitions

   This document makes use of the terms defined in {{?RFC9499}}.

   The following additional terms are used:

   DHCP:
   :  refers to both DHCPv4 and DHCPv6.

   Do53:
   :  refers to unencrypted DNS.

   DNR:
   :  refers to the Discovery of Network-designated Resolvers
      procedure defined in {{!RFC9463}}.

   DDR:
   :  refers to the Discovery of Designated Resolvers procedure
      defined in {{!RFC9462}}.

   Encrypted DNS:
   :  refers to a scheme where DNS exchanges are
      transported over an encrypted channel.  Examples of encrypted DNS
      are DoH {{?RFC8484}}, DoT {{?RFC7858}}, and DoQ {{?RFC9250}}.

   Managed CPE:
   :  refers to a CPE that is managed by an ISP or CPE vendor
      or Security Service Provider.

# Proxied DNS In Local Networks

   {{fig1}} shows various network setups where the CPE embeds a caching
   encrypted DNS forwarder.


~~~~aasvg
   (a)
                  .--------------.         .-------------.
                 |                |       |      ISP      |
          Host---|      LAN      CPE------|  DNS Resolver |
            |    |                |       |'-------------'
            |     '--------------'|       |         |
            |                     |       |         |
            |                     |<=DNR=>|         |
            |<========DNR========>|       |         |
            |                     |                 |
            |<===Encrypted DNS===>|<=Encrypted DNS=>|
            |                     |                 |

   (b)
              .--------------.         .------------.
             |                |       |              |      3rd Party
      Host---|      LAN      CPE------|      ISP     |------DNS Resolver
        |    |                |       |              |        |
        |     '--------------'|       |'------------'         |
        |                     |       |                       |
        |                     |<=DNR=>|                       |
        |<========DNR========>|       |                       |
        |                     |                               |
        |<===Encrypted DNS===>|<========Encrypted DNS========>|
        |                     |                               |
~~~~
{: #fig1 title="Proxied Encrypted DNS Sessions"}

   For all the cases shown in {{fig1}}, the CPE advertises itself as the
   default DNS server to the hosts it serves in the LAN.  The CPE relies
   upon DHCP or RA to advertise itself to internal hosts as the default
   encrypted DNS forwarder using DNR. When receiving a DNS request that a CPE cannot
   handle locally, the CPE forwards the request to an upstream encrypted
   DNS. The upstream encrypted DNS can be hosted by the ISP or provided
   by a third party.

   Such a forwarder presence is required for (but not limuted to):

   * IPv4 service continuity purposes (e.g., {{Section 3.1 of ?RFC8585}}).
   * Supporting advanced services within a local network such as:
     + malware filtering,
     + parental control,
     + Manufacturer Usage Description (MUD) {{?RFC8520}} to only allow
   intended communications to and from an IoT device, and
     + offer multicast DNS proxy service for the ".local" domain {{?RFC6762}}.

   When the CPE behaves as a DNS forwarder, DNS communications can be decomposed into
   two legs to resolve queries:

   *  The leg between an internal host and the CPE.

   *  The leg between the CPE and an upstream DNS resolver.

# Requirements

R-REDUCE-CA: Reduce the use of a Certificate Authority for each CPE,
compared to obtaining and renewing one certificate for each CPE from a public
Certificate Authority.

R-ELIMINATE-CA: Eliminate using Certificate Authorities for each CPE.

R-SUPPORT-CA: Existing support by Certificate Authorities.

R-SUPPORT-CLIENT: Existing support client libraries or client programs


# Non-Requirements

NR-REVOKE: Provide mechanism to revoke certificate. End users are
extremely unlikely to contact the device vendor or their ISP if a CPE
device is replaced (stolen, upgraded).  Rather, the user will replace
the CPE and configure their client devices (laptops, smartphones, IoT
devices) to authorize the new CPE.  As part of that configuration, the
client device might refuse to authorize the previous CPE.  While this
does leave the client devices open to attack by the previous (stolen)
CPE, the attacker would also need to have access to the victim's
physical network or broadcast a strong enough Wi-Fi signal to the
victim's clients.



# Analysis of Solutions to Requirements


This section describes several solutions which can meet some of the requirements.  This is first summarized in the table and
detailed in subsections.


|    solution                 | Reduce CA             | Eliminate CA     | existing CA support | existing client support |
|----------------------------:|:---------------------:|:----------------:|:-------------------:|:-----------------------:|
| Normal certificates         | No                    |  No              |   Yes               |   Yes                   |
| Delegated credentials       | Yes, somewhat         |  No              |   No                |   No, (*)               |
| Name constraints            | Yes                   |  No              |   No                |   No                    |
| STAR certificates           | No                    |  No              |   Yes               |   Yes                   |
| Raw Public Keys             | Yes                   |  Yes             |   n/a               |   Some, (*)             |
| Wildcard certificate        | No                    |  No              |   Yes               |   Yes                   |
| Local Certificate Authority | No                    |  No              |   Yes               |   Yes                   |
{: #table1 title="Summary of Solution Analysis"}



## Normal Certificates {#normal-certificates}

This solution has the CPE request a cloud server obtain a certificate
for the CPE from a public CA. This solution is deployed in production
by Mozilla {{https-local-dom}}, McAfee, and Cujo.  Today, this is best
practice.  However, it suffers from the dependency on both the public
certificate authority and the vendor's service, which are necessary
for both initial deployment and for certificate renewal.

R-REDUCE-CA: no

R-ELIMINATE-CA: no

Both CAs and clients already support the normal mode of operation.

R-SUPPORT-CA: yes

R-SUPPORT-CLIENT: yes

## Delegated Credentials {#delegated}

Delegated credentials {{?RFC9345}} allows the entity operating the CPE (e.g., vendor, ISP) to sign their
own certificates to each of the CPE, rather than relying on a public CA to sign those certificates. The frequency
of CA interactions remains the same as with normal certificates {{normal-certificates}}

R-REDUCE-CA: yes, somewhat by moving CA signing from public CA to a vendor- or ISP-operated CA.

R-ELIMINATE-CA: no

Delegated credentials have no existing support by CAs.  The only client supporting delegated
credentials is Firefox.

R-SUPPORT-CA: no

R-SUPPORT-CLIENT: no, only Firefox


## Name Constraints {#name-constraints}

Name constraints {{Section 4.2.1.10 of ?RFC5280}} allows the entity
operating the CPE (e.g., vendor, ISP) to obtain a certificate from a
public Certificate Authority for a subdomain (dNSName) which is then
used to sign certificates for each CPE.  For example, the network "example.net"
could obtain a name constrained certificate for ".cpe.example.net" and then
issue its own certificates such as "customer-123.cpe.example.net".

R-REDUCE-CA: yes, it reduces interaction with public CAs but has same
number of interactions with the CPE operator's certificate authority.

R-ELIMINATE-CA: no

Name constraints have no existing support by CAs or by clients.

R-SUPPORT-CA: no

R-SUPPORT-CLIENT: no

## Short-Term Automatically Renewed (STAR) Certificates

> Note: STAR by itself does not seem to be a solution.  Perhaps remove this section??

Short-Term Automatically Renewed Certificates {{?RFC8739}} have a
short lifetime (e.g., 30 or 90 days) so that a stolen private key can
only be utilized during that lifetime.  This reduces need for a
certificate revocation list (CRL), OCSP, or OCSP stapling {{Section 8
of ?RFC6066}} (which has a nearly-identical effect as STAR certificates).

R-REDUCE-CA: No

R-ELIMINATE-CA: No

STAR certificates have very good support by CAs via the {{?ACME=RFC8555}} API and perfect support on clients that already perform certificate validation.

R-SUPPORT-CA: yes

R-SUPPORT-CLIENT: yes

## Raw Public Keys

Raw public keys (RPK) {{?RFC7250}} can be authenticated out-of-band or using trust on first use (TOFU).
For a small network, this can be more appealing than a local or remote Certificate Authority signing
keys and dealing with certificate renewal.

R-REDUCE-CA: yes

R-ELIMINATE-CA: yes

Raw public keys have been supported by OpenSSL and wolfSSL since 2023
and by GnuTLS since 2019. However, raw public keys are not supported
by browsers or by curl.

R-SUPPORT-CA: n/a, this system does not use Certificate Authorities at all.

R-SUPPORT-CLIENT: Some; all major libraries support RPK, but clients (browsers and curl) do not support RPK


## Wildcard Certificate {#wildcard}

A wildcard certificate can be issued to the vendor, who then signs each of the CPE certificates. This
removes the public CA from signing the certificates of each CPE; instead, the vendor's CA performs
this signing. Evil twin attacks are prevented by a unique, per-network string in the certificate's
Subject Alt Name (SAN).  This is deployed in production by Plex {{plex-https}}.

R-REDUCE-CA: yes, it reduces interaction with public CAs but has same
number of interactions with the CPE operator's certificate authority.

R-ELIMINATE-CA: no

This operates similar to {#name-constraints} but has advantage of working with existing CAs and
existing clients.

R-SUPPORT-CA: yes

R-SUPPORT-CLIENT: yes

## Local CA: Certificate Authority Built Into CPE

The CPE would be a Certificate Authority capable of signing certificates for other in-home
devices. The CA in the CPE would be limitied to signing only certificates belonging to
that home network (using {{wildcard}}, {delegated}, {{name-constraints}}, or {{wildcard}}).
This allows the CPE to sign certificates for other devices within the network such as
printers, IoT devices, NAS devices, laptops, or anything else needing to be a server
on the local network.

This functionality is beyond the scope of the ADD working group but is
mentioned because it was suggested during an ADD meeting.

R-REDUCE-CA: yes, it reduces interaction with public CAs but has same
number of interactions with the CPE operator's certificate authority for the
CPE itself.  For other devices within the home network (e.g., printers), it
eliminates interaction with the vendor's CA.

R-ELIMINATE-CA: no

While technically the industry could build such a system, building such a system
across the ecosystem of client operating systems would be a daunting task.

R-SUPPORT-CA: yes

R-SUPPORT-CLIENT: yes



# Hosting Encrypted DNS Forwarder in Local Networks

   This section discusses some deployment challenges to host an
   encrypted DNS forwarder within a local network.

## Discovery Mechanisms and Naming Constraints

### Discovery of Designated Resolvers (DDR)

   DDR requires proving possession of an IP address, as the DDR
   certificate contains the server's IPv4 and IPv6 addresses and is
   signed by a certificate authority.  DDR is constrained to public IP
   addresses because (WebPKI) certificate authorities will not sign
   special-purpose IP addresses {{?RFC6890}}, most notably IPv4 private-use
   {{?RFC1918}}, IPv4 shared address {{?RFC6598}}, or IPv6 Unique-Local
   {{?RFC8190}} address space.

   A tempting solution for enabling an encrypted DNS forwarder with DDR is to use the CPE's WAN
   IP address for DDR and prove possession of that IP address.  However,
   the CPE's WAN IPv4 address will not be a public IPv4 address if the
   CPE is behind another layer of NAT (either Carrier Grade NAT (CGN) or
   another on-premise NAT), reducing the success of this mechanism to
   CPE's WAN IPv6 address.

   Also, if the ISP renumbers the subscriber's
   network suddenly (rather than slow IPv6 renumbering described in
   {{?RFC4192}}), encrypted DNS service will be delayed until that new
   certificate is acquired.

###  Discovery of Network-designated Resolvers (DNR)

   DNR requires proving possession of a domain name as the encrypted
   resolver's certificate contains the FQDN. For example, the entity (e.g., ISP or
   network administrator) managing the CPE would assign a unique FQDN to
   the CPE. There are two mechanisms for the CPE to obtain the
   certificate for the FQDN: using one of its WAN IP addresses or
   requesting its signed certificate from an Internet-facing server used
   for remote CPE management (e.g., the Auto Configuration Server (ACS)
   in the CPE WAN Management Protocol {{TR-069}}).

   If the CPE's WAN IP address is used, the CPE needs a public IPv4 or a global unicast IPv6
   address together with DNS A or AAAA records pointing to that CPE's
   WAN address to prove possession of the DNS name to obtain a (WebPKI)
   CA-signed certificate. That is, the CPE fulfills the DNS or HTTP
   challenge discussed in Automatic Certificate Management Environment (ACME) {{?RFC8555}}.  However, a CPE's WAN address
   will not be a public IPv4 address if the CPE is behind another layer
   of NAT (either a CGN or another on-premise NAT), reducing the success
   of this mechanism to a CPE's WAN IPv6 address. If the ISP renumbers the subscriber's
   network, the DNS record will also need to expire and changed to
   reflect the new IP address.

# Limitations of Existing Solutions

## Certificate Issuance Issues

  The following lists some limitations for certificate issuance:

   *  In case of large scale of CPEs (e.g., millions of devices),
      issuing certificate request for a large number of subdomains could
      be treated as an attack by the certificate authorities to
      overwhelm it.

   *  Dependency on the CA to issue a large number of certificates.

   *  If the CPE uses one of its WAN IP addresses to obtain the
      certificate for the FQDN, the Internet-facing HTTP server or a DNS
      authoritative server on the CPE to complete the HTTP or DNS
      challenge can be subjected to DDoS attacks.

## Delegated Certificate Issuance

   Let's consider that the encrypted DNS forwarder is hosted on a CPE and provisioned by a
   service (e.g., ACS) in the operator's network. Also, let's assume that each CPE is assigned
   a unique FQDN (e.g., "cpe-12345.example.com" where 12345 is a unique
   number).

   > It is best to ensure that such an FQDN does not carry any
   Personally Identifiable Information (PII) or device identification
   details like the customer number or device's serial number.

   The CPE generates a public and private key-pair, builds a certificate signing
   request (CSR), and sends the CSR to a service in the operator
   managing the CPE.  Upon receipt of the CSR, the operator's service
   can utilize certificate management protocols like ACME {{?RFC8555}} to automate
   certificate management functions such as domain validation procedure,
   certificate issuance, and certificate revocation.

   The challenge with this technique is that the service will have to
   communicate with the CA to issue certificates for millions of CPEs.
   If an external CA is unable to issue a certificate in time or replace
   an expired certificate, the service would no longer be able to
   present a valid certificate to a CPE.  When the service requests
   certificate issuance for a large number of subdomains (e.g., millions
   of CPEs), it may be treated as an attacker by the CA to overwhelm it.
   Furthermore, the short-lived certificates (e.g., certificates that
   expire after 90 days) issued by the CA will have to be renewed
   frequently.  With short-lived certificates, there is a smaller time
   window to renew a certificate and, therefore, a higher risk that a CA
   outage will negatively affect the uptime of the encrypted DNS
   forwarders on CPEs (and the services offered via these CPEs).

   These challenges can be addressed by using protocols like
   ACME to automate the certificate renewal process, ensuring certificates
   are renewed well before expiration. Additionally, incorporating another
   CA as a backup can provide redundancy and further mitigate the risk of
   outages. By having a secondary CA, the service can switch to the backup
   CA in case the primary CA is unavailable, thus maintaining continuous
   service availability and reducing the risk of service disruption.

   It offers the additional advantage of improving the security of
   Browser and CPE interactions. This ensures that HTTPS access to
   the CPE is possible, allowing the device administrator to securely
   communicate with and manage the CPE.

## Limitations of Name Constraints Extension

A service managing the CPEs could get a CA certificate with name
      constraints extension ({{Section 4.2.1.10 of !RFC5280}}) and the
      service would in-turn act as an ACME server to provision end-entity certificates on CPEs.

* Con:
  + Name constraints extension is not yet supported by CAs,
         although {{!RFC5280}} was standardized way back in 2008.

* Pro:
  + Avoids changing TLS client and server (e.g., stunnel or openssl).

## Limitations of Star certificates

{{!RFC9115}} defines a profile of the ACME protocol for generating
      Delegated certificates.  It allows the CPEs to request from a
      service managing the CPEs, acting as a profiled ACME server, a
      certificate for a delegated identity, i.e., one belonging to the
      service.  The service then uses the ACME protocol (with the
      extensions described in {{?RFC8739}}) to request issuance of a
      short-term, Automatically Renewed (STAR) certificate for the same
      delegated identity.  The generated short-term certificate is
      automatically renewed by the ACME CA, periodically fetched by
      the CPEs, and used to act as encrypted DNS forwarders.

The service can end the delegation at any time by instructing the CA
      to stop the automatic renewal and letting the certificate expire
      shortly thereafter.  Star certificates requires support by CAs but
      does not require changes to the deployed TLS ecosystem.

* Cons:
  + Star certificates require support by CAs.
  + A primary use case of Star certificates is that of a
         Content Delivery Network (CDN), the third party, terminating
         TLS sessions on behalf of a content provider (the holder of a
         domain name).  The number of star certificates required for a
         CDN use case will be very much lower than the use case
         discussed in this draft.  It is yet to be seen if CAs will
         agree to support star certificates at a scale of millions of
         CPEs.

* Pro:
  + Avoids changing TLS client and server.

# Security Considerations

   DNR-related security considerations are discussed in
   {{Section 7 of !RFC9463}}.  Likewise, DDR-related security considerations
   are discussed in {{Section 7 of !RFC9462}}.


   The communication between the CPE and endpoints is encrypted using mechanisms such as WPA2/3,
   and any communication with the DNS server co-located on the CPE is also protected.
   However, the client does not know whether the DNS server is co-located on the CPE or not.
   If the client uses clear text DNS (i.e., Do53), it will assume that the DNS messages are susceptible to
   pervasive monitoring. For instance, in an Enterprise deployment, multiple network devices
   could exist between an endpoint and the CPE;  hosting an encrypted DNS server on
   the CPE minimizes the impact of a breach, which is an essential zero trust principle. Furthermore,
   the client and user would be able to identify the entity hosting the encrypted DNS server
   using the ADN assigned to it.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

This draft is triggered by the discussion that occured at IETF#119 (Brisbane)
about the need to frame the problem to be solved. Thanks to all the
participants to that discussion.

   Acknowledgements from {{?I-D.reddy-add-delegated-credentials}}:
   : Thanks to Neil Cook, Martin Thomson, Tommy Pauly, Benjamin Schwartz,
   and Michael Richardson for the discussion and comments.
