# overture
[![Build Status](https://travis-ci.org/shawn1m/overture.svg)](https://travis-ci.org/shawn1m/overture)
[![GoDoc](https://godoc.org/github.com/shawn1m/overture?status.svg)](https://godoc.org/github.com/shawn1m/overture)
[![Go Report Card](https://goreportcard.com/badge/github.com/shawn1m/overture)](https://goreportcard.com/report/github.com/shawn1m/overture)

Overture is a DNS server/forwarder/dispatcher written in Go.

Overture means an orchestral piece at the beginning of a classical music composition, just like DNS which is nearly the
first step of surfing the Internet.

**Please note: If you are using the binary releases, please follow the instructions in the README file with
corresponding git version tag. The README in master branch are subject to change and does not always reflect the correct
 instructions to your binary release version.**

## Features

+ Full IPv6 support
+ Multiple DNS upstream
    + Via UDP/TCP with custom port
    + Via SOCKS5 proxy (TCP only)
    + With EDNS Client Subnet (ECS) [RFC7871](https://tools.ietf.org/html/rfc7871)
+ Dispatcher
    + IPv6 record (AAAA) redirection
    + Custom domain
    + Custom IP network
+ Minimum TTL modification
+ Hosts (**Regex match** for now and will return ip in random order if necessary)
+ Cache with ECS

### Dispatch process

Overture can force custom domain DNS queries to use selected DNS when applicable.

For custom IP network, overture will query the domain with primary DNS firstly. If the answer is empty or the IP
is not matched then overture will finally use the alternative DNS servers.

## Installation

You can download binary releases from the [release](https://github.com/shawn1m/overture/releases).

For ArchLinux users, package `overture` is available in AUR. If you use a AUR helper i.e. `yaourt`, you can simply run:

    yaourt -S overture

## Usages

Start with the default config file -> ./config.json

    $ ./overture

Or use your own config file:

    $ ./overture -c /path/to/config.json

Verbose mode:

    $ ./overture -v

Log to file:

    $ ./overture -l /path/to/overture.log

For other options, please see help:

    $ ./overture -h

Tips:

+ Root privilege is required if you are listening on port 53.
+ For Windows users, you can run overture on command prompt instead of double click.

###  Configuration Syntax

Configuration file is "config.json" by default:

```json
{
  "BindAddress": ":53",
  "DebugHTTPAddress": "127.0.0.1:5555",
  "PrimaryDNS": [
    {
      "Name": "DNSPod",
      "Address": "119.29.29.29:53",
      "Protocol": "udp",
      "SOCKS5Address": "",
      "Timeout": 6,
      "EDNSClientSubnet": {
        "Policy": "disable",
        "ExternalIP": "",
        "NoCookie": true
      }
    }
  ],
  "AlternativeDNS": [
    {
      "Name": "OpenDNS",
      "Address": "208.67.222.222:443",
      "Protocol": "tcp",
      "SOCKS5Address": "",
      "Timeout": 6,
      "EDNSClientSubnet": {
        "Policy": "disable",
        "ExternalIP": "",
        "NoCookie": true
      }
    }
  ],
  "OnlyPrimaryDNS": false,
  "IPv6UseAlternativeDNS": false,
  "WhenPrimaryDNSAnswerNoneUse": "PrimaryDNS",
  "IPNetworkFile": {
    "Primary": "./ip_network_primary_sample",
    "Alternative": "./ip_network_alternative_sample"
  },
  "DomainFile": {
    "Primary": "./domain_primary_sample",
    "Alternative": "./domain_alternative_sample",
    "Matcher":  "regex-list"
  },
  "HostsFile": "./hosts_sample",
  "MinimumTTL": 0,
  "DomainTTLFile" : "./domain_ttl_sample",
  "CacheSize" : 0,
  "RejectQType": [255]
}
```

Tips:

+ BindAddress: Specifying only port (e.g. `:53`) will have overture listen on all available addresses (both IPv4 and
IPv6). Overture will handle both TCP and UDP requests. Literal IPv6 addresses are enclosed in square brackets (e.g. `[2001:4860:4860::8888]:53`)
+ DebugHTTPAddress: Specifying an HTTP port for debugging, currently used to dump DNS cache, and the request url is `/cache`, available query argument is `nobody`(boolean)

    * true(default): only get the cache size;

        ```bash
        $ curl 127.0.0.1:5555/cache | jq
        {
          "length": 1,
          "capacity": 100,
          "body": {}
        }
        ```

    * false: get cache size along with cache detail.

        ```bash
        $ curl 127.0.0.1:5555/cache?nobody=false | jq
        {
          "length": 1,
          "capacity": 100,
          "body": {
            "www.baidu.com. 1": [
              {
                "name": "www.baidu.com.",
                "ttl": 1140,
                "type": "CNAME",
                "rdata": "www.a.shifen.com."
              },
              {
                "name": "www.a.shifen.com.",
                "ttl": 300,
                "type": "CNAME",
                "rdata": "www.wshifen.com."
              },
              {
                "name": "www.wshifen.com.",
                "ttl": 300,
                "type": "A",
                "rdata": "104.193.88.123"
              },
              {
                "name": "www.wshifen.com.",
                "ttl": 300,
                "type": "A",
                "rdata": "104.193.88.77"
              }
            ]
          }
        }
        ```

+ DNS: You can specify multiple DNS upstream servers here.
    + Name: This field is only used for logging.
    + Address: Same as BindAddress.
    + Protocol: `tcp`, `udp` or `tcp-tls`
        + `tcp-tls`: Address format is "servername:port@serverAddress", try one.one.one.one:853 or one.one.one.one:853@1.1.1.1
    + SOCKS5Address: Forward dns query to this SOCKS5 proxy, `“”` to disable.
    + EDNSClientSubnet: Used to improve DNS accuracy. Please check [RFC7871](https://tools.ietf.org/html/rfc7871) for
    details.
        + Policy
            + `auto`: If client IP is not in the reserved IP network, use client IP. Otherwise, use external IP.
            + `manual`: Use external IP if this field is not empty, otherwise use client IP if it is not reserved IP.
            + `disable`: Disable this feature.
        + ExternalIP: If this field is empty, ECS will be disabled when the inbound IP is not an external IP.
        + NoCookie: Disable cookie.
+ OnlyPrimaryDNS: Disable dispatcher feature, use primary DNS only.
+ IPv6UseAlternativeDNS: Redirect IPv6 DNS queries to alternative DNS servers.
+ WhenPrimaryDNSAnswerNoneUse: If the response of PrimaryDNS exists and there is no `ANSWER SECTION` in it, the final DNS should be defined. (There is no `AAAA` record for most domains right now) 
+ File: Absolute path like `/path/to/file` is allowed. For Windows users, please use properly escaped path like
  `C:\\path\\to\\file.txt` in the configuration.
+ DomainFile.Matcher: Matching policy and implementation, including "full-list", "full-map", "regex-list" and "suffix-tree". Default value is "regex-list".
+ MinimumTTL: Set the minimum TTL value (in seconds) in order to improve caching efficiency, use `0` to disable.
+ CacheSize: The number of query record to cache, use `0` to disable.
+ RejectQType: Reject inbound query with specific DNS record types, check [List of DNS record types](https://en.wikipedia.org/wiki/List_of_DNS_record_types) for details.

#### Domain file example (regex match)

    example.com
    ^xxx.xx

#### IP network file example (CIDR match)

    1.0.1.0/24
    10.8.0.0/16
    ::1/128
    
 #### Domain TTL file example (regex match)
 
     example.com$ 100

#### Hosts file example (regex match)

    127.0.0.1 localhost
    ::1 localhost
    10.8.0.1 example.com$

#### DNS servers with ECS support

+ DNSPod 119.29.29.29:53

**For DNSPod, ECS only works via udp, you can test it by [patched dig](https://www.gsic.uva.es/~jnisigl/dig-edns-client-subnet.html)**

You can compare the response IP with the client IP to test the feature. The accuracy depends on the server side.

```
$ dig @119.29.29.29 www.qq.com +client=119.29.29.29

; <<>> DiG 9.9.3 <<>> @119.29.29.29 www.qq.com +client=119.29.29.29
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64995
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; CLIENT-SUBNET: 119.29.29.29/32/24
;; QUESTION SECTION:
;www.qq.com.            IN  A

;; ANSWER SECTION:
www.qq.com.     300 IN  A   101.226.103.106

;; Query time: 52 msec
;; SERVER: 119.29.29.29#53(119.29.29.29)
;; WHEN: Wed Mar 08 18:00:52 CST 2017
;; MSG SIZE  rcvd: 67
```

```
$ dig @119.29.29.29 www.qq.com +client=119.29.29.29 +tcp

; <<>> DiG 9.9.3 <<>> @119.29.29.29 www.qq.com +client=119.29.29.29 +tcp
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58331
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.qq.com.            IN  A

;; ANSWER SECTION:
www.qq.com.     43  IN  A   59.37.96.63
www.qq.com.     43  IN  A   14.17.32.211
www.qq.com.     43  IN  A   14.17.42.40

;; Query time: 81 msec
;; SERVER: 119.29.29.29#53(119.29.29.29)
;; WHEN: Wed Mar 08 18:01:32 CST 2017
;; MSG SIZE  rcvd: 87
```

## Acknowledgements

+ Dependencies:
    + [dns](https://github.com/miekg/dns): BSD-3-Clause
    + [logrus](https://github.com/Sirupsen/logrus): MIT
+ Code reference:
    + [skydns](https://github.com/skynetservices/skydns): MIT
    + [go-dnsmasq](https://github.com/janeczku/go-dnsmasq):  MIT
+ Contributors: https://github.com/shawn1m/overture/graphs/contributors

## License

This project is under the MIT license. See the [LICENSE](LICENSE) file for the full license text.
