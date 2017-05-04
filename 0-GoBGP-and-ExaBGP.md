# How to set up two routers that talk to each other

## Idea

    Router 1 (ExaBGP) <--> Router 2 (GoBGP)

# Installation of ExaBGP and GoBGP

1. [Download exabgp](https://github.com/Exa-Networks/exabgp/) to `/opt/exabgp` so that `/opt/exabgp/sbin/exabgp` exists.
2. [Add `GOPATH` and `GOROOT` environment variables](https://stackoverflow.com/questions/21001387/how-do-i-set-the-gopath-environment-variable-on-ubuntu-what-file-must-i-edit) and then [Download GoBGP](https://github.com/osrg/gobgp). You should be able to call `gobgp` in the shell now.

We could also use ExaBGP or GoBGP twice but that wouldn't be fun, would it? Now go ahead and create a project folder.

# Configuration of GoBGP

1. Create a file called `router-gobgp.conf` and put the following in there:
```
    [global.config]
        as = 65000
        router-id = "GoBGP Router"
        local-address-list = ["127.0.0.1"]
        port = 1000

    [[neighbors]]
      [neighbors.config]
        neighbor-address = "127.0.0.1"
        peer-as = 65000

      [neighbors.transport.config]
        local-address = "127.0.0.1"
        remote-port = 1001
```

2. Now for a little explanation. In the `global.config` section you can see that we are announcing ourselves to be AS65000 which is a private AS number. This is much like [RFC1918](https://en.wikipedia.org/wiki/Private_network) IP adresses like `192.168.?.?`. The `router-id` is something like our name. In this case we are running two routers (one ExaBGP, one GoBGP) on one machine without communicating with other machines. In an actual scenario you would put most probably the IP of the router there or some other unique name. The `local-address-list` and `port` section contains the adresses and port that the router listens for connections from other devices (i.e. other routers). In summary: We now have a router that announces itself as AS65000 with a name `GoBGP Router`, listening on `127.0.0.1:1000` for connections. Next, let's  have a look at the neighbors section. We add one neighbor (which will be the ExaBGP router) at `127.0.0.1` aswell. It announces the same AS number. This means that those routers are in the same AS and thus are talking IBGP, not EBGP. Would the routers have different AS then they would be talking EBGP. In the `neighbors.transport.config` section we configure that the GoBGP router will search the ExaBGP router at `127.0.0.1:1001`.

3. Now for the other side: the ExaBGP router. Create a file called `router-exabgp.conf` and add this:

```
    group somename {
        local-as 65000;
        router-id "ExaBGP Router";
        local-address 127.0.0.1;

        family {
           ipv4 unicast;
           ipv6 unicast;
        }

        process stdout {
            run "./process.py";
            encoder json;

            receive {
                parsed;
                update;
            }
        }

        neighbor 127.0.0.1 {
            local-address 127.0.0.1;
            connect 1000;
            peer-as 65000;
        }
    }
```

4. This configuration is similar (who would have guessed? they're both routers!). We announce that we are part of AS65000 (IBGP, remember?). We have a name `router-id` that must be unique inside an AS. Also, we are listening on `127.0.0.1` aswell. We will provide the port through the command line. The family section says that we can handle `IPv4` and `IPv6` through unicast. Let's skip the process section for now. Last, we need to add the first router (GoBGP) as a neighbor. Remember that we set that up at `127.0.0.1:1000`. The neighbor is also in the same AS.

5. Next, we need to start both routers.

* `sudo /home/jonas/go/bin/gobgpd -f router-gobgp.conf`
* `sudo env exabgp.tcp.bind="127.0.0.1" exabgp.tcp.port=1001 sudo /opt/exabgp/sbin/exabgp router-exabgp.conf`

6. Finish! You have two routers talking to each other set up in an IBGP configuration. Of course, there are is no actual data in the system yet. For fun, try `gobgp global rib add 10.10.0.0/16 -a ipv4` and see what the ExaBGP router does.
