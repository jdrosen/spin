---
title: Simple Protocol for Inviting Numbers (SPIN)
abbrev: SPIN
docname: draft-rosenberg-dispatch-spin
date: 2022-07-01

# stand_alone: true

ipr: trust200902
area: Applications
wg: Dispatch
kw: Internet-Draft
cat: std

coding: us-ascii
pi:    # can use array (if all yes) or hash here
#  - toc
#  - sortrefs
#  - symrefs
  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

author:
      
        ins: J. Rosenberg
        name: Jonathan Rosenberg
        org: Five9
        email: jdrosen@jdrosen.net
      

normative:
    RFC2119:
    RFC3261:
    RFC8224:
    RFC8225:
    I-D.rosenberg-dispatch-cloudsip:
    I-D.ietf-mls-architecture:

informative:

--- abstract

This document introduces a framework and a protocol for facilitating voice, video and messaging interoperability between application providers. This work is motivated by the recent passage of regulation in the European Union - the Digital Markets Act (DMA) - which, amongst many other provisions, requires that vendors of applications with a large number of users enable interoperability with applications made by other vendors. While such interoperability is broadly present within the public switched telephone network, it is not yet commonplace between over-the-top applications, such as Facetime, WhatsApp, and Facebook Messenger. This document specifically defines the Simple Protocol for Inviting Numbers (SPIN) which is used to deliver invitations to mobile phone numbers that can bootstrap subsequent communications over the Internet. 

--- middle

Introduction        {#problems}
============

Voice, video and messaging today is commonplace on the Internet, enabled by two distinct classes of software. The first are those provided by telecommunications carriers that make heavy use of standards, such as the Session Initiation Protocol (SIP) {{RFC3261}}. In this approach - which we call the telco model - there is interoperability between different telcos, but the set of features and functionality is limited by the rate of definition and adoption of standards, often measured in years or decades. The second model - the app model - allows a single entity to offer an application, delivering both the server side software and its corresponding client-side software. The client-side software is delivered either as a web application, or as a mobile application through a mobile operating system app store. The app model has proven incredibly successful by any measure. It trades off interoperability for innovation and velocity. 

The downside of the loss of interoperability is that entry into the market place by new providers is difficult. Applications like WhatsApp, Facebook Messenger, and Facetime, have user bases numbering in the hundreds of millions to billions of users. Any new application cannot connect with these user bases, requiring the vendor of the new app to bootstrap its own network effects. 

This situation has recently drawn the attention of regulators, and was one of the motivations behind the Digital Markets Act (DMA) in the European Union. Amongst its many provisions, it requires vendors of large communications platforms to enable interoperability with third party vendors. It does not, of course, specify an actual set of protocols or technologies for enabling that interoperability. 

This document seeks to fill that void, by defining a framework for such interoperability. This framework seeks to strike a balance between innovation and standardization, by identifying only those portions of the protocol stack that must be standardized in order to achieve end-to-end security for a minimum feature set between providers, and leaving everything else to APIs and protocols which each vendor can define on it's own. 

This framework identifies the need for a new protocol to solve the identity mapping problem. Specifically, how does an originating user using one application identify a target user in a different application with which they wish to communicate, and then obtain an identifier for the target user in the target application that is utilized by that target user? Consider the following example. User Alice is a user of Facebook Messenger, and wishes to send a 1-1 chat message to her friend Bob. Bob is a user of a different application for messaging - Signal for example - but this fact is not known to Alice. Alice needs to somehow obtain a URI that can be used to send messages to the Signal application targeted at Bob. This is the identity mapping problem, and is addressed by the SPIN protocol defined here. 


# Implications of no Standards Action

In theory the application interoperability envisioned in the DMA could be achieved entirely through the publication of vendor-specific APIs and without standardization. However, this would yield a suboptimal outcome for both users and app developers, as supporting the matrix of pairwise communication flows between all of the affected voice, video, and messaging applications in the market via vendor-specific APIs will create a patchwork of inconsistent user experiences and likely lead to buggy implementations. Using a minimal standardized framework to bootstrap cross-app commmunications will provide more consistency while leaving app developers freedom to continue to make their own design choices. 

Furthermore, the usage of a standards-based solution ensures that end-to-end messaging, voice, and video can happen between providers. Without a standard, each vendor subject to the DMA will publish APIs for access to their services. These APIs have traditionally provided access to messages, voice and video that are not protected by e2e crypto. While it is possible, in theory, that each application provider could amend their APIs to provide access to e2e encrypted content, doing so without an agreed-upon standard will almost certainly lead to third parties decrypting in the cloud to avoid implementing N variations in each client, one for each provider they interop with. 



# Framework {#framework}


The framework for SPIN is shown in the figure below:


~~~~~~~~~~~
+---------------+               +---------------+
|               |               |               |
|Originating Svc+---------------+Terminating Svc|
|               |               |               |
+-------+-------+               +-------+-------+
        |                               |
        |                               |
        |                               |
        |                               |
+-------+-------+               +-------+-------+
|               |               |               |
|Originating App|               |Terminating App|
|               |               |               |
+-------+-------+               +-------+-------+
        |                               |
+-------+-------+    +-----+    +-------+-------+
|Originating OS +----+ SMS +----+Terminating OS |
+---------------+    +-----+    +---------------+


~~~~~~~~~~~

In the framework, we have two users - the originating and terminating. The originating user wishes to send a message, make a video call, or make a voice call, to the terminating user. A fundamental assumption of SPIN is that the originating and terminating users are both identifiable by telephone numbers on the Public Switched Telephone Network (PSTN), and that the terminating user can be reached via SMS. The originating user knows the telephone number for the terminating user. The originating user is using an app running on an operating system. The operating system can be a mobile OS, such as iOS or Android. The originating OS exposes APIs towards the application, which allow the originating app to request communication to a user with the specified number. The originating app is associated with a service running on the Internet, and can connect to it for communications services. There is a similar setup on the terminating side - the user has an application running on an operating system which can receive SMS messages, and their app is associated with a service reachable over the Internet. 

The role of the operating systems in this framework is to act as a trust anchor. The OS is responsible for authenticating the applications and vetting their behaviors, as they normally do on mobile OSs. 

The goal of the SPIN protocol is to allow a user of the originating app to select a service (voice, video or messaging), and select a phone number to which they communicate, and then receive a URI which corresponds to the terminating service which can be used to perform that communication. The URIs of course correspond to protocols for that form of communication. 

Though the framework is expressed in terms that align with mobile operating systems, the same framework can apply in other cases. For example, the terminating service, app and OS can logically be a single entity. As an example, the terminating service, app and OS could be associated with a Contact Center as a Service (CCaaS) provider. In that setup, the SMS messages are delivered directly to the CCaaS provider, and there is not a mobile operating system involved to receive them. 

# Overview of Operations

The behavior of the system is best understood through a high level sequence diagram:


~~~~~~~~~~~

+-----------+         +---------+ +-----------+ +-----+  +---------+           +-----------+ +-----------+
| orig_app  |         | orig_os | | orig_svc  | | sms |  | term_os |           | term_app  | | term_svc  |
+-----------+         +---------+ +-----------+ +-----+  +---------+           +-----------+ +-----------+
      |                    |            |          |          |                      |             |
      |                    |            |          |          |             register |             |
      |                    |            |          |          |<---------------------|             |
      |                    |            |          |          |                      |             |
      | call {number}      |            |          |          |                      |             |
      |------------------->|            |          |          |                      |             |
      |                    |            |          |          |                      |             |
      |                    | inv        |          |          |                      |             |
      |                    |---------------------->|          |                      |             |
      |                    |            |          |          |                      |             |
      |                    |            |          | inv      |                      |             |
      |                    |            |          |--------->|                      |             |
      |                    |            |          |          | -------------\       |             |
      |                    |            |          |          |-| verify sig |       |             |
      |                    |            |          |          | |------------|       |             |
      |                    |            |          |          | ---------------\     |             |
      |                    |            |          |          |-| verify hndlr |     |             |
      |                    |            |          |          | |--------------|     |             |
      |                    |            |          |          |                      |             |
      |                    |            |          | send URI |                      |             |
      |                    |<---------------------------------|                      |             |
      |                    |            |          |          |                      |             |
      |                URI |            |          |          |                      |             |
      |<-------------------|            |          |          |                      |             |
      |                    |            |          |          |                      |             |
      | req passport       |            |          |          |                      |             |
      |------------------->|            |          |          |                      |             |
      |                    |            |          |          |                      |             |
      |           passport |            |          |          |                      |             |
      |<-------------------|            |          |          |                      |             |
      |                    |            |          |          |                      |             |
      | call               |            |          |          |                      |             |
      |-------------------------------->|          |          |                      |             |
      |                    |            |          |          |                      |             |
      |                    |            | INVITE   |          |                      |             |
      |                    |            |--------------------------------------------------------->|
      |                    |            |          |          |                      |             |


      
~~~~~~~~~~~

On the terminating side, the terminating user at some point installs an application which is capable of handling communications for one or more media types (voice, video or messaging). The application will register with the terminating OS, using APIs exposed in the OS, that it is capable of acting as a SPIN handler. As part of the registration, the application provides the OS with a URI for the service it provides of that media type. As discussed below, this can be a proprietary API, or can be a baseline standard protocol. In the case of voice, that baseline standard is SIP, and in particular, cloud SIP {{I-D.rosenberg-dispatch-cloudsip}}. 

Later on, a user in an originating application decides to place a call to a number. The originating application does not have a user with that number as part of its own service, so it knows it needs to use SPIN to route the call. It goes to the operating system on the mobile phone, and requests it to provide a URI for voice communications to the specified phone number. The originating OS then prepares an invitation object. This is a JWT which contains several fields. THe fields include the phone number of the originating user (which must be known and verified by the mobile OS), and an HTTP URI that can be used by the terminating OS to send the results back, and the communications service that is requested. This HTTP URI will normally contain an embedded Authorization header field that contains a short-lived token, valid to send the results back. It then signs the JWT and sends an SMS (more likely, an MMS given the size of the signed object), to the target user's phone number. The terminating OS receives the SMS/MMS, and notices that it contains an invitation object, and thus should not be rendered to the user. Should the terminating user and its OS not support this protocol, it will end up rendering the MMS. The MMS includes some plain text, which can be rendered to the user, indicating that the caller wishes to speak with them, so that the human user can take some action (like a return voice call over the PSTN). 

Assuming the terminating OS supports this protocol, the MMS is absorbed and decoded. THe signature is verified and then the communications service is obtained. In this example use case, it is for a voice call. The terminating OS has an application that has registered itself as a handler for voice. Note that, the terminating user might have multiple applications on their OS which can act as handlers for voice. In such a case, the mobile OS would offer the user a configuration setting to choose one as a default. 

The app had previously registered itself as a handler and provided a SIP URI for the receipt of calls, something like sip:{number}@provider.com. This URI is sent back to the originating OS. Rather than sending this back via SMS/MMS, IP communications are used. The invitation object contained an HTTP URI which can be used by the terminating OS to send the SIP URI. The SPIN protocol defines the exact syntax and semantics of this HTTP POST operation. This is received by the originating OS, which then informs the app that it was able to locate the user. The originating OS provides the communications URI (in this case, a SIP URI for voice calls). 

Next - the originating app places a SIP call. Because we are now dealing with inter-domain and inter-provider calls, secure caller ID is required. SPIN requires that STIR passports {{RFC8225}} are included, sent using {{RFC8224}}. The originating OS is required to obtain a passport that is valid for the originating user. The means by which a mobile OS can generate passports it outside the scope of this specification. The originating app makes an API call into the OS to obtain the passport, which is then returned to the app. The app uses its own app-specific protocols to communicate with its servers, and will send the passport and the terminating user's phone number to its service. Its service will then send a SIP INVITE to the target number, including the passport in the SIP Identity header field. From there, the terminating service can alert its app using the mobile OS push techniques, and a call has been placed.

The SPIN framework therefore consists of the following:

1. A standardized syntax for an invitation object that can be sent via MMS
2. A standardized HTTP-based protocol for providing URIs for communications
3. Recommendations for mobile OS vendors on APIs they should provide to enable SPIN, without actually specifying any details of what those APIs look like
4. Requirements for communications protocols needed for voice, video and messaging between app providers


# Invitation Object Syntax

To be filled in.

# Protocol for Providing URIs

To be filled in

# Mobile OS vendor API Recommendations

To be filled in

# Requirements for Communications Protocols

There are several ways in which the communications protocols could be specified. On one extreme, the standard could leave this entirely up to the terminating provider to define its protocol or API and document it publically. It would then be the responsibility of the originating service to implement each of these APIs for every terminating provider it wishes to speak to. On the other extreme, we can fully specify a protocol - most likely with reference to existing standards. 

SPIN tries to take a middle ground. It allows terminating providers to choose whether their interface is proprietary, or, whether it follows a minimum baseline protocol specified here. 

Because the communications are between providers that may not have previously had an established bilateral relationship, we want the communications to be possible without any kind of manual configuration. For this reason, SPIN specifies that the default voice and video communications protocol is cloud SIP {{I-D.rosenberg-dispatch-cloudsip}}, and it utilizes the media protocols standardized by webRTC. The usage of cloud SIP allows scalable, reliable, inter-provider SIP over the Internet, and the usage of the webRTC media stack provides a well-defined baseline media stack that is already widely implemented. The SIP messaging MUST utilize {{RFC8224}} to ensure secure user identity. Media between the originating and terminating service will be DTLS-SRTP by virtue of using webRTC, and e2e media encryption is supported and bootstrapped using a certificate bound to the user's phone numbers. 

Details to be filled out. 

For messaging, SPIN will mandate MLS {{I-D.ietf-mls-architecture}}. Work is needed to determine how to handle authentication and MLS federation. This will be accompanied by a lightweight protocol for encapsulating the MLS messages, as well as application functions for sending actual messages, message contents, delivery and read notifications, and so on.

Obviously a LOT of details to be added.

# Security Considerations

The SPIN protocol defined here is meant to address the following threats:

* A malicious application that "steals" incoming calls or chats against user wishes. To prevent this, this protocol enlists the mobile operating system as a trusted third party that governs dispatch of communication requests to the right application based on user preferences. 

* A malicious application that spams target users with requests for communication. This is mitigated by enlisting the aid of the mobile operating system on the terminating side to absorb SMS's conforming to this specification, and not presenting them to the user. Digital signatures are used over the content of the SMS messages, and the terminating OS can validate that it trusts the sender before taking further action. 

* Intermediates that eavesdrop on communications between app providers. This is mitigated by using e2e encryption across messaging, voice and video, ensuring it can be retained when crossing provider boundaries. 



