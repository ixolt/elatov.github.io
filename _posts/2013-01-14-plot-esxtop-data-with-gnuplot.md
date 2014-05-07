---
title: Plot Esxtop Data With gnuplot
author: Karim Elatov
layout: post
permalink: /2013/01/plot-esxtop-data-with-gnuplot/
dsq_thread_id:
  - 1406205899
categories:
  - Networking
  - Storage
  - VMware
tags:
  - Average Guest MilliSec/Read
  - awk
  - CSV
  - esxtop
  - esxtop batch mode
  - gnuplot
  - head
  - headers
  - Reads/sec
  - set format x
  - set timefmt
  - tail
---
Let&#8217;s say you collected esxtop batch data per the instructions laid out in VMware KB <a href="http://kb.vmware.com/kb/1004953" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://kb.vmware.com/kb/1004953']);">1004953</a>. So in the end you just had a huge CSV (Comma Separated Values) file, with a lot of data. In my example I was seeing some latency on my NFS datastore and I wanted to find out what is going on. I downloaded the file to my linux machine and here is resulted file:

[code]  
[elatov@klaptop]$ ls -lh esxtop_data.csv  
-rw-rw-r-- 1 elatov elatov 309M Dec 11 18:54 esxtop_data.csv  
[/code]

So the first thing that I wanted to do was grab all the headers and separate them by new lines (instead of commas). This will make it easier to count what esxtop counter/field corresponds to what column in the collected data set. Here is command to achieve that:

[code]  
[elatov@klaptop]$ head -1 esxtop_data.csv | sed 's/\,/\n/g' > headers  
[/code]

Now checking out the &#8216;headers&#8217; file, I saw the following:

[code]  
[elatov@klaptop]$ head headers  
"(PDH-CSV 4.0) (UTC)(0)"  
"\\local\Memory\Memory Overcommit (1 Minute Avg)"  
"\\local\Memory\Memory Overcommit (5 Minute Avg)"  
"\\local\Memory\Memory Overcommit (15 Minute Avg)"  
"\\local\Physical Cpu Load\Cpu Load (1 Minute Avg)"  
"\\local\Physical Cpu Load\Cpu Load (5 Minute Avg)"  
"\\local\Physical Cpu Load\Cpu Load (15 Minute Avg)"  
"\\local\Physical Cpu(0)\% Processor Time"  
"\\local\Physical Cpu(1)\% Processor Time"  
"\\local\Physical Cpu(2)\% Processor Time"  
[/code]

That is perfect, the columns are now separated by new lines. Now the name of the NFS datastore was &#8220;NFS_Datastore&#8221;. So searching for that in the &#8216;headers&#8217; file, I saw the following:

[code]  
[elatov@klaptop]$ grep NFS_Datastore headers  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Active Commands"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Commands/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Reads/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Writes/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\MBytes Read/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\MBytes Written/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Average Guest MilliSec/Command"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Average Guest MilliSec/Read"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Average Guest MilliSec/Write"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Failed Commands/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Failed Reads/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Failed Writes/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Failed MBytes Read/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Failed MBytes Written/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Failed Reserves/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Aborts/sec"  
"\\local\Physical Disk NFS Volume(NFS_Datastore)\Resets/sec"  
[/code]

So now we see all the available counters/fields for our NFS datastore. I first wanted to concentrate on the Read latency, so I wanted to see what column number did the counter &#8220;Average Guest MilliSec/Read&#8221; correspond to. So I ran the following to determine that:

[code]  
[elatov@klaptop]$ grep -n NFS_Datastore headers | grep 'Average Guest MilliSec/Read'  
16391:"\\local\Physical Disk NFS Volume(NFS_Datastore)\Average Guest MilliSec/Read"  
[/code]

So the column number that corresponded to read latency of the NFS datastore was &#8217;16391&#8242;. So now let&#8217;s grab two columns from our dataset: the first one, which corresponds to the time, and the &#8217;16391&#8242;th one, which corresponds to read latency of our datastore. To do this there are two steps, first let&#8217;s just grab the data and skip the headers. This is done like so:

[code]  
[elatov@klaptop]$ tail -n +2 esxtop_data.csv > data  
[/code]

Next let&#8217;s just grab the columns that we desire from the dataset:

[code]  
[elatov@klaptop]$ awk -F , '{print $1","$16391}' data > nfs\_read\_latency_data  
[/code]

Confirming we just have the two column, we can check the data really fast:

[code]  
[elatov@klaptop]$ head nfs\_read\_latency_data  
"12/11/2012 19:09:23","0.00"  
"12/11/2012 19:09:26","0.00"  
"12/11/2012 19:09:29","3.11"  
"12/11/2012 19:09:33","1.54"  
"12/11/2012 19:09:36","0.52"  
"12/11/2012 19:09:39","0.00"  
"12/11/2012 19:09:42","0.00"  
"12/11/2012 19:09:45","0.00"  
"12/11/2012 19:09:48","0.00"  
"12/11/2012 19:09:51","0.00"  
[/code]

Now let&#8217;s plot just the NFS read latency and see what we discover:

[code]  
[elatov@klaptop]$ gnuplot -p -e 'set grid;set xdata time; set timefmt "%m/%d/%Y %H:%M:%S"; set datafile sep ","; set y2tics autofreq; set format x "%H:%M" ; plot "nfs\_read\_latency_data" using 1:2 with lines title "Read Lat/s'  
[/code]

After running the above command I saw the following graph:

<a href="http://virtuallyhyper.com/2013/01/plot-esxtop-data-with-gnuplot/gnuplot_graph_nfs_latency/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://virtuallyhyper.com/2013/01/plot-esxtop-data-with-gnuplot/gnuplot_graph_nfs_latency/']);" rel="attachment wp-att-5378"><img src="http://virtuallyhyper.com/wp-content/uploads/2012/12/gnuplot_graph_nfs_latency.png" alt="gnuplot graph nfs latency Plot Esxtop Data With gnuplot" width="972" height="620" class="alignnone size-full wp-image-5378" title="Plot Esxtop Data With gnuplot" /></a>

So it looks like we had two spikes, one at about 19:10 and the other at 20:20. Let&#8217;s pick the second one and concentrate on that. First let&#8217;s find the exact time of the spike. It looks like the spike reached about 170ms. So let&#8217;s search for a value above a hundred and see if we can find our spike.

[code]  
\[elatov@klaptop]$ grep '1[0-9\]\[0-9\]' nfs\_read\_latency_data  
"12/11/2012 19:10:07","117.87"  
"12/11/2012 20:20:10","120.69"  
"12/11/2012 20:20:13","171.45"  
"12/11/2012 20:20:16","113.84"  
[/code]

So the spike was at 20:20 just like I thought. So let&#8217;s grab data just for that minute and plot it. First let&#8217;s grab the data for that minute:

[code]  
[elatov@klaptop]$ grep '20:20:' nfs\_read\_latency\_data > nfs\_read\_latency\_data\_20\_20  
[/code]

Next to confirm the data:

[code]  
[elatov@klaptop]$ cat nfs\_read\_latency\_data\_20_20  
"12/11/2012 20:20:00","0.00"  
"12/11/2012 20:20:03","0.00"  
"12/11/2012 20:20:06","90.91"  
"12/11/2012 20:20:10","120.69"  
"12/11/2012 20:20:13","171.45"  
"12/11/2012 20:20:16","113.84"  
"12/11/2012 20:20:19","0.00"  
"12/11/2012 20:20:22","0.00"  
"12/11/2012 20:20:25","0.00"  
"12/11/2012 20:20:28","0.00"  
"12/11/2012 20:20:31","0.00"  
"12/11/2012 20:20:35","4.10"  
"12/11/2012 20:20:38","0.00"  
"12/11/2012 20:20:41","16.73"  
"12/11/2012 20:20:44","4.38"  
"12/11/2012 20:20:47","5.63"  
"12/11/2012 20:20:50","0.00"  
"12/11/2012 20:20:53","0.00"  
"12/11/2012 20:20:56","0.00"  
[/code]

Lastly let&#8217;s plot the data:

[code]  
[elatov@klaptop]$ gnuplot -p -e 'set grid;set xdata time; set timefmt "%m/%d/%Y %H:%M:%S"; set datafile sep ","; set y2tics autofreq; set format x ":%M:%S" ; plot "nfs\_read\_latency\_data\_20_20" using 1:2 with lines title "Read Lat/ms'  
[/code]

Here is what I saw:

<a href="http://virtuallyhyper.com/2013/01/plot-esxtop-data-with-gnuplot/gnuplot_nfs_latency_2020/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://virtuallyhyper.com/2013/01/plot-esxtop-data-with-gnuplot/gnuplot_nfs_latency_2020/']);" rel="attachment wp-att-5379"><img src="http://virtuallyhyper.com/wp-content/uploads/2012/12/gnuplot_nfs_latency_2020.png" alt="gnuplot nfs latency 2020 Plot Esxtop Data With gnuplot" width="861" height="545" class="alignnone size-full wp-image-5379" title="Plot Esxtop Data With gnuplot" /></a>

Now usually the latency happens due to an increase in commands, but let&#8217;s confirm if that is the case. First let&#8217;s figure out the column number for &#8220;Reads/sec&#8221; is.

[code]  
[elatov@klaptop]$ grep -n NFS_Datastore headers | grep '\\Reads/sec'  
16386:"\\local\Physical Disk NFS Volume(NFS_Datastore)\Reads/sec"  
[/code]

So the column for that is &#8217;16386&#8242;. Now let&#8217;s grab all 3 columns and store them in a file:

[code]  
[elatov@klaptop]$ grep '20:20:' data | awk -F , '{print$1","$16386","$16391}' > nfs\_latency\_and\_nfs\_reads\_data\_20_20  
[/code]

To confirm the data, lets see a snippet of our subset:

[code]  
[elatov@klaptop]$ head nfs\_latency\_and\_nfs\_reads\_data\_20_20  
"12/11/2012 20:20:00","0.00","0.00"  
"12/11/2012 20:20:03","0.00","0.00"  
"12/11/2012 20:20:06","418.35","90.91"  
"12/11/2012 20:20:10","481.29","120.69"  
"12/11/2012 20:20:13","352.22","171.45"  
"12/11/2012 20:20:16","430.02","113.84"  
"12/11/2012 20:20:19","0.00","0.00"  
"12/11/2012 20:20:22","0.00","0.00"  
"12/11/2012 20:20:25","0.00","0.00"  
"12/11/2012 20:20:28","0.00","0.00"  
[/code]

That looks good, not let&#8217;s plot both columns in the same graph:

[code]  
[elatov@klaptop]$ gnuplot -p -e 'set grid;set xdata time; set timefmt "%m/%d/%Y %H:%M:%S"; set datafile sep ","; set y2tics autofreq; set format x ":%M:%S" ; plot "nfs\_latency\_and\_nfs\_reads\_data\_20\_20" using 1:3 with lines title "Read Lat/ms"; replot "nfs\_latency\_and\_nfs\_reads\_data\_20\_20" using 1:2 with lines title "Read/s'  
[/code]

And here was the graph that I saw:

<a href="http://virtuallyhyper.com/2013/01/plot-esxtop-data-with-gnuplot/gnuplot_nfs_lat_nfs_reads/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://virtuallyhyper.com/2013/01/plot-esxtop-data-with-gnuplot/gnuplot_nfs_lat_nfs_reads/']);" rel="attachment wp-att-5381"><img src="http://virtuallyhyper.com/wp-content/uploads/2012/12/gnuplot_nfs_lat_nfs_reads.png" alt="gnuplot nfs lat nfs reads Plot Esxtop Data With gnuplot" width="912" height="541" class="alignnone size-full wp-image-5381" title="Plot Esxtop Data With gnuplot" /></a>

We can clearly see an increase in the reads sent and therefore the read latency increased. Now to find out why the increase in reads. Let&#8217;s check for all the virtual disks (that are owned by VMs) and their reads at the time of the spike. First here are all the columns for all the Virtual Disk and their reads per second:

[code]  
[elatov@klaptop]$ grep -n 'Virtual Disk' headers | grep 'Reads/sec'  
16470:"\\local\Virtual Disk(SQL02 (df09cdf2-1e74-45fb-9ccc-90d1c402fc6b))\Reads/sec"  
16477:"\\local\Virtual Disk(SQL02 (df09cdf2-1e74-45fb-9ccc-90d1c402fc6b):scsi0:0)\Reads/sec"  
16484:"\\local\Virtual Disk(SQL02 (df09cdf2-1e74-45fb-9ccc-90d1c402fc6b):scsi0:1)\Reads/sec"  
16491:"\\local\Virtual Disk(SQL02 (df09cdf2-1e74-45fb-9ccc-90d1c402fc6b):scsi0:2)\Reads/sec"  
16498:"\\local\Virtual Disk(SQL02 (df09cdf2-1e74-45fb-9ccc-90d1c402fc6b):scsi0:3)\Reads/sec"  
16505:"\\local\Virtual Disk(SQL02 (df09cdf2-1e74-45fb-9ccc-90d1c402fc6b):scsi0:4)\Reads/sec"  
16554:"\\local\Virtual Disk(SQL01 (104c07ff-3663-44fb-9d89-a563c801611c))\Reads/sec"  
16561:"\\local\Virtual Disk(SQL01 (104c07ff-3663-44fb-9d89-a563c801611c):scsi0:0)\Reads/sec"  
16568:"\\local\Virtual Disk(SQL01 (104c07ff-3663-44fb-9d89-a563c801611c):scsi0:1)\Reads/sec"  
16575:"\\local\Virtual Disk(SQL01 (104c07ff-3663-44fb-9d89-a563c801611c):scsi0:2)\Reads/sec"  
16582:"\\local\Virtual Disk(SQL01 (104c07ff-3663-44fb-9d89-a563c801611c):scsi0:3)\Reads/sec"  
16589:"\\local\Virtual Disk(SQL01 (104c07ff-3663-44fb-9d89-a563c801611c):scsi0:4)\Reads/sec"  
16596:"\\local\Virtual Disk(SQL01 (104c07ff-3663-44fb-9d89-a563c801611c):scsi0:5)\Reads/sec"  
16617:"\\local\Virtual Disk(Linux (867c62db-267d-41a1-baaa-17d2a5538baf))\Reads/sec"  
16624:"\\local\Virtual Disk(Linux (867c62db-267d-41a1-baaa-17d2a5538baf):scsi0:0)\Reads/sec"  
[/code]

Now search for all of those columns at our time, we see the following:

[code]  
[elatov@klaptop]$ grep '20:20:13' data | awk -F , '{print $1","$16470","$16477","$16484","$16491","$16498","$16505","$16554","$16561","$16568","$16575","$16582","$16589","$16596","$16617","$16624}'  
"12/11/2012 20:20:13","6.36","0.00","6.36","0.00","0.00","0.00","345.87","0.00","345.87","0.00","0.00","0.00","0.00","0.00","0.00"  
[/code]

We can see that columns &#8217;16554&#8242; and &#8217;16568&#8242; have a high amount of commands sent at the time of our spike. Those columns correspond to the following virtual disks:

[code]  
16554:"\\local\Virtual Disk(SQL01 (104c07ff-3663-44fb-9d89-a563c801611c))\Reads/sec"  
16568:"\\local\Virtual Disk(SQL01 (104c07ff-3663-44fb-9d89-a563c801611c):scsi0:1)\Reads/sec"  
[/code]

This VM was a SQL server, it had huge queries which would take a while and would cause the latency. After talking to the DBA, they optimized their queries and they didn&#8217;t take that much time any more. With that in place, the latency subsided. 

Now to break down our gnuplot command:  
[code]  
set grid;set xdata time; set timefmt "%m/%d/%Y %H:%M:%S"; set datafile sep ","; set y2tics autofreq; set format x ":%M:%S" ; plot "nfs\_latency\_and\_nfs\_reads\_data\_20_20" using 1:3 with lines title "Read Lat/ms"  
[/code]

The information was taken from &#8220;<a href="http://www.gnuplot.info/docs_4.4/gnuplot.pdf" onclick="javascript:_gaq.push(['_trackEvent','download','http://www.gnuplot.info/docs_4.4/gnuplot.pdf']);">GnuPlot Version 4.4 Documentation</a>&#8221;

**set grid:** 

> The set grid command allows grid lines to be drawn on the plot. 

**set xdata time**

> gnuplot supports the use of time and/or date information as input data. This feature is activated by the commands set xdata time, set ydata time, etc. 

**set timefmt**

> This command applies to time series where data are composed of dates/times. It has no meaning unless the command set xdata time is given also.  
> Syntax:  
> [code]  
> set timefmt "<format string>"  
> show timefmt  
> [/code]  
> The string argument tells gnuplot how to read time data from the data file. The valid formats are:
> 
> <a href="http://virtuallyhyper.com/2013/01/plot-esxtop-data-with-gnuplot/gnuplot_time_formate/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://virtuallyhyper.com/2013/01/plot-esxtop-data-with-gnuplot/gnuplot_time_formate/']);" rel="attachment wp-att-5382"><img src="http://virtuallyhyper.com/wp-content/uploads/2012/12/gnuplot_time_formate.png" alt="gnuplot time formate Plot Esxtop Data With gnuplot" width="418" height="231" class="alignnone size-full wp-image-5382" title="Plot Esxtop Data With gnuplot" /></a> 

Here is an example from the documentation:

> The following example demonstrates time/date plotting.  
> Suppose the fi le &#8220;data&#8221; contains records like  
> 03/21/95 10:00 6.02e23  
> This file can be plotted by  
> [code]  
> set xdata time  
> set timefmt "%m/%d/%y"  
> set xrange ["03/21/95":"03/22/95"]  
> set format x "%m/%d"  
> set timefmt "%m/%d/%y %H:%M"  
> plot "data" using 1:3  
> [/code] 

**set datafile sep**

> The command *set data file separator &#8220;char&#8221;* tells gnuplot that data fields in subsequent input fi les are separated by &#8216;char&#8217; rather than by whitespace. The most common use is to read in csv (comma-separated value) fi les written by spreadsheet or database programs. 

**set y2tics autofreq**

> Positions of the tics are calculated automatically by default or if the autofreq option is given 

**set format x &#8220;:%M:%S&#8221;**

> In time/date mode, the acceptable formats are:
> 
> <a href="http://virtuallyhyper.com/2013/01/plot-esxtop-data-with-gnuplot/gnuplot_time_format_tics/" onclick="javascript:_gaq.push(['_trackEvent','outbound-article','http://virtuallyhyper.com/2013/01/plot-esxtop-data-with-gnuplot/gnuplot_time_format_tics/']);" rel="attachment wp-att-5383"><img src="http://virtuallyhyper.com/wp-content/uploads/2012/12/gnuplot_time_format_tics.png" alt="gnuplot time format tics Plot Esxtop Data With gnuplot" width="357" height="445" class="alignnone size-full wp-image-5383" title="Plot Esxtop Data With gnuplot" /></a>
> 
> Except for the non-numerical formats, these may be preceded by a &#8220;0&#8243; (&#8220;zero&#8221;, not &#8220;oh&#8221;) to pad the fi eld length with leading zeroes, and a positive digit, to de fine the minimum field width (which will be overridden if the specifi ed width is not large enough to contain the number). There is a 24-character limit to the length of the printed text; longer strings will be truncated.  
> Examples:  
> Suppose the text is &#8220;76/12/25 23:11:11&#8243;. Then  
> [code]  
> set format x # defaults to "12/25/76" \n "23:11"  
> set format x "%A, %d %b %Y" # "Saturday, 25 Dec 1976"  
> set format x "%r %D" # "11:11:11 pm 12/25/76"  
> [/code]  
> Suppose the text is &#8220;98/07/06 05:04:03&#8243;. Then  
> [code]  
> set format x "%1y/%2m/%3d %01H:%02M:%03S" # "98/ 7/ 6 5:04:003"  
> [/code] 

**plot &#8220;data&#8221; using 1:3 with lines title &#8220;Read Lat/ms&#8221;**

> The most common data file modifi er is using.  
> Syntax:  
> [code]  
> plot 'file' using {<entry> {:<entry> {:<entry> ...}}} {'format'}  
> [/code]  
> Each &#8216;entry&#8217; may be a simple column number that selects the value from one fi eld of the input t, an expression enclosed in parentheses, or empty. 

More information from the same document:

> There are many plotting styles available in gnuplot. They are listed alphabetically below. The commands set style data and set style function change the default plotting style for subsequent plot and splot commands.
> 
> You also have the option to specify the plot style explicitly as part of the plot or splot command. If you want to mix plot styles within a single plot, you must specify the plot style for each component.  
> Example:  
> [code]  
> plot 'data' with boxes, sin(x) with lines  
> [/code]  
> Each plot style has its own expected set of data entries in a data fi le. For example by default the lines style expects either a single column of y values (with implicit x ordering) or a pair of columns with x in the fi rst and y in the second. 

<p class="wp-flattr-button">
  <a class="FlattrButton" style="display:none;" href="http://virtuallyhyper.com/2013/01/plot-esxtop-data-with-gnuplot/" title=" Plot Esxtop Data With gnuplot" rev="flattr;uid:virtuallyhyper;language:en_GB;category:text;tags:Average Guest MilliSec/Read,awk,CSV,esxtop,esxtop batch mode,gnuplot,head,headers,Reads/sec,set format x,set timefmt,tail,blog;button:compact;">Let&#8217;s say you collected esxtop batch data per the instructions laid out in VMware KB 1004953. So in the end you just had a huge CSV (Comma Separated Values) file, with...</a>
</p>