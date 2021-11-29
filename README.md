# Cisco_CUBE_SIP_to_Telco
Cisco CUBE SIP to Telco


## Problem

When Telco provider is routing trafic based on SIP To: field, not based on SIP URI field, cisco gateway get only pilot number and can't route call to real destination.

SIP INVITE sample:
```
INVITE sip:381113333400@10.0.0.1:5060 SIP/2.0
Via: SIP/2.0/UDP 10.10.10.10:5060;branch=dfskjgfghjgyufuygf.1
To: "381113333488 381113333488"<sip:381113333488@ims.telco-example-domain.com>;cscf
From: <sip:0117778888@sip1.telco-example-domain.com;user=phone>;tag=45345345345-345345345-
Call-ID: XADDFAFFDSFSDFSDF-1233423457896@10.100.100.100
CSeq: 123462010 INVITE
Max-Forwards: 28
Content-Length: 249
Contact: <sip:0117778888@10.10.10.10:5060;transport=udp>
Content-Type: application/sdp
Call-Info: <sip:10.100.100.100>;appearance-index=1
Allow: ACK, BYE, CANCEL, INFO, INVITE, OPTIONS, PRACK, REFER, NOTIFY, UPDATE
Accept: application/media_control+xml
Accept: application/sdp
Accept: application/x-hotsip-filetransfer+xml
Accept: multipart/mixed
Supported: 100rel, timer
P-Asserted-Identity: <sip:+381113218135@sip1.telco-example-domain.com;user=phone>
Privacy: none
P-Charging-Vector: icid-value=sdfdsgfhfjrty6465785685675fdegf
Min-SE: 900
Session-Expires: 1800
P-Charging-Function-Addresses: ccf="aaa://sip1.ims.telco-example-domain.com:3866;transport=tcp"
P-Called-Party-ID: <sip:381113333400@ims.telco-example-domain.com>
Recv-Info: x-broadworks-client-session-info

v=0
o=BroadWorks 2104998929 1 IN IP4 10.10.10.10
s=-
c=IN IP4 10.10.10.10
t=0 0
a=sendrecv
m=audio 34098 RTP/AVP 8 18 101
a=rtpmap:8 PCMA/8000
a=rtpmap:18 G729/8000
a=fmtp:18 annexb=no
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-15

```

In sample above SIP RquestURI is __"INVITE sip:381113333400@10.0.0.1:5060 SIP/2.0"__ and To filed is __"To: "381113333488 381113333488"<sip:381113333488@ims.telco-example-domain.com>;cscf"__, while From filed is __"From: <sip:0117778888@sip1.telco-example-domain.com;user=phone>;tag=45345345345-345345345-"__.


## Solution 1

Based on this example it is needed to get To filed and copy it to RequestURI. On newer IOS version it can be done using sip profiles and on older version TCL script is needed (Solution 2).


```
voice service voip
 ip address trusted list
  ipv4 10.10.10.10 255.255.255.255
  ipv4 10.200.200.200 255.255.255.255
 rtp-port range 8000 48198
 ! PORT RANGE - JUST TO BE SURE, FOR FIREWALL CONFIGURATION
 allow-connections sip to sip
 no supplementary-service sip moved-temporarily
 no supplementary-service sip refer
 fax protocol t38 nse version 0 ls-redundancy 0 hs-redundancy 0 fallback pass-through g711ulaw
 modem passthrough nse codec g711ulaw
 trace
  shutdown
  ! RECOMENDED TO BE SHUTDOWN
  ! NO SHUT DURING SETUP/DEBUG PROCESS
 sip
  header-passing
  registrar server expires max 600 min 60
  early-offer forced
  midcall-signaling passthru
  sip-profiles inbound
  ! INCOMING SIP PROFILE IMPORTANT FOR APPLYING IT ON INCOMNG DIAL-PEER 

voice class codec 1
 codec preference 1 g711alaw
 codec preference 2 g711ulaw
 codec preference 3 g729r8

voice class sip-profiles 100
 rule 1 request ANY sip-header To copy "sip:(.*)@" u01
 rule 2 request ANY sip-header SIP-Req-URI modify "(.*) sip:.*@(.*)" "\1 sip:\u01@\2"
!
!
interface GigabitEthernet0/0/0
 bandwidth 100000
 ip address 10.0.0.1 255.255.255.252
 no ip redirects
 no ip unreachables
 no ip proxy-arp
 ip access-group acl_TELCO_SIP_IN in
 ip access-group acl_TELCO_SIP_OUT out
! HERE USING ZONE BASED FIREWALL IS RECOMENDED, BUT IT IS GOOD ENOUGHT

ip route 10.10.10.10 255.255.255.255 10.0.0.2
! route to telco sip server
!
ip host ims.telco-example-domain.com 10.10.10.10
!
!
ip access-list extended acl_TELEKOM_SIP_IN
 10 permit icmp any any echo-reply
 20 permit tcp any host 10.0.0.1 established
 30 permit udp host 10.10.10.10 host 10.0.0.1 eq 5060
 40 permit udp host 10.10.10.10 host 10.0.0.1 range 16384 32767
 50 permit udp host 10.10.10.10 host 10.0.0.1 range 8000 48198
 60 deny   ip any any log
ip access-list extended acl_TELEKOM_SIP_OUT
 10 permit ip host 10.0.0.1 any
 20 deny   ip any any log

!
dial-peer voice 10 voip
 ! INCOMING PEER THAT IS MATCHING BY DEFAULT (JUST IN CASE DIALPEER), 
 ! COR LIST ARE MAKING SHURE THAT NO OUTGOING DP IS MATCHED WHEN THIS DP IS HIT
 corlist incoming DEFAULT-KEY
 corlist outgoing DEFAULT-LOCK
 description DEFAULT INCOMING DIAL PEER
 huntstop
 session protocol sipv2
 incoming called-number .
 voice-class codec 1
 dtmf-relay sip-notify sip-kpml rtp-nte
 no vad
!

voice translation-rule 80
 rule 1 /3811133334(..)/ /10\1/
!
voice translation-rule 90
 rule 1 /^38111/ /9/
 rule 2 /^381/ /90/
 rule 3 /^/ /9/
!

voice translation-profile _INCOMING_TELCO_SIP_TEST_
 translate calling 90
 translate called 80


dial-peer voice 20 voip
 ! INCOMING DIAL PEER FROM TELCO PROVIDER
 corlist incoming TELCO-KEYS
 corlist outgoing DEFAULT-LOCK
 description INCOMING FROM TELCO SIP
 translation-profile incoming _INCOMING_TELCO_SIP_TEST_
 huntstop
 session protocol sipv2
 incoming called-number ^3811133334..$
 voice-class codec 1
 voice-class sip profiles 100 inbound
 voice-class sip bind control source-interface GigabitEthernet0/0/0
 voice-class sip bind media source-interface GigabitEthernet0/0/0
 ! bind all to interface facing telco provider
 dtmf-relay sip-notify sip-kpml rtp-nte
 no vad
!

dial-peer voice 100 voip
 ! OUTGOING DIAL-PEER TO OUR SIP SERVER (Cisco Unified Communication Manager)
 corlist incoming DEFAULT-KEY
 corlist outgoing CUCM-LOCK
 huntstop
 preference 1
 destination-pattern 10..$
 session protocol sipv2
 session target ipv4:10.200.200.200 
 voice-class codec 1
 voice-class sip bind control source-interface GigabitEthernet0/0/1
 voice-class sip bind media source-interface GigabitEthernet0/0/1
 ! Bind all to inside interface
 dtmf-relay sip-notify sip-kpml rtp-nte
 no vad
!

dial-peer voice 30 voip
 ! INCOMING DIAL-PEER FROM OUR SIP SERVER (Cisco Unified Communication Manager)
 corlist incoming CUCM-KEYS
 corlist outgoing DEFAULT-LOCK
 description INCOMING CUCM
 huntstop
 session protocol sipv2
 incoming called-number ^9T
 voice-class codec 1
 voice-class sip bind control source-interface GigabitEthernet0/0/1
 voice-class sip bind media source-interface GigabitEthernet0/0/1
 dtmf-relay sip-notify sip-kpml rtp-nte
 no vad
!

voice translation-rule 71
 rule 1 /^9/ //
!
voice translation-rule 72
 rule 1 /10\(..\)/ /3811133334\1/
 rule 2 /^.*/ /381113333400/
 ! RULE 2 DEFAULT RULE (JUST IN CASE)

voice translation-profile _OUTGOING_TELEKOM_SIP_TEST_
 translate calling 72
 translate called 71

dial-peer voice 200 voip
 ! OUTGOING DIAL-PEER TO TELCO SIP SERVER
 corlist incoming DEFAULT-KEY
 corlist outgoing TELEKOM-LOCK
 translation-profile outgoing _OUTGOING_TELEKOM_SIP_TEST_
 huntstop
 destination-pattern 9T
 session protocol sipv2
 session target ipv4:10.10.10.10
 voice-class codec 1
 voice-class sip bind control source-interface GigabitEthernet0/0/0
 voice-class sip bind media source-interface GigabitEthernet0/0/0
 ! bind all to interface facing telco provider
 dtmf-relay sip-notify sip-kpml rtp-nte
 no vad
!
! COR LIST ARE HERE TO MAKE SURE THAT INCOMING LEG OVER TELCO PROVIDER
! CAN ONLY BE ROUTED TO OUR LOCAL SIP SERVER AND VICE VERSA
gateway
 timer receive-rtp 1200
!
sip-ua
 credentials number 381113333400 username 381113333400@ims.telco-example-domain.com password SOME_PASSWORD realm ims.telco-example-domain.com
 retry invite 2
 registrar dns:ims.telco-example-domain.com expires 3600
 sip-server ipv4:10.10.10.10
 host-registrar
!

```

## Solution 2

In case of an older IOS we need to use TCL script

IOS SAMPLE:
```
application
 service telco flash:telco.tcl
!
! 
dial-peer voice 20 voip
 ! INCOMING DIAL PEER FROM TELCO PROVIDER
 service telco
 corlist incoming TELCO-KEYS
 corlist outgoing DEFAULT-LOCK
 description INCOMING FROM TELCO SIP
 translation-profile incoming _INCOMING_TELCO_SIP_TEST_
 huntstop
 session protocol sipv2
 incoming called-number ^3811133334..$
 voice-class codec 1
 voice-class sip profiles 100 inbound
 voice-class sip bind control source-interface GigabitEthernet0/0/0
 voice-class sip bind media source-interface GigabitEthernet0/0/0
 ! bind all to interface facing telco provider
 dtmf-relay sip-notify sip-kpml rtp-nte
 no vad

```

TCL Script:
```tcl
 proc setup { } {

leg proceeding leg_incoming

set To [infotag get leg_proto_headers "To"]
set numero1 $To

regexp {sip:([0-9]+)@} $To w numero1

set numero $numero1
regexp {tel:\+([0-9]+)} $numero1 w numero


if { [regexp {381113333417} $numero ]} { set numero "1017"}
# HERE WE ARE USING TCL TO TRANSFORM NUMBER, IT CAN ALSO BE REGEX "3811133334(..)" to-> "10\1"
# IT CAN NOT BE DONE USING VOICE TRANSLATION PROFILES. Well actualy it can, but we need to create new dial-peer
# matching new (translated) number

puts "\n >>>>> MY-TCL-SCRIPT: To = $To ; numero = $numero \n"
leg setup $numero callInfo leg_incoming
}


proc setup_done { } {
#
#       Handle SETUP DONE.
#
}


proc cleanup { } {
   call close
}


proc disco { } {

set status [infotag get evt_last_disconnect_cause]
puts "disco with status $status"

        leg disconnect leg_all
        #call close
}


proc act_SetupDone { } {


set status [infotag get evt_status]
puts "act_SetupDone with status $status"
    if { $status == "ls_007" } {
          leg disconnect leg_all -c 17
    } else {
          if { $status == "ls_009" } {
                leg disconnect leg_all -c 18
          } else {
                if { $status != "ls_000" } {
                  leg disconnect leg_all -c 29
                }
          }
        }
}

requiredversion 2.0


#----------------------------------
# State Machine
#----------------------------------

set fsm(CALL_INIT,ev_setup_indication) "setup GETDEST"
set fsm(any_state,ev_disconnected) "disco same_state"
set fsm(CALLDISCONNECT,ev_disconnect_done) "cleanup same_state"
set fsm(any_state,ev_setup_done)                      "act_SetupDone          same_state"

fsm define fsm CALL_INIT
```

That is it folks.
