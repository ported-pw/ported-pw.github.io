---
published: true
---
#### How to fix transparency issues in GNOME, add temperature monitors and an unread email counter.

Conky is a really great tool when it comes to monitoring your system and spicing up your boring desktop in the process. 
### The Transparency
When I found out about it in [this great video](https://www.youtube.com/watch?v=XTkYUUfewck), I immediately wanted to try it out, so I installed Conky Manager, enabled a \"widget\", and was greated by this:
![]({{site.baseurl}}/images/2016-02-13/before.png)
Great. No transparency and things are overlaying everywhere. After some research, I found [this section](https://wiki.archlinux.org/index.php/Conky#Integrate_with_GNOME) in the Arch Wiki. I ended up replacing this `own_window` section

~~~
own_window yes
own_window_type normal
own_window_transparent yes
own_window_hints undecorated,below,sticky,skip_taskbar,skip_pager
own_window_colour 000000
own_window_argb_visual no
own_window_argb_value 0
~~~

with this one

~~~
own_window yes
own_window_type desktop
own_window_transparent yes
own_window_colour 000000
own_window_argb_visual yes
own_window_argb_value 255
own_window_hints undecorated,below,sticky,skip_taskbar,skip_pager
~~~

And there you go:
![]({{site.baseurl}}/images/2016-02-13/after.png)
Much better!

### Hard drive temperatures
After I finished fixing the conky widgets that came with conky-manager, I started customizing them. I wanted to display temperatures. For hard drives, this is quite easy, since there is a `hddtemp` conky variable that accepts a block device like `/dev/sda/` as a parameter. However, since calling `hddtemp` in the terminal requires root permissions and conky should never run as root, we need to use its daemon. Enable and start it
~~~bash
sudo systemctl enable hddtemp
sudo systemctl start hddtemp
~~~
and you will see your hard drive temperature show up. However, when trying to access anything else than `/dev/sda`, conky will display `N/A`. This is because the `hddtemp` daemon only watches `/dev/sda` by default. You can change this by editing its service file:
~~~bash
sudo systemctl edit hddtemp
~~~
This will open a [drop-in snippet](https://wiki.archlinux.org/index.php/Systemd#Drop-in_snippets) in your favorite text editor.  Edit it to override the `ExecStart` directive and add more hard drives to monitor like this:
~~~bash
[Service]
ExecStart=
ExecStart=/usr/bin/hddtemp -dF /dev/sda /dev/sdb
~~~
You should now be able to access all hard drives named in the service file.

### Other temperatures
For other temperatures, like CPU or GPU, it is not that easy. There are at least two ways of doing this: Parsing the output of a tool like `lm_sensors` (`sensors` command) or writing scripts directly accessing them. I am going to cover the second method here.   
I did some digging on the interwebs and found out that on Arch you can find most temperatures under `/sys/bus/pci/drivers/`. When digging around in there, I found the `k10` (AMD CPU temperature chip) and the `radeon` driver folders. Under an odd named folder (probably PCIe adress) starting with `0000` you should find the `hwmon` subfolder. Dig around in there, and you will find files named `temp1_input` or similar. Try some of them out using `cat` and check them against `sensors`. When you have found the path, you can include it in a script like this:

~~~bash
#!/bin/bash
echo $(($(cat /sys/bus/pci/drivers/radeon/0000\\:01\\:00.0/hwmon/hwmon2/temp1_input) / 1000))
~~~

However, I had an occasion where the CPU temperature was moving back and forth `hwmon/hwmon0` and  `hwmon/hwmon1` on each reboot. To counter that, use globbing:

~~~bash
#!/bin/bash
echo $(($(cat /sys/bus/pci/drivers/k10temp/0000\\:00\\:18.3/hwmon/hwmon[0,1]/subsystem/hwmon3/temp2_input) / 1000))
~~~

When your scripts work, you can include them in conky like this:

~~~
${execi 10 /path/to/your/script.sh}
~~~
where `10` is the time in between script calls in seconds.

### Unread emails

At one point, I wanted to add an unread email counter to conky. Since I use IMAP and no local mailbox, I couldn\'t use conkys variables for that task. Luckily I found [this script](http://www.unix.com/302337795-post1.html) written in perl that works perfectly when included like shown above. But it is only for one account. Fort that purpose, I had to quickly learn some perl array stuff and came up with this script:

~~~perl
#!/usr/bin/perl

# gimap.pl by gxmsgx
# modified by chameleon for multiple accounts
# description: get the count of unread messages on gmail imap

use strict;
use Mail::IMAPClient;
use IO::Socket::SSL;

my @account1 = (\'host\', port, \'username\', \'password\');
my @example = (\'imap.gmail.com\', 993, \'max@gmail.com\', \'password\');
my @accounts = ([@gmail1],[@example]);

my $i = 0;
my $unseen_sum = 0;
while($i < scalar @accounts) {
  my $socket = IO::Socket::SSL->new(
     PeerAddr => $accounts[$i][0],
     PeerPort => $accounts[$i][1],
    )
    or die \"socket(): $@\";

  my $client = Mail::IMAPClient->new(
     Socket   => $socket,
     User     => $accounts[$i][2],
     Password => $accounts[$i][3],
    )
    or die \"new(): $@\";

  if ($client->IsAuthenticated()) {
      my $msgct;

      $client->select(\"INBOX\");
      $msgct = $client->unseen_count||\'0\';
      $unseen_sum += $msgct;
  }

  $client->logout();
  $i++;
}
print \"$unseen_sum\
\";
~~~

### Resources
https://wiki.archlinux.org/index.php/Conky  
https://github.com/brndnmtthws/conky/wiki/Configure

I hope you liked this article and, if you have any suggestions on what to add here or corrections, write a comment :)

*Note: This article was transferred over from my old website in June, 2018*
