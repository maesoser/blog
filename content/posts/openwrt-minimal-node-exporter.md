+++
title = "How to add a prometheus node exporter to a lightweight openwrt router"
date = "2022-12-28"
description = "Adding a prometheus node exporter to a lightweight openwrt installation by making use of uhttpd cgi-bin folder and lua."

[taxonomies]
tags = ["homelab"]

[extra]
author = { name = "Maesoser", social= "https://github.com/maesoser" }
+++

I find exciting to work in constrained environments. Having to work with the tools that I have in hand forces me to think creatively and sometimes to solve things by writing small pieces of software, which I find really rewarding. This was one of these situations. By chance, I had to deal with a TL-WR841N flashed with Openwrt. This is a device with pretty big hardware limitations. Having only 4MB of ROM and 32MB of RAM even Openwrt discourages you from using new OS versions as it is impossible to install and run new packages and, even at the bare minimum, you could experience problems related to Out-Of-Memory situations.

In this case, I wanted to find a way to extract some basic information like CPU, memory consumption and network interface statistics from the router. But I wasn't able to install the [official openwrt node exporter package](https://openwrt.org/packages/pkgdata/prometheus-node-exporter-lua) and its dependencies like lua-socket given the aforementioned device limitations.

I found a [minimalistic version of the lua exporter](https://gist.github.com/lyda/ba33d229a1c01fc5e445) from [@lyda](https://github.com/lyda) and after some small changes I was able to remove the code related to the socket module that allowed the script to act as a small server and just print the output in the console:

```lua
#!/usr/bin/lua

netsubstat = {"IcmpMsg", "Icmp", "IpExt", "Ip", "TcpExt", "Tcp", "UdpLite", "Udp"}
cpu_mode = {"user", "nice", "system", "idle", "iowait", "irq",
            "softirq", "steal", "guest", "guest_nice"}
netdevsubstat = {"receive_bytes", "receive_packets", "receive_errs",
                 "receive_drop", "receive_fifo", "receive_frame", "receive_compressed",
                 "receive_multicast", "transmit_bytes", "transmit_packets",
                 "transmit_errs", "transmit_drop", "transmit_fifo", "transmit_colls",
                 "transmit_carrier", "transmit_compressed"}

function space_split(s)
  elements = {}
  for element in s:gmatch("%S+") do
    table.insert(elements, element)
  end
  return elements
end

function line_split(s)
  elements = {}
  for element in s:gmatch("[^\n]+") do
    table.insert(elements, element)
  end
  return elements
end

function get_contents(filename)
  local f = io.open(filename, "rb")
  local contents = ""
  if f then
    contents = f:read "*a"
    f:close()
  end

  return contents
end

function print_metric_type(metric, mtype)
  this_metric = metric
  print("# TYPE " .. metric .. " " .. mtype .. "")
end

function print_metric(labels, value)
  if labels then
    print(string.format("%s{%s} %g", this_metric, labels, value))
  else
    print(string.format("%s %g", this_metric, value))
  end
end

print ("Content-type: text/plain; version=0.0.4; charset=utf-8\n\r")

local uname = space_split(io.popen("uname -a"):read("*a"))
local stat = get_contents("/proc/stat")
local file_nr = space_split(get_contents("/proc/sys/fs/file-nr"))
local loadavg = space_split(get_contents("/proc/loadavg"))
local meminfo = line_split(get_contents(
                  "/proc/meminfo"):gsub("[):]", ""):gsub("[(]", "_"))
local netstat = get_contents("/proc/net/netstats") .. get_contents("/proc/net/snmp")
local netdevstat = line_split(get_contents("/proc/net/dev"))
for i, line in ipairs(netdevstat) do
  netdevstat[i] = string.match(netdevstat[i], "%S.*")
end

print_metric_type("node_boot_time", "gauge")
print_metric(nil, string.match(stat, "btime ([0-9]+)"))
print_metric_type("node_context_switches", "counter")
print_metric(nil, string.match(stat, "ctxt ([0-9]+)"))
print_metric_type("node_cpu", "counter")
local i = 0
while string.match(stat, string.format("cpu%d ", i)) do
  cpu = space_split(string.match(stat, string.format("cpu%d ([0-9 ]+)", i)))
  local label = string.format('cpu="cpu%d",mode="%%s"', i)
  for ii, mode in ipairs(cpu_mode) do
    print_metric(string.format(label, mode), cpu[ii] / 100)
  end
  i = i + 1
end
print_metric_type("node_filefd_allocated", "gauge")
print_metric(nil, file_nr[1])
print_metric_type("node_filefd_maximum", "gauge")
print_metric(nil, file_nr[3])
print_metric_type("node_forks", "counter")
print_metric(nil, string.match(stat, "processes ([0-9]+)"))
print_metric_type("node_intr", "counter")
print_metric(nil, string.match(stat, "intr ([0-9]+)"))
print_metric_type("node_load1", "gauge")
print_metric(nil, loadavg[1])
print_metric_type("node_load15", "gauge")
print_metric(nil, loadavg[3])
print_metric_type("node_load5", "gauge")
print_metric(nil, loadavg[2])
for i, mi in ipairs(meminfo) do
  local mia = space_split(mi)
  print_metric_type("node_memory_" .. mia[1], "gauge")
  if table.getn(mia) == 3 then
    print_metric(nil, mia[2] * 1024)
  else
    print_metric(nil, mia[2])
  end
end
for i, nss in ipairs(netsubstat) do
  local substat_s = string.match(netstat, nss .. ": ([A-Z][A-Za-z0-9 ]+)")
  if substat_s then
    local substat = space_split(substat_s)
    local substatv = space_split(string.match(netstat, nss .. ": ([0-9 -]+)"))
    for ii, ss in ipairs(substat) do
      print_metric_type("node_netstat_" .. nss .. "_" .. ss, "gauge")
      print_metric(nil, substatv[ii])
    end
  end
end
local nds_table = {}
local devs = {}
for i, nds in ipairs(netdevstat) do
  local dev, stat_s = string.match(netdevstat[i], "([^:]+): (.*)")
  if dev then
    nds_table[dev] = space_split(stat_s)
    table.insert(devs, dev)
  end
end
for i, ndss in ipairs(netdevsubstat) do
  print_metric_type("node_network_" .. ndss, "gauge")
  for ii, d in ipairs(devs) do
    print_metric('device="' .. d .. '"', nds_table[d][i])
  end
end

print_metric_type("node_procs_blocked", "gauge")
print_metric(nil, string.match(stat, "procs_blocked ([0-9]+)"))
print_metric_type("node_procs_running", "gauge")
print_metric(nil, string.match(stat, "procs_running ([0-9]+)"))
print_metric_type("node_time", "counter")
print_metric(nil, os.time())
print_metric_type("node_uname_info", "gauge")
print_metric(string.format('domainname="(none)",machine="%s",nodename="%s",' ..
                           'release="%s",sysname="%s",version="%s %s %s %s %s %s %s"',
                           uname[11], uname[2], uname[3], uname[1], uname[4], uname[5],
                           uname[6], uname[7], uname[8], uname[9], uname[10]), 1)
```

However, I still had a pretty important challenge ahead: how to get that information. Maybe by login in through ssh, running that script and capturing the output? That was quite a complex approach. The ideal solution would be to serve that through http, just as the [Prometheus documentation instructs us to do](https://prometheus.io/docs/instrumenting/writing_exporters/) but I wasn't able to install the socket module. What else could I use as http server on such a stripped down Openwrt version? Well, it _did have_ a full http server, [uhttpd](https://openwrt.org/docs/guide-user/services/webserver/http.uhttpd), the one that is powering Luci, the openwrt web interface. And it turns out that this server has a [Common Gateway Interface (CGI)](https://en.wikipedia.org/wiki/Common_Gateway_Interface) that is one of the parts responsible of serving the Openwrt web interface.

So what I did was to copy my small script to `/www/cgi-bin/metrics` and give it execute permissions and after doing that, just by accessing `http://router.local/cgi-bin/metrics` I was able to remotely run the script and expose the metrics.

In the end, I was able to find a solution that worked for this specific situation and was able to extract and expose the metrics I needed from the router. While working in constrained environments can be challenging, it can also be extremely rewarding when you find a nice and simple solution like I did! The `/cgi-bin/` approach is also interesting to integrate third party modules into an Openwrt router. I hope to have the opportunity to explore this option further in the coming months.
