+++
author = "Sergey Nuzhdin"
categories = ["docker", "infrastructure", "kubernetes", "monitoring", "prometheus"]
date = "2017-06-19"
description = "Series of posts about migration from commercial monitoring systems to opensource. Replace NewRelic with Prometheus"
featured = "grafana-cover.png"
featuredalt = ""
featuredpath = "date"
linktitle = ""
title = "Going open-source in monitoring, part II: Creating the first dashboard in Grafana"
+++

This post is one of a series of posts about monitoring of infrastructure and services. Other posts in the series:

0. **[Intro](/post/going-open-source-in-monitoring-part-0-intro/)**
1. **[Deploying Prometheus and Grafana to Kubernetes](/post/going-open-source-in-monitoring-part-i-deploying-prometheus-and-grafana-to-kubernetes/)**
2. **Creating the first dashboard in Grafana** (this article)
3. 10 most useful Grafana dashboards to monitor Kubernetes and services
4. Configuring alerts in Prometheus and Grafana
5. Making sense of logs with ELK(EFK) stack and Sentry
6. Replacing commercial APM monitoring

<hr />

Having [grafana.net/dashboards](https://grafana.net/dashboards) with dozens of dashboards to choose from is awesome and helps get started with monitoring, but creating your own dashboards from scratch is also very helpful.
When I first deployed Grafana, I imported a lot of ready dashboards, really a lot. Many of them worked out of the box, but many did not. In most cases, the fix was in change of template variables, but some required deeper involvement and understanding of how everything works. Such understanding could be acquired by creating your own dashboards.

Since one of the goals of the whole project is to replace NewRelic, the first dashboard will replicate its basic server monitoring page. In case you do not know, it looks like this.



{{< figure src="/img/2017/06/newrelic-server-view.png"  alt="NewRelic Server View" >}}



The corresponding dashboard we building is shown below.

{{< figure src="/img/2017/06/newrelic-prom-view.png"  alt="Grafana Server View Dashboard" >}}



# Creating server monitoring dashboard

It’s time to create the dashboard. If you just installed Grafana you’ll have 
“Create your first dashboard” button on the homepage. Otherwise, click “create new” from the top dropdown. 
I’m not going to copy the [gettings-started guide](http://docs.grafana.org/guides/getting_started/) from grafana docs, so if you do not know grafana basics, like how to add a graph to the dashboards - read the [guide](http://docs.grafana.org/guides/getting_started/) or watch the 10min [beginners guide to building dashboards](https://www.youtube.com/watch?v=sKNZMtoSHN4&index=7&list=PLDGkOdUX1Ujo3wHw9-z5Vo12YLqXRjzg2) to get a quick intro to setting up Dashboards and Panels.

**Templating**

One of the essential features of Grafana is an ability to create variables. It allows you to create interactive and dynamic dashboards. You can have a single variable, like server selector, or a chain of variables each new populated based on previous selection, like namespace -> deployment -> Pod.

Templating is usually not the first thing covered in manuals. The common way is to create all graphs with hardcoded values and then replace it with the variable. I’m going to cover templating first and use variables from the beginning. 

For this dashboard, we need only one variable - server selector. 
 
To add one go to **Manage dashboards** → **Templating** → **Variables**,  and click  **+New.**

{{< figure src="/img/2017/06/templating1.png"  alt="Grafana template add" >}}



You’ll get to the edit variable tab.
First, set **Name** and **Label** of the new variable. I prefer to have identical label and name, it helps you not to keep the mapping between label and its name in mind.

Next, we need to set query options. This is the place where we can select data to populate dropdown.
**Data source**: prometheus data source you’ve created
**Refresh**: one of: Never / On Dashboard Load / On Time Range change. Choose what is suitable for your setup. 
**Query**:  label_values(node_boot_time{component=”node-exporter”}, instance). What does this query say. Take `node_boot_time` metric, filter it by label component="node-exporter" and return a  list of values of the label `instance` . [More about queries](http://docs.grafana.org/features/datasources/prometheus/#query-variable).
**Regex**: Regular expression is optional, but sometimes is very helpful. For example, in my case, the result of the query is IP:PORT, but I want to show only IP in the dropdown. With regex you can make required transformations.

During edit of the query, you will see a realtime result in the bottom,  under the “Preview of the values" block.


{{< figure src="/img/2017/06/templating2.png"  alt="Grafana template edit" >}}

More detailed manual about templating could be found in the [docs](http://docs.grafana.org/reference/templating/) 

Now, when we have our node variable, with all servers in it, we can add some graphs.

**CPU usage**

{{< figure src="/img/2017/06/newrelic_cpu_usage.png" class="image left" width="49%" alt="NewRelic CPU usage" >}}
{{< figure src="/img/2017/06/grafana_cpu_usage.png" class="" width="49%" alt="Grafana CPU usage" >}}

<br />

Let’s walk through creating of this one in details. 
To build this graph we need 4 queries, all using `node_cpu` metric from node-exporter. Here are the queries.


    system: (avg by (instance) (irate(node_cpu{instance=~"$node:.*",mode=~"system|irq|softirq"}[5m])) * 100)
    user: (avg by (instance) (irate(node_cpu{instance=~"$node:.*",mode="user"}[5m])) * 100)
    iowait: (avg by (instance) (irate(node_cpu{instance=~"$node:.*",mode="iowait"}[5m])) * 100)
    idle: (avg by (instance) (irate(node_cpu{instance=~"$node:.*",mode="idle"}[5m])) * 100)

All these queries differ only by the mode filter, so let’s look only at the first one - system.
What is happening here: *node_cpu* metric is filtered by instance using our previous template variable and by mode. Since it is counters we need to use irate(rate) function to calculate the per-second rate. This will produce the list of results, one for each CPU/core on the machine, so we need to aggregate it to get a single per-node value. At the end, we need to multiply it by 100 to get percents of the total load.

Here is a good article about the [rate vs irate](https://www.robustperception.io/irate-graphs-are-better-graphs/) difference and [this one](https://www.robustperception.io/understanding-machine-cpu-usage/) is about CPU usage calculation.

**SystemLoad**


{{< figure src="/img/2017/06/newrelic_load_average.png" class="image left" width="49%" alt="NewRelic Load Average" >}}
{{< figure src="/img/2017/06/grafana_load_average.png" class="" width="49%" alt="Grafana Load Average" >}}


    load5: node_load5{instance=~"$node:.*"}

**Memory**


{{< figure src="/img/2017/06/newrelic_memory.png" class="image left" width="49%" alt="NewRelic Memory" >}}
{{< figure src="/img/2017/06/grafana_memory.png" class="" width="49%" alt="Grafana Memory" >}}

    total: node_memory_MemTotal{instance=~"$node:.*"}
    used: node_memory_MemTotal{instance=~"$node:.*"} - node_memory_MemFree{instance=~"$node:.*"} - node_memory_Cached{instance=~"$node:.*"} - node_memory_Buffers{instance=~"$node:.*"} - node_memory_Slab{instance=~"$node:.*"}
    swap: (node_memory_SwapTotal{instance=~"$node:.*"} - node_memory_SwapFree{instance=~"$node:.*"})


**Network I/O**

{{< figure src="/img/2017/06/newrelic_network.png" class="image left" width="49%" alt="NewRelic Network" >}}
{{< figure src="/img/2017/06/grafana_network.png" class="" width="49%" alt="Grafana Network" >}}

    received: sum(irate(node_network_receive_bytes{instance=~"$node.*"}[5m]))
    transmitted: sum(irate(node_network_transmit_bytes{instance=~"$node.*"}[5m]))

**Disk I/O**


{{< figure src="/img/2017/06/newrelic_disk_io.png" class="image left" width="49%" alt="NewRelic DiskIO" >}}
{{< figure src="/img/2017/06/grafana_disk_io.png" class="" width="49%" alt="Grafana DiskIO" >}}

    read_by_device: irate(node_disk_reads_completed{instance=~"$node:.*", device!~"dm-.*"}[5m])
    write_by_device: irate(node_disk_writes_completed{instance=~"$node:.*", device!~"dm-.*"}[5m])

**Top 10 memory hungry pods**

{{< figure src="/img/2017/06/grafana_top10_memory.png" alt="Grafana Top 10 memory hungry PODs" >}}


For this graph, we’re going to use two new functions - **sort_desc** and **topk**. 
`topk` is used to limit results, while `sort_desc` is to sort it.
`by (pod_name)` - will group results by pod_name


    metrics: sort_desc(topk(10, sum(container_memory_usage_bytes{pod_name!=""}) by (pod_name)))

**Top 10 CPU hungry pods**

{{< figure src="/img/2017/06/grafana_top10_cpu.png" alt="Grafana Top 10 CPU hungry PODs" >}}


    metrics: sort_desc(topk(10, sum by (pod_name)( rate(container_cpu_usage_seconds_total{pod_name!=""}[1m] ) )))


## Node overview
{{< figure src="/img/2017/06/grafana_node_overview.png" alt="Grafana Node overview" >}}


This row is a bit different from all the others. While all previous stats were using Graph type, here we're going to use Singlestats **and Table.

**Total RAM:**


    type: singlestat
    metrics: node_memory_MemTotal{instance=~"$node:.*"}
    options: stat=current, unit=bytes, decimals=1

**CPU cores:**

    type: singlestat
    metrics: count(count(node_cpu{instance=~"$node:.*"}) by (cpu))

**Uptime:**

    type: singlestat
    metrics: node_time{instance=~"$node:.*"} - node_boot_time{instance=~"$node:.*"}
    options: stat=current, unit=seconds, decimals=1

**Disk Space Usage:**


    type: singlestat
    metrics: sum((node_filesystem_size{instance=~"$node:.*",mountpoint="/rootfs"} - node_filesystem_free{instance=~"$node:.*",mountpoint="/rootfs"}) / node_filesystem_size{instance=~"$node:.*",mountpoint="/rootfs"}) * 100

**FileSystem Fill Up Time**

{{< figure src="/img/2017/06/grafana_disk_fill_prediction.png" alt="Grafana Disk fill prediction" >}}


This one is a bit more sophisticated. It uses simple linear regression to predict time until disks fill up.
We’re using `avg by (device)` to show each device only once, otherwise, we will have one line for each mountpoint. 


    type: table
    metrics: avg by (device) (deriv(node_filesystem_free{device=~"/dev/sd.*",instance=~"$node:.*"}[4h]) > 0)


> In case you only have 3 mountpoints (/etc/hosts, /etc/hostsname, /etc/resolv.conf) exposed by node-exporter - double check that your node-exporter pods have all the rootfs directories mounted. As stated [here](https://github.com/prometheus/node_exporter/blob/master/README.md#using-docker).


# Wrap up

That’s it. Now we have a node overview dashboard which shows all the basic information available in NewRelic server view. As a bonus, we’ve got disk fullness prediction and information about most RAM/CPU hungry pods.
Source of this dashboard is available on [GitHub](https://github.com/lwolf/kube-monitoring/blob/master/dashboards/server-overview.json).

Stay tuned.



