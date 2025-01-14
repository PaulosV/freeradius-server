= Extensible Authentication Protocol (EAP)

## Introduction

Extensible Authentication Protocol(EAP), rfc2284, is a general protocol
that allows network access points to support multiple authentication
methods. Each EAP-Type indicates a specific authentication mechanism.
802.1x standard authenticates wireless LAN users trying to access
enterprise networks.

RADIUS attribute used for EAP is EAP-Message, 79(rfc2869). RADIUS
communicates all EAP messages by embedding them in this attribute.

General Terminology
Supplicant/EAP Client - is the software on the end-user/client machine
                        (machine with the wireless card).
Authenticator/NAS/Access Point(AP) -  A network device providing users
                                  with a point of entry into the network.
EAPOL - EAP over LAN as defined in 802.1x standard.
EAPOW - EAP over Wireless.

```
   +----------+        +----------+        +----------+
   |          |  EAPOL |          | RADIUS |          |
   |  EAP     |<------>|  Access  |<------>|  RADIUS  |
   |  Client  |  EAPOW |  Point   |  (EAP) |  Server  |
   |          |        |          |        |          |
   +----------+        +----------+        +----------+
```

The sequence of events, for EAP-MD5, runs as follows:

1. The end-user associates with the Access Point(AP).
2. The supplicant specifies AP to use EAP by sending EAP-Start.
3. AP requests the supplicant to Identify itself (EAP-Identity).
4. Supplicant then sends its Identity (username) to the AP.
5. AP forwards this EAP-response AS-IS to the RADIUS server.
   (The supplicant and the RADIUS server mutually authenticate via AP.
   AP just acts as a passthru till authentication is finished.)
6. The server sends a challenge to the supplicant.
7. The supplicant carries out a hash on the password and sends
   this hashed password to the RADIUS server as its response.
8. The RADIUS server performs a hash on the password for that supplicant
   in its user database and compares the two hashed values and
   authenticates the client if the two values match(EAP-Success/EAP-Failure)
9. AP now opens a port to accept data from the end-user.

Currently, EAP is widely used in wireless networks than in wired networks.
In 802.11/wireless based networking, following sequence of events happen in
addition to the above EAP events.

10. RADIUS server and the supplicant agree to a specific WEP key.
11. The supplicant loads the key ready for logging on.
12. The RADIUS server sends the key for this session (Session key) to the AP.
13. The AP encrypts its Broadcast key with the Session key
14. The AP sends the encypted key to the supplicant
15. The supplicant decrypts the Broadcast key with the Session key and
    the session continues using the Broadcast and Session keys until
    the session ends.

References:

The Implementation of EAP over RADIUS is based on the following RFCs

* `rfc2869` -- RADIUS Extensions
* `rfc2284` -- PPP Extensible Authentication Protocol (EAP)
* `rfc2716` -- PPP EAP TLS Authentication Protocol

Following links help to understand HOW EAP works
http://www.ieee802.org/1/mirror/8021/docs2000/ieee_plenary.PDF

## EAP Code Organization

EAP is implemented as a module in freeradius and the code is placed
in src/modules/rlm_eap.
All EAP-Types are organized as subdirectories in rlm_eap/types/.

Each EAP-Type, like types/rlm_eap_md5, contains a chunk of code that
knows how to deal with a particular kind of authentication mechanism.

To add a new EAP-Type then a new directory should be created as
rlm_eap/types/rlm_eap_XXXX, where XXXX is EAP-Type name
ie for EAP-Type like ONE TIME PASSWORD (OTP) it would be rlm_eap_otp

Path                         | Description
-----------------------------|------------------------------------
`src/modules/rlm_eap`        | Contains the basic EAP and generalized interfaces \
                               to all the EAP-Types.
`rlm_eap/types`              | Contains all the supported EAP-Types
`rlm_eap/types/rlm_eap_md5`  | EAP-MD5 authentication.
`rlm_eap/types/rlm_eap_tls`  | EAP-TLS based authentication.
`rlm_eap/types/rlm_eap_ttls` | TTLS based authentication.
`rlm_eap/types/rlm_eap_peap` | Windows PEAP based authentication.
`rlm_eap/types/rlm_eap_leap` | Cisco LEAP authentication.
`rlm_eap/types/rlm_eap_sim`  | EAP-SIM (GSM) based authentication

## Configuration

Add the eap configuration stanza to the modules section in `radiusd.conf`
to load and control rlm_eap and all the supported EAP-Types:

  For example:

```unlang
modules {
	...
	eap {
		default_eap_type = md5

		md5 {
		}
	    ...
	}
	...
}
```

NOTE: You cannot have empty eap stanza. At least one EAP-Type sub-stanza
should be defined as above, otherwise the server will not know what type
of eap authentication mechanism to be used and the server will exit
with error.

All the various options and their associated default values for each
`EAP-Type` are documented in the sample radiusd.conf that is provided
with the distribution.

Since the EAP requests may not contain a requested EAP type, the
`default_eap_type` configuration options is used by the EAP module
to determine which EAP type to choose for authentication.

NOTE: EAP cannot authorize a user. It can only authenticate.
Other Freeradius modules authorize the user.


## EAP SIM server

To configure EAP-SIM authentication, the following attributes must be
set in the server. This can be done in the users file, but in many cases
will be taken from a database server, via one of the SQL interface.

If one has SIM cards that one controls (i.e. whose share secret you know),
one should be able to write a module to generate these attributes
(the triplets) in the server.

If one has access to the SS7 based settlement network, then a module to
fetch appropriate triplets could be written. This module would act as
an authorization only module.

The attributes are:

Attribute     | Size
--------------|-----------
EAP-Sim-Rand1 |		16 bytes
EAP-Sim-SRES1 |		 4 bytes
EAP-Sim-KC1 	|	   8 bytes
EAP-Sim-Rand2 |		16 bytes
EAP-Sim-SRES2 |		 4 bytes
EAP-Sim-KC2 	|	   8 bytes
EAP-Sim-Rand3 |		16 bytes
EAP-Sim-SRES3 |		 4 bytes
EAP-Sim-KC3 	| 	 8 bytes

NOTE: `EAP-SIM` will send WEP attributes to the resquestor.

## EAP Clients

1. XSupplicant - freeradius (EAP/TLS) notes may be found at:

http://www.eax.com/802/
or http://www.missl.cs.umd.edu/wireless/eaptls/

XSupplicant is hosted by:

http://www.open1x.org/

2. XP - freeradius (EAP/TLS) notes may be found at:

http://www.denobula.com/EAPTLS.pdf

## Testing

You will find several test cases in src/tests/ for the EAP-SIM code.

## FAQ & Examples

How do i use it?

1. How can I enable EAP-MD5 authentication ?

In radiusd.conf

``` 
  modules {
  	...
  	eap {
  		default_eap_type = md5
  		md5 {
  		}
  	    ...
  	}
  	...
  }

  # eap sets the authenticate type as EAP
  recv Access-Request {
  	...
  	eap
  }

  # eap authentication takes place.
  process Access-Request {
  	eap
  }

  # If you are proxying EAP-LEAP requests
  # This is required to make LEAP work.
        post-proxy {
  	eap
  }
```

2. My Userbase is in LDAP and I want to use EAP-MD5 authentication

In radiusd.conf

```
  modules {
  	...
  	eap {
  		default_eap_type = md5
  		md5 {
  		}
  	    ...
  	}
  	...
  }

  # ldap gets the Configured password.
  # eap sets the authenticate type as EAP
  recv Access-Request {
  	...
  	ldap
  	eap
  	...
  }

  # eap authentication takes place.
  process Access-Request {
  	...
  	eap
  	...
  }
```

3. How can I Proxy EAP messages, with/without User-Name attribute
in the `Access-Request` packets

With `User-Name` attribute in `Access-Request` packet,
`EAP-proxying` is just same as RADIUS-proxying.

If `User-Name` attribute is not present in `Access-Request` packet,
Freeradius can proxy the request with the following configuration
in radiusd.conf

```
  #  eap module should be configured as the First module in
  #  the authorize stanza

  recv Access-Request {
  	eap
  	...  other modules.
  }
```

With this configuration, eap_authorize creates `User-Name` attribute
from `EAP-Identity` response, if it is not present.
Once `User-Name` attribute is created, RADIUS proxying takes care
of EAP proxying.

4. How Freeradius can handle `EAP-START` messages ?

In most of the cases this is handled by the Authenticator.

Only if it is required then, in radiusd.conf

```
recv Access-Request {
	eap
	...  other modules.
}
```

With the above configuration, RADIUS server immediately responds with
EAP-Identity request.

NOTE: EAP does not check for any Identity or maintains any state in case
of EAP-START. It blindly responds with EAP-Identity request.
Proxying is handled only after EAP-Identity response is received.

5. I want to enable multiple EAP-Types, how can I configure ?

In radiusd.conf

```
modules {
	...
	eap {
		default_eap_type = tls
		md5 {
		}
		tls {
			...
		}
	    ...
	}
	...
}
```

The above configuration will let the server load all the EAP-Types,
but the server can have only one default EAP-Type, as above.

Once EAP-Identity response is received by the server, based on the
default_eap_type, the server will send a new request (MD5-Challenge
request incase of md5, TLS-START request incase of tls) to the supplicant.
If the supplicant is rfc2284 compliant and does not support the
EAP-Type sent by the server then it sends EAP-Acknowledge with the
supported EAP-Type. If this EAP-Type is supported by the server then it
will send the respective EAP-request.

Example: If the supplicant supports only EAP-MD5 but the server
default_eap_type is configured as EAP-TLS, as above, then the server
will send TLS-START after EAP-Identity is received. Supplicant will
respond with EAP-Acknowledge(EAP-MD5). Server now responds with
MD5-Challenge.

## Installation

EAP, EAP-MD5, and Cisco LEAP do not require any additional packages.
Freeradius contains all the required packages.

For EAP-TLS, EAP-TTLS, and PEAP, OPENSSL, <http://www.openssl.org/>,
is required to be installed.
Any version from 0.9.7, should fairly work with this module.

EAP-SIM should not require any additional packages.

## Implementation (For Developers)

The rlm_eap module only deals with EAP specific authentication mechanism
and the generic interface to interact with all the EAP-Types.

Currently, these are the existing interfaces,

```
int	attach(CONF_SECTION *conf, void **type_arg);
int	initiate(void *type_arg, EAP_HANDLER *handler);
int	authenticate(void *type_arg, EAP_HANDLER *handler);
int	detach(void **type_arg);
```

`attach()` and `detach()` functions allocate and deallocate all the
required resources.

`initiate()` function begins the conversation when EAP-Identity response
is received. Incase of EAP-MD5, `initiate()` function sends the challenge.

`authenticate()` function uses specific EAP-Type authentication mechanism
to authenticate the user. During authentication many EAP-Requests and
EAP-Responses takes place for each authentication. Hence authenticate()
function may be called many times. EAP_HANDLER contains the complete
state information required.

## How EAP works

as posted to the list, by John Lindsay <jlindsay@internode.com.au>

To make it clear for everyone, the supplicant is the software on the
client (machine with the wireless card).

The EAP process doesn't start until the client has associated with
the Access Point using Open authentication.  If this process isn't
crystal clear you need to go away and gain understanding.

Once the association is made the AP blocks all traffic that is not
802.1x so although associated the connection only has value for EAP.
Any EAP traffic is passed to the radius server and any radius traffic
is passed back to the client.

So, after the client has associated to the Access Point, the
supplicant starts the process for using EAP over LAN by asking the
user for their logon and password.

Using 802.1x and EAP the supplicant sends the username and a one-way
hash of the password to the AP.

The AP encapsulates the request and sends it to the RADIUS server.

The radius server needs a plaintext password so that it can perform
the same one-way hash to determine that the password is correct.  If
it is, the radius server issues an access challenge which goes back
via to the AP to the client. (my study guide says client but my
brain says 'supplicant')

The client sends the EAP response to the challenge via the AP to the
RADIUS server.

If the response is valid the RADIUS server sends a success message
and the session WEP key (EAP over wireless) to the client via the
AP.  The same session WEP key is also sent to the AP in the success
packet.

The client and the AP then begin using session WEP keys. The WEP key
used for multicasts is then sent from the AP to the client.  It is
encrypted using the session WEP key.

## ACKNOWLEDGEMENTS

* Primary author - Raghu <raghud@mail.com>
* EAP-SIM	 - Michael Richardson <mcr@sandelman.ottawa.on.ca>
* The development of the EAP/SIM support was funded by
  Internet Foundation Austria (http://www.nic.at/ipa).
