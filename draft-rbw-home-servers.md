---
title: "Identifying and Authenticating Home Servers: Requirements and Solution Analysis"
abbrev: "Home Servers"
category: info

docname: draft-rbw-home-servers-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Internet"
workgroup: "network"
keyword:
 - deployment
 - encryption
 - TLS


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
  Matter:
    title: "Matter: The Foundation for Connected Things"
    seriesinfo:
      Version: 1.2
    author:
      org: Connectivity Standards Alliance
    date: 23 October 2023
    target: https://csa-iot.org/all-solutions/matter/

informative:
  https-local-dom:
     title: "HTTPS for Local Domains"
     author:
       org: Mozilla
       name: Martin Thomson
     date: April 2021
     target: https://docs.google.com/document/d/170rFC91jqvpFrKIqG4K8Vox8AL4LeQXzfikBQXYPmzU

--- abstract

Obtaining CA-signed certificates for servers within a home network
is difficult and causes problems at scale.  This document explores
requirements to improve this situation and analyzes existing
solutions.

--- middle

# Introduction

   This document describes the problem encountered to host servers in
   a home network and how to connect to those servers using an
   encrypted transport protocol such as TLS or DTLS.  It also identifies a set
   of requirements and discusses to what extent existing solutions can
   (or can't) meet these requirements.


# Scope

This document describes the current state of the art for deploying
encrypted servers within the home.  New mechanisms should be described
in separate documents.

There are three types of equipment:

* legacy local equipment which lacks memory, CPU, or sufficient
  security to reasonably support an encrypted transport protocol and
  associated key management.  An example is a 10 year old router with
  built-in 802.11n Wi-Fi.  This type of equipment is out of scope.

* high functioning local equipment that provides sufficient hardware
  and software capability to support an encrypted transport protocol
  and associated key management.  An example is a printer, NAS, or
  higher-end consumer router.

* high functioning virtualized equipment, where some functions are
  provided via local or remote software.  An example is virtualized
  CPE (vCPE).

# Requirements

With automated certificate enrollment and renewal {{?ACME=RFC8555}}, a
public Certification Authority can sign a certificate for each CPE.
This causes a few issues:

   *  In case of large scale of CPEs (e.g., millions of devices),
      issuing certificate request for a large number of subdomains could
      be treated as an attack by the certificate authorities to
      overwhelm it.  This can be resolved with contracts and
      payment but all of these problems are barriers to moving to
      another certificate authority.

   *  Dependency on the CA to issue a large number of certificates,
      which causes CA availability to impact service availability.

   * If the CPE uses one of its WAN IP addresses to obtain the
      certificate for the FQDN, the Internet-facing HTTP server or a
      DNS authoritative server on the CPE to complete the HTTP or DNS
      challenge can be subjected to attacks. To avoid this exposure,
      and to accomodate environments where the CPE cannot obtain a
      public IP address (e.g., CGN), a vendor-operated service would
      need to obtain the signed certificate on the CPE's behalf.

For these reasons, it is desirable to reduce the need for a public CA.

   R-REDUCE-CA:
   : Reduce the use of a public Certification Authority.

Taking an additional step from the previous requirement, eliminating
running one's own Certification Authority would avoid the complexities
of certificate management.

   R-ELIMINATE-CA:
   : Eliminate using Certification Authorities for each CPE.

The ability to immediately deploy using existing Certification
Authorities is important to evaluate.

   R-SUPPORT-CA:
   : Existing support by Certification Authorities.

The ability to immediately deploy on clients is important to evaluate.

  R-SUPPORT-CLIENT:
   : Existing support client libraries or client software intances.


End users are extremely unlikely to contact the device vendor or their
ISP if a CPE device is replaced (stolen, upgraded, etc.).  Rather, the
user will replace the CPE and configure their client devices (laptops,
smartphones, IoT devices, etc.) to authorize the new CPE.  As part of
that configuration, the client device can refuse to authorize the
previous CPE (e.g., encourage the user to remove authorization
for the previous certificate).  Even without ability to revoke a
certificate, the attacker in possession of the associated private key
would also need to get on-path via access to the victim's physical
network or broadcast a strong enough Wi-Fi signal to the victim's clients.

  R-REVOKE-AUTH:
  : Provide a mechanism for the end user to disable access to a previously-
  authorized encrypted service, to accomodate a lost/stolen/sold device.
  This might be via certificate revocation (via a Certificate Authority) or
  via some other means.



# Analysis of Solutions to Requirements


This section describes several solutions which can meet some of the requirements.  This is first summarized in {{table1}} and
detailed in the following subsections.


|    Solution                 | Reduce CA             | Eliminate CA     | Existing CA Support | Existing Client Support | Revoke Auth |
|----------------------------:|:---------------------:|:----------------:|:-------------------:|:-----------------------:|:-----------:|
| Normal certificates         | No                    |  No              |   Yes               |   Yes                   |   Some      |
| Delegated credentials       | Yes, somewhat         |  No              |   No                |   No, (*)               |   Some      |
| Name constraints            | Yes                   |  No              |   No                |   No                    |   Some      |
| ACME delegated certificates | No                    |  No              |   Yes               |   Yes                   |   Some      |
| Raw Public Keys             | Yes                   |  Yes             |   n/a               |   Some, (*)             |   Yes       |
| Self-Signed Certificate     | Yes                   |  Yes             |   n/a               |   Yes, poor experience  |   Yes       |
| Local Certificate Authority | No                    |  No              |   Yes               |   Yes                   |   Yes       |
| Matter                      | Yes                   |  No              |   Yes               |   Yes                   |   Yes       |
{: #table1 title="Summary of Solution Analysis"}



## Normal Certificates {#normal-certificates}

This solution has the CPE requests a (cloud) server to obtain a certificate
for the CPE from a public CA. This solution is deployed in production
by Mozilla {{https-local-dom}}, McAfee, and Cujo.  Today, this is best
practice.  However, it suffers from the dependency on both the public
Certification Authority and the vendor's service (necessary because the
CPE cannot always obtain a publicly-accessible IPv4 address necessary
to get an ACME-signed certificate itself), which are necessary
for both initial deployment and for certificate renewal.

R-REDUCE-CA: no

R-ELIMINATE-CA: no

Both CAs and clients already support the normal mode of operation.

R-SUPPORT-CA: yes

R-SUPPORT-CLIENT: yes

R-REVOKE-AUTH:
  : Somewhat, if client retrieves CRL frequently and if CRL is updated
    frequently, and user has mechanism to declare the certificate as
    invalid.


## Delegated Credentials {#delegated}

Delegated credentials {{?RFC9345}} allows the entity operating the CPE
(e.g., vendor or ISP) to sign a 7-day validity for the CPE's public key.
The frequency of CA interactions remains the same as with normal
certificates ({{normal-certificates}}), but the interactions are with
the vendor's service rather than the public CA.

As currently specified in {{?RFC9345}}, the same name would be issued to all CPEs
making it impossible to identify whether the delegated credential is
issued to the intended CPE or an "evil-twin" CPE. This drawback can be corrected
by enhancing {{?RFC9345}} to include a string that uniquely identifies the
delegated credential (e.g., including hash of customer id or other unique identifier
in the FQDN such as "HASH.cpe.example.com").

  > For the sake of simplifying the analysis, this document assumes such an
  > enhancement to {{?RFC9345}} has been standardized and deployed.

R-REDUCE-CA:
  : yes, somewhat by moving CA signing from public CA to a
vendor- or ISP-operated service.

R-ELIMINATE-CA: no

Delegated credentials have no existing support by CAs. Clients need to
support {{Section 4.1.1 of ?RFC9345}} which requires sending an
extension in their TLS 1.3 ClientHello.  The only client supporting
delegated credentials is Firefox.

R-SUPPORT-CA: no

R-SUPPORT-CLIENT: no, only supported by Firefox

R-REVOKE-AUTH:
  : Somewhat, if client retrieves CRL frequently and if CRL is updated
    frequently, and user has mechanism to declare the certificate as
    invalid.


## Name Constraints {#name-constraints}

Name constraints ({{Section 4.2.1.10 of ?RFC5280}}) allows the entity
operating the CPE (e.g., vendor or ISP) to obtain a certificate from a
public Certification Authority for a subdomain (dNSName) which is then
used to sign certificates for each CPE.  For example, the network "example.net"
could obtain a name constrained certificate for ".cpe.example.net" and then
issue its own certificates such as "customer-123.cpe.example.net".

R-REDUCE-CA:
  : yes, it reduces interaction with public CAs but has same
number of interactions with the CPE operator's CA.

R-ELIMINATE-CA: no

Name constraints have no existing support by CAs or by clients.

R-SUPPORT-CA: no

R-SUPPORT-CLIENT: no

R-REVOKE-AUTH:
  : Somewhat, if client retrieves CRL frequently and if CRL is updated
    frequently, and user has mechanism to declare the certificate as
    invalid.


## ACME Delegated Certificates

ACME Delegated Certificates {{?RFC9115}} allows the CPE to use a vendor-
operated service to obtain a CA-signed ACME delegated certificate. It allows
the CPE to request from a service managing the CPE, acting as a
profiled ACME server, a certificate for a delegated identity,
i.e., one belonging to the CPE. The CPE then uses the ACME protocol (with the
extensions described in {{?RFC8739}}) to request issuance of a short-term,
Automatically Renewed (STAR) certificate for the same delegated identity.
The generated short-term certificate is automatically renewed by the public CA,
is periodically fetched by the CPE.

R-REDUCE-CA: No

R-ELIMINATE-CA: No

ACME delegated certificates do not require changes to client authentication libraries or operation.

R-SUPPORT-CA: Unknown

R-SUPPORT-CLIENT: yes

R-REVOKE-AUTH:
  : Somewhat, if client retrieves CRL frequently and if CRL is updated
    frequently, and user has mechanism to declare the certificate as
    invalid.


## Raw Public Keys

Raw public keys (RPK) {{?RFC7250}} can be authenticated out-of-band or using trust on first use (TOFU).
For a small network, this can be more appealing than a local or remote Certification Authority signing
keys and dealing with certificate renewal.

R-REDUCE-CA: yes

R-ELIMINATE-CA: yes

RPKs have been supported by OpenSSL and wolfSSL since 2023
and by GnuTLS since 2019. However, RPKs are not supported
by browsers or by curl.

R-SUPPORT-CA:
  : n/a, this system does not use Certification Authorities at all.

R-SUPPORT-CLIENT:
  : Some; all major libraries support RPK, but clients (browsers and curl) do not support RPK.
Further, DNR and DDR in verified discovery mode expect to encounter certificates and do not support RPK.

R-REVOKE-AUTH:
  : Yes, user can remove the raw public key from list of authorized public keys.


## Self-signed Certificate

A self-signed certificate requires the client to authorize the connection, which is usually
a "click OK to continue" dialog box and is a trust on first use (TOFU) solution.  While it is
possible the user verifies the certificate matches expectations, this seldom occurs. The
certificate warnings are normalized by users which weakens security overall.

R-REDUCE-CA: yes, public CA's are not used at all.

R-ELIMINATE-CA: yes

Existing clients support self-signed certificates fairly well, but the
user experience with clients is poor: some clients can't remember to
trust a certificate and clicking through certificate warnings will
become normalized.

Modifying self-signed certificate handling for ".local" {{?RFC6761}}
or ".internal" {{?I-D.davies-internal-tld}} might be worth further
study.

R-SUPPORT-CA:
  : n/a, this system does not use Certification Authories at all.

R-SUPPORT-CLIENT: Yes, but poor user experience.

R-REVOKE-AUTH:
  : Yes, user can remove the certificate from list of authorized certificates.


## Local CA: Certification Authority Built Into CPE

The CPE would be a CA capable of signing certificates for other in-home
devices. The CA in the CPE would be limitied to signing only certificates belonging to
that home network (using Sections {{<delegated}} or {{<name-constraints}}).
This allows the CPE to sign certificates for other devices within the network such as
printers, IoT devices, NAS devices, laptops, or anything else needing to be a server
on the local network.

> This feature is beyond the scope of the ADD working group but is
mentioned because it was suggested during an ADD meeting.

R-REDUCE-CA:
  : yes, it reduces interaction with public CAs but has same
number of interactions with the CPE operator's CA for the
CPE itself.  For other devices within the home network (e.g., printers), it
eliminates interaction with the vendor's CA.

R-ELIMINATE-CA: no

While technically the industry could build such a system, building such a system
across the ecosystem of client operating systems would be a daunting task.

R-SUPPORT-CA: yes

R-SUPPORT-CLIENT:
  : yes, if the clients add the home's CA to their trust list.

R-REVOKE-AUTH:
  : Yes, user can update CRL on local certificate authority, and clients
    can retrieve the updated CRL.


## Matter

Home devices could be enrolled in {{Matter}} and client devices could
be configured to trust Matter certificates.

R-REDUCE-CA:
  : yes, it reduces interaction with public CAs but has same
number of interactions with the device vendor's CA to issue the
Matter Device Attestation Certificate (DAC) to each device.

R-ELIMINATE-CA: no

R-SUPPORT-CA: yes

R-SUPPORT-CLIENT:
  : yes, if the clients add the Matter Product Attestation Intermediates (PAIs) to their trust list.

R-REVOKE-AUTH: Yes



# Security Considerations

TBD.

# IANA Considerations

This document has no IANA actions.

--- back

# Example of Delegated Certificate Issuance

   Let's consider that the customer router, needing a certificate for
   its HTTPS-based management, is provisioned by a service (e.g., ACS)
   in the operator's network. Also, let's assume that each CPE is
   assigned a unique FQDN (e.g., "cpe-12345.example.com" where 12345
   is a unique number).

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

# Acknowledgments
{:numbered="false"}

This draft is the result of discussions at IETF119 and IETF120 in the
ADD working group.  Thanks to all the participants at the microphone,
hallways, and mailing list.

   Acknowledgements from {{?I-D.reddy-add-delegated-credentials}}:
   : Thanks to Neil Cook, Martin Thomson, Tommy Pauly, Benjamin Schwartz,
   and Michael Richardson for the discussion and comments.
