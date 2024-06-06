# Examine alerts, logs, and rules with Suricata
**Project Description:** To understand some alerts and logs generated by Suricata I will examine a rule and practice using Suricata to trigger alerts on network traffic. I'll also analyze log outputs, such as a fast.log and eve.json file. I'll learn how to examine a prewritten signature and its log output in Suricata, an open-source intrusion detection system, intrusion prevention system, and network analysis tool.

<!-- In this project, you’ll explore Suricata alerts and logs, including the general process of rule creation. 
The Suricata tool monitors network interfaces and applies rules to the packets that pass through the interface. Suricata determines whether each packet should generate an alert and be dropped, rejected, or allowed to pass through the interface. Source and destination networks must be specified in the Suricata configuration. Custom rules can be written to specify which traffic should be processed
-->

> [!NOTE]
> A sample.pcap file and a custom.rules file resides in the home folder of the linux virtual machine we are monitoring.


## Task 1. Examine a custom rule in Suricata
The `/home/analyst` directory contains a `custom.rules` file that defines the network traffic rules, which Suricata captures.
Explore the composition of the Suricata rule defined in the `custom.rules` file.
  + Use the `cat` command to display the rule in the custom.rules file:
```
cat custom.rules
```
The command returns the rule as the output in the shell:


```
alert http $HOME_NET any -> $EXTERNAL_NET any (msg:"GET on wire";
flow:established,to_server; content:"GET";
http_method; sid:12345; rev:3;)

```
Let's examine each component of this rule from the output above. It has an `action`, a `header`, and `rule options`.
#### 1. action: 
+ The action is the first part of the signature. It determines the action to take if all conditions are met.
> [!IMPORTANT]
> + Actions differ across network intrusion detection system (NIDS) rule languages, but some common actions are `alert, drop, pass,` and `reject`.
> + Using our example, the file contains a single `alert` as the action. The `alert` keyword instructs to alert on selected network traffic. The IDS will inspect the traffic packets and send out an alert in case it matches.
> + Note that the `drop` action also generates an alert, but it drops the traffic. A `drop` action only occurs when Suricata runs in IPS mode.
> + The `pass` action allows the traffic to pass through the network interface. The pass rule can be used to override other rules. An exception to a drop rule can be made with a pass rule. For example, the following rule has an identical signature to the previous example, except that it singles out a specific IP address to allow only traffic from that address to pass:
> + eg. ```pass http 172.17.0.77 any -> $EXTERNAL_NET any (msg:"BAD USER-AGENT";flow:established,to_server;content:!”Mozilla/5.0”; http_user_agent; sid: 12365; rev:1;)```
> + The `reject` action does not allow the traffic to pass. Instead, a TCP reset packet will be sent, and Suricata will drop the matching packet. A TCP reset packet tells computers to stop sending messages to each other.

#### 2. header:
+ The header defines the signature’s network traffic, which includes attributes such as protocols, source and destination IP addresses, source and destination ports, and traffic direction.
> [!IMPORTANT]
> + The next field after the action keyword is the protocol field. In our example, the protocol is http, which determines that the rule applies only to HTTP traffic.
> + The parameters to the protocol `http` field are `$HOME_NET any -> $EXTERNAL_NET any`. The arrow indicates the direction of the traffic coming from the `$HOME_NET` and going to the destination IP address `$EXTERNAL_NET`.
> + `$HOME_NET` is a Suricata variable defined in `/etc/suricata/suricata.yaml` that you can use in your rule definitions as a placeholder for your local or home network to identify traffic that connects to or from systems within your organization.
> + In this project `$HOME_NET` is defined as the 172.21.224.0/20 subnet.
> + The word `any` means that Suricata catches traffic from any port defined in the `$HOME_NET` network.
> + **Therefore the above signature signature triggers an alert when it detects any http traffic leaving the home network and going to the external network.**

> [!NOTE]
> The `$` symbol indicates the start of a variable. Variables are used as placeholders to store values.

#### 3. rule options:
+ rule options allow you to customize signatures with additional parameters. Configuring rule options helps narrow down network traffic so you can find exactly what you’re looking for. rule options are typically enclosed in a pair of parentheses and separated by semicolons.
> [!IMPORTANT]
> + Let's further examine the `rule options` in our output:
> + ```
>   msg:"GET on wire"; 
>   flow:established,to_server; content:"GET";
>   http_method; sid:12345; rev:3;)
>   ```
> + The `msg:` option provides the alert text. In this case, the alert will print out the text `“GET on wire”,` which specifies why the alert was triggered.
> + The `flow:established,to_server` option determines that packets from the client to the server should be matched. (In this instance, a server is defined as the device responding to the initial SYN packet with a SYN-ACK packet.)
> + The `content:"GET"` option tells Suricata to look for the word `GET` in the content of the http.method portion of the packet.
> + The `sid:12345` (signature ID) option is a unique numerical value that identifies the rule.
> + The `rev:3` option indicates the signature's revision which is used to identify the signature's version. Here, the revision version is 3.
> + To summarize, this signature triggers an alert whenever Suricata observes the text `GET` as the HTTP method in an HTTP packet from the home network going to the external network.

## Task 2. Trigger a custom rule in Suricata

Now that we are familiar with the composition of the custom Suricata rule, you must trigger this rule and examine the alert logs that Suricata generates.
1. List the files in the `/var/log/suricata` folder by running the command `ls -l /var/log/suricata`
```
analyst@6e92b5f119af:~$ ls -l /var/log/suricata
total 0
```
> [!Note]
> In the output above, note that before running Suricata, there are no files in the /var/log/suricata directory.

2. Run `suricata` using the `custom.rules` and `sample.pcap` files:
```
sudo suricata -r sample.pcap -S custom.rules -k none
```
We get the following output:
```
4/6/2024 -- 20:49:16 - <Notice> - This is Suricata version 4.1.2 RELEASE
4/6/2024 -- 20:49:17 - <Notice> - all 2 packet processing threads, 4 management threads initialized, engine started.
4/6/2024 -- 20:49:17 - <Notice> - Signal Received.  Stopping engine.
4/6/2024 -- 20:49:19 - <Notice> - Pcap-file module read 1 files, 200 packets, 54238 bytes
```
It returns an output stating how many packets were processed by Suricata.

+ Now you’ll further examine the options in the command:
  + The `-r sample.pcap` option specifies an input file to mimic network traffic. In this case, the sample.pcap file.
  + The `-S custom.rules` option instructs Suricata to use the rules defined in the custom.rules file.
  + The `-k none` option instructs Suricata to disable all checksum checks
    
> [!NOTE]
> Checksums are a way to detect if a packet has been modified in transit. Because you are using network traffic from a sample packet capture file, you won't need Suricata to check the integrity of the checksum.

<!-- Suricata adds a new alert line to the /var/log/suricata/fast.log file when all the conditions in any of the rules are met. -->

3. List the files in the `/var/log/suricata` folder again using `ls -l /var/log/suricata`
   It outputs:
```
analyst@d7fb4ba894b8:~$ ls -l /var/log/suricata
total 16
-rw-r--r-- 1 root root 1432 Jun  4 20:49 eve.json
-rw-r--r-- 1 root root  292 Jun  4 20:49 fast.log
-rw-r--r-- 1 root root 2685 Jun  4 20:49 stats.log
-rw-r--r-- 1 root root  349 Jun  4 20:49 suricata.log
```
> [!NOTE]
> After running Suricata, there are now four files in the /var/log/suricata directory, including the fast.log and eve.json files. 

4. Use the `cat` command to display the `fast.log` file generated by Suricata:
```
cat /var/log/suricata/fast.log
```
The output returns alert entries in the log:
```
analyst@6e92b5f119af:~$ cat /var/log/suricata/fast.log
11/23/2022-12:38:34.624866  [**] [1:12345:3] GET on wire [**] [Classification: (null)] [Priority: 3] {TCP} 172.21.224.2:49652 -> 142.250.1.139:80
11/23/2022-12:38:58.958203  [**] [1:12345:3] GET on wire [**] [Classification: (null)] [Priority: 3] {TCP} 172.21.224.2:58494 -> 142.250.1.102:80
```
> [!IMPORTANT]
> Each line or entry in the `fast.log` file corresponds to an alert generated by Suricata when it processes a packet that meets the conditions of an alert generating rule. Each alert line includes the message that identifies the rule that triggered the alert, as well as the source, destination, and direction of the traffic.

## Task 3. Examine eve.json output
Lets examine the additional output that Suricata generates in the eve.json file.
+ This file is located in the `/var/log/suricata/` directory.
+ The `eve.json` file is the standard and main Suricata log file and contains a lot more data than the `fast.log` file.
+ This data is stored in a JSON format, which makes it much more useful for analysis and processing by other applications.

1.	Use the `cat` command to display the entries in the `eve.json` file, using command `cat /var/log/suricata/eve.json`:
```
{"timestamp":"2022-11-23T12:38:34.624866+0000","flow_id":1939457247443093,"pcap_cnt":70,"event_type":"alert","src_ip":"172.21.224.2","src_port":49652,"dest_ip":"142.250.1.139","dest_port":80,"proto":"TCP","tx_id":0,"alert":{"action":"allowed","gid":1,"signature_id":12345,"rev":3,"signature":"GET on wire","category":"","severity":3},"http":{"hostname":"opensource.google.com","url":"\/","http_user_agent":"curl\/7.74.0","http_content_type":"text\/html","http_method":"GET","protocol":"HTTP\/1.1","status":301,"redirect":"https:\/\/opensource.google\/","length":223},"app_proto":"http","flow":{"pkts_toserver":4,"pkts_toclient":3,"bytes_toserver":357,"bytes_toclient":788,"start":"2022-11-23T12:38:34.620693+0000"}}
{"timestamp":"2022-11-23T12:38:58.958203+0000","flow_id":762337709692148,"pcap_cnt":151,"event_type":"alert","src_ip":"172.21.224.2","src_port":58494,"dest_ip":"142.250.1.102","dest_port":80,"proto":"TCP","tx_id":0,"alert":{"action":"allowed","gid":1,"signature_id":12345,"rev":3,"signature":"GET on wire","category":"","severity":3},"http":{"hostname":"opensource.google.com","url":"\/","http_user_agent":"curl\/7.74.0","http_content_type":"text\/html","http_method":"GET","protocol":"HTTP\/1.1","status":301,"redirect":"https:\/\/opensource.google\/","length":223},"app_proto":"http","flow":{"pkts_toserver":4,"pkts_toclient":3,"bytes_toserver":357,"bytes_toclient":797,"start":"2022-11-23T12:38:58.955636+0000"}}
``` 
+ The output returns the raw content of the file. You'll notice that there is a lot of data returned that is not easy to understand in this format.

2. Use the `jq` command to display the entries in an improved format:
```
jq . /var/log/suricata/eve.json | less
```
The Command outputs
```
analyst@0fde164ce18d:~$ jq . /var/log/suricata/eve.json | less
{
  "timestamp": "2022-11-23T12:38:34.624866+0000",
  "flow_id": 1939457247443093,
  "pcap_cnt": 70,
  "event_type": "alert",
  "src_ip": "172.21.224.2",
  "src_port": 49652,
  "dest_ip": "142.250.1.139",
  "dest_port": 80,
  "proto": "TCP",
  "tx_id": 0,
  "alert": {
    "action": "allowed",
    "gid": 1,
    "signature_id": 12345,
    "rev": 3,
    "signature": "GET on wire",
    "category": "",
    "severity": 3
  },
  "http": {
    "hostname": "opensource.google.com",
    "url": "/",
    "http_user_agent": "curl/7.74.0",
    "http_content_type": "text/html",
    "http_method": "GET",
    "protocol": "HTTP/1.1",
    "status": 301,
    "redirect": "https://opensource.google/",
    "length": 223
  },
  "app_proto": "http",
  "flow": {
    "pkts_toserver": 4,
    "pkts_toclient": 3,
    "bytes_toserver": 357,
    "bytes_toclient": 788,
    "start": "2022-11-23T12:38:34.620693+0000"
  }
}
{
  "timestamp": "2022-11-23T12:38:58.958203+0000",
  "flow_id": 762337709692148,
  "pcap_cnt": 151,
  "event_type": "alert",
  "src_ip": "172.21.224.2",
  "src_port": 58494,
  "dest_ip": "142.250.1.102",
  "dest_port": 80,
  "proto": "TCP",
  "tx_id": 0,
  "alert": {
    "action": "allowed",
    "gid": 1,
    "signature_id": 12345,
    "rev": 3,
    "signature": "GET on wire",
    "category": "",
    "severity": 3
  },
  "http": {

```
> [!Note] 
> + You can use the lowercase f and b keys to move forward or backward through the output. Also, if you enter a command incorrectly and it fails to return to the command-line prompt, you can press CTRL+C to stop the process and force the shell to return to the command-line prompt.
> + Press Q to exit the less command and to return to the command-line prompt.
> + The jq tool is very useful for processing JSON data - For more on `jq` - <a href='https://www.baeldung.com/linux/jq-command-json'>Guide to Linux jq Command for JSON processing</a>

3. Use the `jq` command to extract specific event data from the eve.json file:
```
jq -c "[.timestamp,.flow_id,.alert.signature,.proto,.dest_ip]" /var/log/suricata/eve.json
```
Output result belo:
```
["2022-11-23T12:38:34.624866+0000",1939457247443093,"GET on wire","TCP","142.250.1.139"]
["2022-11-23T12:38:58.958203+0000",762337709692148,"GET on wire","TCP","142.250.1.102"]
```
> [!Note]
> The jq command above extracts the fields specified in the list in the square brackets from the JSON payload. The fields selected are the timestamp `(.timestamp)`, the flow id `(.flow_id)`, the alert signature or msg `(.alert.signature)`, the protocol `(.proto)`, and the destination IP address `(.dest_ip)`.

4. Use the `jq` command to display all event logs related to a specific `flow_id `from the `eve.json` file. The flow_id value is a 16-digit number and will vary for each of the log entries. Replace X with any of the flow_id values returned by the previous query:
```
jq "select(.flow_id==X)" /var/log/suricata/eve.json
```
> [!Note]
> + A network flow refers to a sequence of packets between a source and destination that share common characteristics such as IP addresses, protocols, and more.
> + In cybersecurity, network traffic flows help analysts understand the behavior of network traffic to identify and analyze threats. Suricata assigns a unique `flow_id` to each network flow. All logs from a network flow share the same `flow_id`.
> + This makes the `flow_id` field a useful field for correlating network traffic that belongs to the same network flows.
```
aanalyst@0fde164ce18d:~$ jq "select(.flow_id==1939457247443093)" /var/log/suricata/eve.json
{
  "timestamp": "2022-11-23T12:38:34.624866+0000",
  "flow_id": 1939457247443093,
  "pcap_cnt": 70,
  "event_type": "alert",
  "src_ip": "172.21.224.2",
  "src_port": 49652,
  "dest_ip": "142.250.1.139",
  "dest_port": 80,
  "proto": "TCP",
  "tx_id": 0,
  "alert": {
    "action": "allowed",
    "gid": 1,
    "signature_id": 12345,
    "rev": 3,
    "signature": "GET on wire",
    "category": "",
    "severity": 3
  },
  "http": {
    "hostname": "opensource.google.com",
    "url": "/",
    "http_user_agent": "curl/7.74.0",
    "http_content_type": "text/html",
    "http_method": "GET",
    "protocol": "HTTP/1.1",
    "status": 301,
    "redirect": "https://opensource.google/",
    "length": 223
  },
  "app_proto": "http",
  "flow": {
    "pkts_toserver": 4,
    "pkts_toclient": 3,
    "bytes_toserver": 357,
    "bytes_toclient": 788,
    "start": "2022-11-23T12:38:34.620693+0000"
  }
}
```

## SUMMARY
I now have practical experience in running Suricata to:
+ to trigger alerts on network traffic.
+ create custom rules and run them in Suricata,
+ monitor traffic captured in a packet capture file, and
+ examine the fast.log and eve.json output.








