---
layout: post
title: "no_proxy variable in requests"
category: posts
---

During deployment of Openstack we experienced issues with configuration of proxy settings in our environment. To access external sites, we have to user http proxy. We have DNS implemented only in Openstack Management network, but users use VPN access from external networks to reach API endpoints by IP addresses from their notebooks or workstations.

API endpoints are in network `172.27.8.0/24`. We have multiple API endpoint IPs, because we have more clouds (Production, Test), and these IPs could change in the future.

In Linux, we have two variables which we need to setup on machine where we want to have access to APIs addresses and to Internet as well:
 
{% highlight bash %}
export http_proxy=http://172.16.1.1:3128/
export no_proxy=127.0.0.1,localhost.localdomain,172.27.8.0/24
{% endhighlight %}
 
`http_proxy` environment variable is used to specify http proxy server where we would like to send all http requests.
`no_proxy` is used to make exceptions and specify, which domains or IP addresses should be reached directly.

Unfortunately, implementation of most of software/libraries (wget, curl, httplib2, requests) is not able to parse network/mask format in no_proxy variable and exclude whole subnet. It is also not possible to use any other format like asterisks or zeros, because there is no other logic behind. This causes Forbidden errors when accessing API from users python script or by `nova`/`glance`/`keystone`/`neutron` client. 

All clients but `neutron` uses [requests][1] library. Hopefully it'll change in near future, because I just [implemented](https://github.com/kennethreitz/requests/commits?author=kmadac "kmadac no_proxy commits") possiblity to use IP subnets in no_proxy variable into requests module and it was successfully merged to master.

Implementation checks whether http request is valid IP address. If yes, it loops through records in `no_proxy` variable and if record is subnet in CIDR format it checks whether IP in http request belongs to that subnet. If it does, request will be routed directly to http server.

Code snippet:

{% highlight python %}
ip = netloc.split(':')[0]
if is_ipv4_address(ip):
    for proxy_ip in no_proxy:
        if is_valid_cidr(proxy_ip):
            if address_in_network(ip, proxy_ip):
                return {}
{% endhighlight %}

From now on, you can exclude whole subnet from http proxy and execute 
{% highlight bash %}
nova list
{% endhighlight %}

from machine and get expected output.

As I wrote above, neutron client does not use [requests][1] at the moment. It uses `httplib2`, but there is plan to replace it -> [more info][2].

[1]: https://github.com/kennethreitz/requests "Requests"
[2]: https://wiki.openstack.org/wiki/SecureClientConnections "SecureClientConnections"