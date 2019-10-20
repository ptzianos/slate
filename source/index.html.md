---
title: Architecture and Integration Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - java

toc_footers:
  - <a href='https://github.com/lord/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true
---

# Introduction

Welcome to the documentation of the Hermes project! Hermes is a small, fully
open-source ecosystem of tools that can be used to store and retrieve data
using Distributed Ledgers. The tools are especially geared towards pushing,
selling and retrieving monitoring data.

The main components that comprise the ecosystem are the Hermes Marketplace,
the Hermes Android Application and the Carbon-ledger project. The Android
Application is used to gather data from a smartphone, the Marketplace is
used to post advertisements/requests for data and coordinate transactions,
and the Carbon-Ledger can be used to retrieve and follow streams of data.

# Hermes Android Application

The Android Application is built to be used with Android API version >= 24.
It allows a user to stream data from a sensor or another application into
a Distributed Ledger.

Currently supported Distributed Ledgers:
1. IOTA

## Integration with the application

As mentioned previously, the application allows other external applications
to connect to it via an AIDL interface. An example, of such an integration
has been done in the 
[Roomr application](https://github.com/ntsiam/roomr-android). There is also
documentation on how to perform the integration also in the official Android
docs [here](https://developer.android.com/guide/components/aidl). More
specifically, you need to copy the
`android-client/app/src/main/aidl/org/hermes/IHermesService.aidl` file into
your code and then you just need to import it:

```java
import org.hermes.IHermesService;
```

This is the interface that is implemented by the Hermes Application and it
resembles a server that you can stream data into. The first step it to try to
connect to the service that implements the `IHermesService`.

## Service Connection

If the Hermes application has not been installed in the system or if the
application has not been started, an error will be thrown at this
stage. The user must have started and be registered or logged into the
application, in order for the `IHermesService` endpoint to be established.
Once the connection has been established time you need to invoke
the `register` method of the stub. If everything works fine, the method will
return a UUID as a string. You need to keep this string, because it will be
used as an identifier for your sensor. Every sensor must have it's own
UUID, so if you wish to stream data from more than one sensors, you need to
invoke `register` once for each sensor, but you can use the same service object
for all of them. The `register` method, needs at least three arguments:

1. The ID of the sensor,
2. The type of data that will be streamed,
3. The source of the data that will be streamed

The ID of the sensor must be a dot separated string, that can have as many
components as needed. It must contain words that will help identify the
stream of data. This tag will be combined with the first 10 characters
of the digest of the key that the Hermes Application uses to sign all data
streams, so it can be uniquely identified in a secure way.

```java
ServiceConnection serviceConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName className, IBinder service) {
        iHermesService = IHermesService.Stub.asInterface((IBinder) service);
        try {
            uuid = iHermesService.register("some.android.app", "string", "sensor_type", null, null);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        if (uuid == null)
            return;
    }

    @Override
    public void onServiceDisconnected(ComponentName componentName) {
        iHermesService = null;
    }
};
```

## Finalizing the service connection establishment

The ServiceConnection object has been initialized, but it has not been
triggered yet. Now you need to connect it to an implementation of the
`IHermesService`, specifically the `LedgerService` class of the Hermes
application. The following code achieves  that:

```java
import static android.content.pm.PackageManager.*;

PackageInfo otherApp;
try {
    otherApp = getBaseContext().getPackageManager().getPackageInfo("org.hermes", GET_SERVICES);
} catch (NameNotFoundException e) {
    // Hermes application is definitely not installed in the system!
    return;
}

// We are still not sure that the Hermes application is installed.
// We need to do one final check to ensure that the LedgerService in particular is present in
// the system.
ServiceInfo hermesService = null;
for (int i=0 ; i<otherApp.services.length ; i++) {
    ServiceInfo serviceInfo = otherApp.services[i];
    if (serviceInfo.name.equals("org.hermes.LedgerService")) {
        hermesService = serviceInfo;
        break;
    }
}
if (hermesService == null) {
    // packages of the Hermes application are installed but not the LedgerService
} else {
    Intent intent = new Intent();
    intent.setComponent(new ComponentName(hermesService.packageName, hermesService.name));
    if (getBaseContext().bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)) {
        // now we have finallu established a connection to the ledger service
    } else {
        // there was some error and we can not finish establishing the connection
    }
}
```

## Sending data to the application

Once a sensor has been registered, you can start streaming data immediately.
There are two methods that you can use: `sendData` and `sendDataString`,
depending on the type of data you wish to send. There is no reason to send
data with a certain rate, but keep in mind that you have no control over how
data will be broadcast to the network. The Hermes Application will buffer
these data and will broadcast them as soon as possible, but keep in mind
that smartphones are subject to bad or non-existent network connectivity,
so some of the packets might not make it. A small example of code that
invokes the `sendDataString` method is the following:

```java
JSONObject json = new JSONObject();
json.put("lat", currentPosition.latitude);
json.put("lng", currentPosition.longitude);
json.put("timestamp", unixTime);
String res = iHermesService.sendDataString(uuid, json.toString(), null, null, null, null, null, null, -1, null);
```

## Privacy concerns and integration

An application like Hermes obviously comes with some privacy issues. The point
behind the architecture of the application is that the user must **always**
be in control and have final say over what is being streamed. In order for
any data to be streamed, the user must log in to the application and
explicitly allow it to start broadcasting data. When a new application
connects to the Hermes application, it will appear in the sensor list. Even if
the service connection has been established and data packets have started
arriving, nothing will be streamed until the user allows it. The application
does perform some buffering internally to allow the user some time to react
and ensure that as few packets as possible are dropped during phases of poor
or non-existent network connectivity.

# Streaming Protocol

There is nothing in the architecture that forces a specific protocol to be
used for streaming data into a ledger. Every advertisement in a marketplace
contains a string that shows which protocol is used for a stream. The
supported protocols are the following:

1. Hermes Plaintext,
2. Hermes Plaintext v2 (under development),
...

## Hermes Plaintext

This is the default protocol used by the application to stream its data. It
builds a linked list on top of the ledger, where each node contains one or
more packets of data. The header of the node contains the address of the next
and the previous node and the signature of the contents of the node. The
contents of the nodes are signed with the key of the Hermes application. All
fields are separated with a double colon. Each packet of data is encoded into
a string using the [Carbon 2.0 format](https://github.com/metrics20/go-metrics20).


## Hermes Plaintext v2

Pretty much the same as v1 except that it allows the data inside the node to
be encrypted and be chunked. Some ledgers do not allow an arbitrary amount of
data to be stored in each packet, so this protocol handles this allowing the
data of one packet to be split into multiple nodes. It also makes more efficient
use of space because it doesn't replicate the headers of each packet.


# Marketplace

The marketplace can be used to create advertisements and requests for streams
of data. The REST API of the marketplace is versioned.

Under construction...
