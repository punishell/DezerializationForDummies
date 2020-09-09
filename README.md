# DezerializationForDummies
![tpb](https://raw.githubusercontent.com/punishell/DezerializationForDummies/master/tpb.jpg)

Some interesting qoutes and comments regarding to deserialization in JAVA.

## What the Fu*k is going on?
### Code example:
```
public class Session {
  public String username;
  public boolean loggedIn;
  
  public void loadSession(byte[] sessionData) throws Exception {
    ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(sessionData));
    this.username = ois.readUTF();
    this.loggedIn = ois.readBoolean();
  }
}
```

### The problem starts in ObjectInputStream
We can identify entry points for deserialization vulnerabilities by reviewing application source code for the use of the class ‘java.io.ObjectInputStream’ (and specifically the ‘readObject’ method), or for serializable classes that implement the ‘readObject’ method. 
If an attacker can manipulate the data that is provided to the ObjectInputStream then that data presents an entry point for deserialization attacks. Alternatively, or if the Java source code is unavailable, we can look for serialized data being stored on disk or transmitted over the network, provided we know what to look for!

### Spot the protocol
The Java serialization format begins with a two-byte magic number which is always hex **0xAC ED**.
Look for the four-byte sequence 0xAC ED 00 05 in order to identify Java serialization, sometimes client application kept a network connection to the server open the entire time, so four-byte header only exists once at the very beginning of a serialization stream.

The most obvious indicator of Java serialization data is the presence of Java class names in the dump, such as ‘java.rmi.dgc.Lease’. In some cases Java class names might appear in an alternative format that begins with an ‘L’, ends with a ‘;’, and uses forward slashes to separate namespace parts and the class name (e.g. ‘Ljava/rmi/dgc/VMID;’).

### Something Fu*ky
![sth_fucky](https://raw.githubusercontent.com/punishell/DezerializationForDummies/master/sth_fucky.jpg)


Identified the use of serialized data, we need to identify the offset into that data where we can actually inject a payload. The target needs to call ‘ObjectInputStream.readObject’ in order to deserialize and instantiate an object (payload) and support property-oriented programming, however it could call other ObjectInputStream methods first, such as ‘readInt’ which will simply read a 4-byte

The readObject method will read the following content types from a serialization stream:
0x70 – TC_NULL
0x71 – TC_REFERENCE
0x72 – TC_CLASSDESC
0x73 – TC_OBJECT
0x74 – TC_STRING
0x75 – TC_ARRAY
0x76 – TC_CLASS
0x7B – TC_EXCEPTION
0x7C – TC_LONGSTRING
0x7D – TC_PROXYCLASSDESC
0x7E – TC_ENU

Here comes [the tool](https://github.com/NickstaDB/SerializationDumper)
Which we can uhelp to identify entry points for deserialization.
Example:
```
$ java -jar SerializationDumper-v1.0.jar ACED00057708af743f8c1d120cb974000441424344
STREAM_MAGIC - 0xac ed
STREAM_VERSION - 0x00 05
Contents
  TC_BLOCKDATA - 0x77
    Length - 8 - 0x08
    Contents - 0xaf743f8c1d120cb9
  TC_STRING - 0x74
    newHandle 0x00 7e 00 00
    Length - 4 - 0x00 04
    Value - ABCD - 0x41424344
```
In this example the stream contains a TC_BLOCKDATA followed by a TC_STRING which can be replaced with a payload.
### Now What?
![ricky](https://raw.githubusercontent.com/punishell/DezerializationForDummies/master/ricky.png)

Having identified an entry point, the next thing we need are POP gadgets.
In order to execute some commmand we need POP Gadget chain but dont worry here is another great [the tool](https://github.com/frohoff/ysoserial/).

### Fu*k off i got work to do
![cyrus](https://raw.githubusercontent.com/punishell/DezerializationForDummies/master/cyrus.png)

So whats now? Go and test new knowledge in [the lab](https://github.com/NickstaDB/DeserLab).


To run the server and client, you can use the following commands:
```
java -jar DeserLab.jar -server 127.0.0.1 6666
 [+] DeserServer started, listening on 127.0.0.1:6666
 [+] Connection accepted from 127.0.0.1:50410
 [+] Sending hello...
 [+] Hello sent, waiting for hello from client...
 [+] Hello received from client...
 [+] Sending protocol version...
 [+] Version sent, waiting for version from client...
 [+] Client version is compatible, reading client name...
 [+] Client name received: testing
 [+] Hash request received, hashing: test
 [+] Hash generated: 098f6bcd4621d373cade4e832627b4f6
 [+] Done, terminating connection.
 
java -jar DeserLab.jar -client 127.0.0.1 6666
 [+] DeserClient started, connecting to 127.0.0.1:6666
 [+] Connected, reading server hello packet...
 [+] Hello received, sending hello to server...
 [+] Hello sent, reading server protocol version...
 [+] Sending supported protocol version to the server...
 [+] Enter a client name to send to the server:
 testing
 [+] Enter a string to hash:
 test
 [+] Generating hash of "test"...
 [+] Hash generated: 098f6bcd4621d373cade4e832627b4f6

```
Ok so lets capture the trafic and run above client again after:

```
kali@kali:#  tcpdump -i lo -n -w deserlab.pcap 'port 6666'
```
Now lets check  our serialized data:
```
kali@kali:# tshark -r deserlab.pcap -T fields -e tcp.srcport -e data -e tcp.dstport -E separator=, | grep -v ',,' | grep '^6666,' | cut -d "," -f2 |tr "n" ":" | sed s/://g | tr -d '\n' 
                                                                                                                                                                         
aced00057704f000baaa77020101737200146e622e64657365722e4861736852657175657374e52ce9a92ac1f9910200024c000a64617461546f486173687400124c6a6176612f6c616e672f537472696e673b4c00077468654861736871007e00017870740004746573747400203039386636626364343632316433373363616465346538333236323762346636root@kali
```

The above command only selects the server response, if you want to get the client data, you need to change the port number. The final result is as follows:

```
aced00057704f000baaa77020101737200146e622e64657365722e486 [...]
```
Now we can run analyse tool:
```
java -jar SerializationDumper-v1.0.jar aced00057704f000baaa77020101
After execution, it should output something similar to the following:

STREAM_MAGIC-0xac ed
STREAM_VERSION-0x00 05
Contents
 TC_BLOCKDATA-0x77
 Length-4-0x04
 Contents-0xf000baaa
 TC_BLOCKDATA-0x77
 Length-2-0x02
 Contents-0x0101
 TC_OBJECT-0x73
 TC_CLASSDESC-0x72
 className
 Length-20-0x00 14
 Value-nb.deser.HashRequest-0x6e622e64657365722e4861736852657175657374
```
if you have problem running it from command line try:
```
java -jar SerializationDumper.jar -f hex.txt 
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true

STREAM_MAGIC - 0xac ed
STREAM_VERSION - 0x00 05
Contents
  TC_BLOCKDATA - 0x77
    Length - 4 - 0x04
    Contents - 0xf000baaa
  TC_BLOCKDATA - 0x77
    Length - 2 - 0x02
    Contents - 0x0101
  TC_OBJECT - 0x73
    TC_CLASSDESC - 0x72
      className
        Length - 20 - 0x00 14

```














ps. almose everything here is ctrl+c strl+v but i did enjoy adding some TPB pictures and learn something :)
### Reference
https://nickbloor.co.uk/2017/08/13/attacking-java-deserialization/

https://github.com/NickstaDB/SerializationDumper

https://docs.oracle.com/javase/7/docs/api/java/io/Serializable.html

https://juejin.im/entry/6844903501353451534

http://randomlinuxtech.blogspot.com/2017/08/java-deserialization-howto.html

https://blog.csdn.net/qsort_/article/details/104874111

https://blog.csdn.net/qsort_/article/details/104969138

https://meteatamel.wordpress.com/2012/02/13/jmx-rmi-vs-jmxmp/

https://github.com/frohoff/ysoserial/

