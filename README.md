# Mirth Code Template Library for on-the-fly TLS offloading using stunnel

# UPDATE: 2025-12-11 - The Open Integration Engine Project is releasing a free open source TLS plugin, ETA two weeks

### Purpose: For users without the TLS plugin, this code template library uses stunnel to enable in and outbound TLS connections.  It is an attempt to ease the configuration of stunnel and do it on the fly.

For Linux only!

Prerequsities:

- Import https://github.com/nextgenhealthcare/connect-examples/tree/master/Code%20Templates/Execute%20Runtime%20Command from Mirth's repo.
- stunnel installed on your linux host, ideally in the path of the mirth server process owner.  If not you will need to edit a code template.
- A private key and server certificate as seperate files at a minimum. (2 files) or the next bullet.
- A private key and server certificate w/CA or CA chain appended (still 2 files, one has the private key, the other has the server cert and the CA chain).
- import the code template library.

To use:

In the Global Undeploy script add:
```JS
stunnel_cleanup();  // This routine kills the stunnel process for channels that have been undeployed when that are undeployed
```
In a channel where you want a TLS wrapped listener:

- Add your private key and server certificate / with chain to server configuration map
- In the deploy script of your channel add the below:

```JS
$gc('mirth_receiver_os_listening_port', '8043')   // Make sure this port is not in use on the linux box
$gc('mirth_receiver_listening_port', '9043')    // Use this var on the source tab for port MUST be above port 1024 and not in use.  Var name is velocity syntax -> ${mirthListeningPort}
stunnel_listener_setup($gc('mirth_receiver_os_listening_port'), $gc('mirth_receiver_listening_port'), $cfg('my_private_key'), $cfg('my_server_cert'))
```
Your Configuration Map would look like:

![image](https://user-images.githubusercontent.com/44065187/198896274-52cec159-08fc-4a19-a18b-7b7b88653e0d.png)


Note the endpoint sending to your listener should be sending to ```http://<mirtth-ip>:8043``` in the above example, and the mirth chnannel should be listening on ```9043```


In a channel where you want a TLS wrapped sender:

- In the deploy script of your channel add the below:

```JS
$gc('endpoint','www.google.com:443')
$gc('mirth_sender_listening_port', '10043')
stunnel_sender_setup($gc('endpoint'), $gc('mirth_sender_listening_port')). // note we throw the first parameter into $g also.
```
Note your mirth sender should be sending to ```http://localhost:10043``` in the above example.

![alt text](https://github.com/pacmano1/mirthstunnel/blob/main/screenshot1.png?raw=true)




# Limitations

- There is no checking for avaiable ports.
- There is no checking for valid private keys and certificates. stunnel will fail on some but not all errors if these are not correct.
- There can be only one listener and one sender per channel. The cleanup code, which kills stunnel and removes the tmp directory depends on this fact.
- If you have cron jobs that cleanup /tmp and the jobs are rather dumb about it they may delete the temp dirs the code templates create. You can always modify the templates to write to a directory structure that the mirth server process owner has control.
- Client certificates (mutual TLS) are not supported.

# Todo
- use tmpfs?
- support client certs
- move temp dirs to appdata?
- Mixing $g and $gc too much, clean it up
- Add CA collection option to then enable more strict checks for senders.

# FAQ

*Why do this per chnanel?*

Effectively to make the termination of the stunnel process(es) and deletion of the temp directories created easier.

*I need multiple channels accessing an stunnel sender, how does that work?*

Nothing prevents this, the stunnel process running in Linux has no concept of what is connecting to it.

*Where are log files for the process runnning on linux?*
Typically ```/var/log/stunnel4``` but consult your linux distribution version of stunnel.

*Something seems wrong, how do I clean up?*

- Login into your server hosting mirth.
- kill all stunnel processes that have a uid in their process name.
- Remove all directores in ```/tmp``` with a UID as their name and contain a file named "stunnel.conf", e.g. ```rm -rf $(find /tmp -name stunnel.conf -execdir pwd \;)```

