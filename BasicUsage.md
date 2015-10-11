# Introduction #
The two classes you are confronted with are TCMPortMapper and TCMPortMapping. TCMPortMapper is your primary source of information. If it is running, it you can get your external IP Address, the name of the router (in case it supports UPNP), and the status of the port mappings. Even when it is not running you can access your primary local IP Address, the IP Address of your Router, the HardwareAddress of your Router as well as the manufacturer corresponding to that hardware address. With this information you could e.g. do location based services.

# Code you need to put in #
When you providing a Network service, you need to put this code at the point where you know the ports your app is using:

```
TCMPortMapper *pm = [TCMPortMapper sharedInstance];
[pm addPortMapping:
   [TCMPortMapping portMappingWithLocalPort:myListeningPort 
                        desiredExternalPort:myExternalPort 
                          transportProtocol:TCMPortMappingTransportProtocolTCP
                                   userInfo:nil]];
[pm start];
```

After these calls the TCMPortMapper will try to map this port, and as long as it is running, will adapt to changed network environments. If you want to provide User Feedback (which I hope you want ;) ) the simplest way of doing this is to register for these two notifications, as well as sending you a call when initializing:

```
TCMPortMapper *pm = [TCMPortMapper sharedInstance];
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(portMapperDidStartWork:) 
                                      name:TCMPortMapperDidStartWorkNotification object:pm];
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(portMapperDidFinishWork:)
                                      name:TCMPortMapperDidFinishWorkNotification object:pm];
[O_portStatusImageView setDelegate:self];
if ([pm isAtWork]) {
    [self portMapperDidStartWork:nil];
} else {
    [self portMapperDidFinishWork:nil];
}
```

What you probably want to do in the start variant is to start your progress indicator, update your status text line. When you put the progress indicator over your status Image, you'd probably want to hide that:

```
- (void)portMapperDidStartWork:(NSNotification *)aNotification {
    [O_portStatusProgressIndicator startAnimation:self];
    [O_portStatusImageView setHidden:YES];
    [O_portStatusTextField setStringValue:@"Checking port status..."];
}
```

In the portMapperDidFinishWork: method you obviously need to figure out if the port mapping has taken place (in case of a fixed public IP port mappings will also be set to mapped), and show that. The easiest way is to check all your port mappings for their mapping status. If you only have one, then the code used here should be fine:

```
- (void)portMapperDidFinishWork:(NSNotification *)aNotification {
    [O_portStatusProgressIndicator stopAnimation:self];
    TCMPortMapper *pm = [TCMPortMapper sharedInstance];
    // since we only have one mapping this is fine
    TCMPortMapping *mapping = [[pm portMappings] anyObject];
    if ([mapping mappingStatus]==TCMPortMappingStatusMapped) {
        [O_portStatusImageView setImage:[NSImage imageNamed:@"URLIconOK"]];
        [O_portStatusTextField setStringValue:
            [NSString stringWithFormat:@"see://%@:%d",
                      [pm externalIPAddress],[mapping externalPort]]];
    } else {
        [O_portStatusImageView setImage:[NSImage imageNamed:@"URLIconNotOK"]];
        [O_portStatusTextField setStringValue:@"No public mapping."];
    }
    [O_portStatusImageView setHidden:NO];
}
```

In addition to that - to be a good citizen and remove the port mappings on termination of your application - you need to put this line into your applicationWillTerminate:

```
[[TCMPortMapper sharedInstance] stopBlocking];
```

If you want to provide your users with an option wether to automatically forward your ports, you simply wire it up with the start / stop methods of the port mapper and add a check for the default around the start message you send when your service is setup.

# Changing Port Numbers of Mappings #

Currently the TCMPortMapping objects are immutable objects. If you want to change a port number for an existing mapping then you have to remove it first and readd a different one like this:

```
TCMPortMapper *pm = [TCMPortMapper sharedInstance];
[[TCMPortMapper sharedInstance] removePortMapping:oldMapping];
TCMPortMapping *newMapping = [TCMPortMapping portMappingWithLocalPort:port desiredExternalPort:port transportProtocol:TCMPortMappingTransportProtocolTCP userInfo:nil];
[pm addPortMapping:newMapping];
```

If you manage more than one mapping and want to remove/change them individually then you have to either hold on to them in your code, mark them with a corresponding userInfo or be able to identify them using the port numbers alone.

# Reducing File Size #

If you just need the framework for automatic port forwarding inside your app then you probably don't care about the router name. If you don't you can remove the `OUItoCompany.plist` file from the framework and save about 434 KB.

# Location Awareness #

If you want to build a location aware Application you can use the `-routerHardwareAddress` method to take that as a hint for your location. If you want to do so you should register your self to the `TCMPortMapperDidFinishSearchForRouterNotification` to do stuff based on the location change and also too `TCMPortMapperWillStartSearchForRouterNotification` to go into an intermediate state where you don't know about your location.