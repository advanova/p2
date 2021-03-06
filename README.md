# Conversations Push Proxy
An [XEP-0357: Push Notifications](https://xmpp.org/extensions/xep-0357.html) app server that relays push messages between the user’s server and Googles Firebase Cloud Messaging.

## Background
Due to restrictions in Firebase Cloud Messaging (and most other push services), only the developer can create push notifications for their apps. For this reason a user’s server wouldn’t be able to wake up a user’s device directly but has to proxy that wake up signal through the infrastructure of the app developer.

Here is a quick description of how this relationship is set up.

*All examples below use real world values from my personal device with a test account. The fact that I’m comfortable making that publicly available can act as an indication on how non privacy sensitive that data is.*

### What Conversations sends to the app server

* FCM Token: The FCM token is essentially the address of the device that will be used to address push notifications to. That address can change over time. The address is generated by the Google Play Service on the user’s phone. Conversations has no influence over how and when this is (re)generated and can only request it.

* Secure Device Id: Since the FCM token can change over time the app server needs a second unique ID per device so when it receives a new FCM token it knows whether this one replaces an existing device or is for an entirely new device. Conversations uses the [Android ID](https://developer.android.com/reference/android/provider/Settings.Secure.html#ANDROID_ID) which is a unique string that is generated by the Android operating system and persists over the life time of the app. In older versions of Android this used to be shared across all apps. Nowadays every app gets its own ID generated by the OS.

* The app server then generates a random string (Node ID in the language of the XEP) and stores a combination of the FCM Token, the node id, the domain of the account (not the entire account) and a hashed string of the account jid + the secure device id. Since the account jid is essentially *salted* with the secure device id the app server operator won’t be able to reverse the account jids by looking at the database.
The node id is sent back to the client.

```xml
<iq from="xiaomia1@jabber.de/Conversations.sAdA" id="cXo5bNCF6wgD" to="p2.siacs.eu" type="set">
  <command xmlns="http://jabber.org/protocol/commands" action="execute" node="register-push-fcm">
    <x xmlns="jabber:x:data" type="submit">
      <field var="token">
        <value>eeDXSJjASJY:APA91bEKxhXK54-vHhY9O55JmU2R0nDJL2rRENm-W9uPY6x3jHi0i0OyvPu6js9jVPZqDeX9ZQydZCBZE19o7a0kK4_n88fCgufXjaOlvalh9VibB2zOI7dQRTaDNB3H5s4dicpWD0m4</value>
      </field>
      <field var="android-id">
        <value>92afd7a91cdba9a0</value>
      </field>
    </x>
  </command>
</iq>
```

```xml
<iq from="p2.siacs.eu" id="cXo5bNCF6wgD" to="xiaomia1@jabber.de/Conversations.sAdA" type="result">
  <command xmlns="http://jabber.org/protocol/commands" action="complete" node="register-push-fcm" sessionid="1526463999190">
    <x xmlns="jabber:x:data" type="form">
      <field type="jid-single" var="jid">
        <value>p2.siacs.eu</value>
      </field>
      <field type="text-single" var="node">
        <value>eKxTS5n3bOe0</value>
      </field>
      <field type="text-single" var="secret">
        <value>KK4+H5WMfpRG/2zCbuKq6wpX</value>
      </field>
    </x>
  </command>
</iq>
```

### What Conversations sends to the user’s server

After registering with the App server, Conversations sends the node ID and the jid of the app server (p2.siacs.eu) to the user's server.

```xml
<iq type='set' id='x42'>
  <enable xmlns='urn:xmpp:push:0' jid='p2.siacs.eu' node='eKxTS5n3bOe0' />
</iq>
```

### What the app server stores

```
device                                   | domain    | token                                                                                                                                                    | node         | secret                
ec164939a8485ee6b7f7871071a11c7bb18aead5 | jabber.de | eeDXSJjASJY:APA91bEKxhXK54-vHhY9O55JmU2R0nDJL2rRENm-W9uPY6x3jHi0i0OyvPu6js9jVPZqDeX9ZQydZCBZE19o7a0kK4_n88fCgufXjaOlvalh9VibB2zOI7dQRTaDNB3H5s4dicpWD0m4 | eKxTS5n3bOe0 | KK4+H5WMfpRG/2zCbuKq6wpX
```

### What the user’s server sends to the app server

When a push is required (this is determined with internal logic of the user’s server that is out of scope of this document) the user’s server will send the node id to the app server. The user’s server *can* also add additonal information like number of messages pending, the sender jid of the last message and even the body of the last message. Whether or not this information is included is up to the implementation and the configuration of the user’s server and is out of scope for this document as well. Whether or not the app server receives that additional information it will just ignore it and not process it.
The sender jid for the push is the the jid of the server. Since the app server didn’t store the account jid and since the account jid is not included in the push it won’t know for which account the push is actually for. It will just be able to look up the corresponding FCM token based on the node id.

An example of that communication can be found in [XEP-0357 Section 7](https://xmpp.org/extensions/xep-0357.html#publishing).

### What the app server sends via Google to the user’s device

Upon receiving a push request from a XMPP server, the app server looks up the hashed account jid + secure device id combination and the FCM token. It then sends the hash via Google to the user's device, using the token (remember: 'token' means 'device address' in FCM language).

The hash can then be used by Conversations to determine which of a number of accounts should be woken up. The hash is not reversible but since Conversations has a limited number of accounts and also knows the secure device id it can just calculate all hashes. Google only sees that hash and nothing else.

```json
{"to":"fcm-token","data":{"account":"ec164939a8485ee6b7f7871071a11c7bb18aead5"}}
```

### Further optimizations

Currently, when receiving a push from a Jabber server, the app server takes the domain name and the node id to look up a push token. If the node id is unique enough, we don’t actually need the domain as a second identifier. However since push in general is still somewhat experimental, I would like to keep the domain information for now to be able to properly debug the app server when something goes wrong and narrow down potential proplems to a specific server.

The domain will be removed from the database once everything goes smoothly.

## Manual Installation

### Build from source
```
git clone https://github.com/iNPUTmice/p2.git
cd p2
mvn package
```

### Manual Install
```
cp target/p2-0.3.jar /opt
mkdir /etc/p2
cp p2.conf.example /etc/p2/config.json
mkdir /var/lib/p2
useradd -d /var/lib/p2 -r p2
chown -R p2:p2 /var/lib/p2
cp dist/p2.service /etc/systemd/system
systemctl daemon-reload
systemctl enable p2.service
systemctl start p2.service
```

There is currently no way to reload the configuration file at runtime but you can always restart the service with `systemctl restart p2.service`.

### Database

Since version 0.3 the Conversations Push Proxy requires a database (drivers for MariaDB are included by default but you can easily change `pom.xml` to include other drivers as well.)

The necessary tables will be created automatically on first use so you just have to create a database and a user. You can do this by starting `mysql` and typing:
```sql
create user 'p2'@'localhost' identified by 'secret';
grant all privileges on p2.* to 'p2'@'localhost';
```


## Installation using distro packages

### Arch linux

1. Install [p2-git](https://aur.archlinux.org/packages/p2-git/) from the
   [Arch User Repository](https://aur.archlinux.org/)
   (e.g. using `yaourt -S p2-git`)

2. `systemctl enable p2.service`

3. `systemctl start p2.service`

## Configuration of FCM, Xmpp-Server, and Conversations

### Firebase Cloud Messaging (FCM)

Navigate to the [Firebase Console](https://console.firebase.google.com), create a project, add cloud messaging to the project and create
an Android app.

You will then need to copy three things from the console:

1. the `service account JSON file` and reference it in `p2.conf` as `fcm/serviceAccountFile`.
2. the sender-id and use it for `push.xml` as `gcm_defaultSenderId`.
3. the app-id and use it for `push.xml` as `google_app_id`.

### Xmpp-Server

You need to configue your xmpp-server to incorporate the push app server p2 as an external component.
Example for ejabberd given below:

```yaml
  ##
  ## ejabberd_service: Interact with external components (transports, ...)
  ##
  -
    port: 5347
    module: ejabberd_service
    access: all
    shaper_rule: fast
    ip: "127.0.0.1"
    hosts:
      "p2.yourserver.tld":
        password: "verysecure"

```

### Building Conversations

Create a file `src/conversationsPlaystore/res/values/push.xml` with the following contents and execute `./gradlew assembleConversationsPlaystoreCompatDebug`

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
  <string name="app_server">p2.yourserver.tld</string>
  <string name="gcm_defaultSenderId" translatable="false">copyfromapiconsole</string>
  <string name="google_app_id">1:copyfromapiconsole:android:copyfromapiconsole</string>
</resources>
```
