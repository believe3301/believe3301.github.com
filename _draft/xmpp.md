<iq id='1' type='get'>
    [ ... payload ... ]
</iq>

info/query [request|response]

id:client generate,and response iq with the same id

type: get|set|result|error 

get|set ->request
result|error ->response


roster [friend list]
1> use <iq/> stanzas, <query /> child element. namespace: jabber:iq:roster
2> can contains multi item for get/set and result
3> <query /> ver attribute, generate by server
4> <item /> approved attribute, pre-approved subscription by server
5> <item /> ask attribute, sub-status by server
6> <item /> jid attribute, must contains by server or client [PK]
7> <item /> name attribute, handle with jid
8> <item /> subscription attribute,state of the presence, none|to|from|both 
9> <item /> <group/> category


managing presence
1> the entity has a subscription to user presence, call contact 
2> presence stanzas : subscribe, unsubscribe, subscribed, unsubscribed
3> 
request:
UC:client send outbound subscription, to attribute is bare jid
US:stamp from attribute, and delivered local or remotely, and roster push with ask
CS:may auto-reply
CC:send subscried or unsubscribed request

approved:
need update roster with subscription='from' to all available.

cancel:

    sub-states 

exchange presence
-------------------------
1 publish presend by sending inital presence, terminated by sending unavailable presence
2 broadcast without to, and type is none or unavailable.
3 directed presence ?? (not subsribed but send presence)
4 process probe (get contact presence, send by server)
5 presence priorities
6 presence state


xmpp
1> support mult-device login
add roster
    1> one connected resource send roster set
    2> server to sended resource roster result
    3> server to all available resource roster push
remove roster
    1> subscription value is remove
    2> send unsubscribe or unsubscribed to contact
2>
