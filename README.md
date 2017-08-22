#clogin-wrapper

`clogin-wrapper` is a wrapper for the builtin `clogin` script included in the Rancid tools. This script loops through a file of IPs and runs the individual built `clogin` command. The script has an auto timeout of 3 minutes, incase the `clogin` scripts gets stuck. I came across this on IBM switches. If the running config hasn't been saved, the IBMs will come up with a warning when trying to close the connection to the switch
```WARNING: The running-config is different to startup-config.
Confirm operation without saving running-config to startup-config (y/n) ?```
This warning breaks `clogin`. Now there are ways to go fix that with `clogin` but that could also cause other issues. Maybe you are just doing show commands and don't know if you should save the config. It's just better to have `clogin` timeout if it gets stuck.

# Setup

1. Make `clogin-wrapper.sh` executable
  * `chmod +x clogin-wrapper.sh`

# Building Config Changes
We are going to build the text file of IPs. This is what the `clogin-wrapper` loops through.

`vim ips.txt`

Build your list of IPs, one IP per-line

```
10.0.99.47
10.0.99.51
10.0.99.52
10.0.99.53
10.0.99.54
10.0.99.55
10.0.99.56
10.0.99.57
10.0.99.58
```

Now we need a text file of the commands you want ran on the switches

`vim cmds.txt`

Type out the commands you want to run, make sure each command is one line at a time. Your text file should look as if you're typing directly on the switch. Each line is a new command. For example if you want to add a new vlan on all switches in your text file, it would look like this:

```
conf t
vlan 100
name new_vlan
exit
exit
wr mem
```
It's best practice to always end your commands so that you return to the main switch prompt `hostname#`. So for the above example I needed to `exit` twice since I was in the vlan config block. You don't need to worry about logging off of the switch, `clogin` will take care of that for you. 

For example, only show commands:

```
show version
show mac address-table count 
```
The above command file will execute those commands and save the output to a file. I'll talk about the output files in the next section.

# Running the script
Now we are ready to run the `clogin-wrapper.sh` script.

`./clogin-wrapper.sh -c cmds.txt -i ips.txt -s /usr/libexec/rancid/clogin`

That will run the commands listed in the `cmds.txt` file on all of the IPs in your `ips.txt` file. Running it like this will take away your terminal until the script is done. If you only have a few commands and switches it should only take a few minutes, but if you have lots of switches this can take a long time to run. It's better to run this script in the background. That way you can exit out of your ssh session with rancid, and the `clogin-wrapper.sh` script will continue to run. You can either run the command with `screen` or the `nohup` command. 

For example: `nohup ~/clogin-wrapper.sh -c cmds.txt -i ips.txt -s /usr/libexec/rancid/clogin &`

If you run the command like this you have to specify the full path to the `clogin-wrapper.sh` script.

### Output
There will be a text file created for each switch. The file contains all output of the switch from all of the commands ran. Even commands that didn't change the config will have an output in the output file. It looks like the terminal if you had ran the commands yourself. You can look at the file and see exactly what `clogin` did. This is helpful if your commands didn't do what you thought you did. If you are running this on a lot of switches you might want to specify a directory for all of these output files to go in.

`mkdir switch_outputs`

Then your `clogin-wrapper` command would look like this

`./clogin-wrapper.sh -c cmds.txt -i ips.txt -o switch_outputs/ -s /usr/libexec/rancid/clogin`

All of the output files will be in the directory `switch_outputs`. The output files are named like `ip.output`. Now you can look in the `switch_outputs` directory and see all of the output files.

```
[netops-jonathansmith@rancid ~]$ ls switch_outputs/
10.0.99.47.output
10.0.99.51.output
10.0.99.52.output
10.0.99.53.output
10.0.99.54.output
10.0.99.55.output
10.0.99.56.output
10.0.99.57.output
10.0.99.58.output
```

### clogin location
You must specify the location of the `clogin` script. By default the `clogin-wrapper` will try to execute in the current directory it is located in. That's only if you have copied the clogin and `.cloginrc` files to the same directory as the `clogin-wrapper` script.

It's a lot easier to just use the rancid clogin script. The location of rancid's clogin script is `/usr/libexec/rancid/clogin.` You can specify the path to `clogin` when executing the `clogin-wrapper` script with the `-s` option.

### Other clogin-wrapper options

Here is the output of the `clogin-wrapper.sh` script, you can see all of the different options.

```
[netops-jonathansmith@rancid ~]$ ./clogin-wrapper.sh 
Usage: ./clogin-wrapper.sh [-c TEXT] [-i TEXT] [-o TEXT] [-s TEXT] [-t 30-1200]
-c path to text file containing commands to run
-i path to text file containing the IPs
-o path to directory for output files. Output files will be ip.output
-s path to clogin script if not in same directory as this script
-t timeout in seconds. This is if clogin can't properly logout of the switch default is 180(3 mins)
-v verbose mode
Example: ./clogin-wrapper.sh -c cmds-to-run.txt -i ips.txt -o switch_outputs -s /path/to/clogin -t 300
```


# Monitoring the script

There are several different ways you can monitor the progress of the script. The best way is to just see how many output files have been made `ls switch_outputs/ | wc -l`. This will return how many files are in the `switch_output` directory. If you want to see where the script is at run `ps -ef`. That will return a slimmed down list of the current processes that are running, you'll be able to see what switch the script is running on.

```
[netops-jonathansmith@rancid test]$ ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
jonathansmith   3932  4309  0 15:06 pts/0    00:00:00 /bin/sh /admin/jonathansmith/test/clogin-wrapper.sh -c cmds.txt -i ips.txt -o switch_ou
jonathansmith   3933  3932  0 15:06 pts/0    00:00:00 /usr/bin/expect -- ../clogin -x cmds 10.0.99.54
jonathansmith   3934  3933  0 15:06 pts/0    00:00:00 /bin/sh /admin/jonathansmith/test/clogin-wrapper.sh -c cmds.txt -i ips.txt -o switch_ou
jonathansmith   3935  3934  0 15:06 pts/0    00:00:00 sleep 180
jonathansmith   3936  3933  0 15:06 pts/1    00:00:00 telnet 10.0.99.54
jonathansmith   3939  4309  0 15:06 pts/0    00:00:00 ps -ef
jonathansmith   4309  4308  0 11:06 pts/0    00:00:00 -bash
```

You can see there that the script is on 10.0.99.54

# Important Notes
The most important thing you can do is make sure your commands are correct. Once the script starts it will execute all commands in the file. You can stop it if something is wrong, but it's better to not have anything wrong. You can run `kill -9 3932` to kill the script. `3932` is the PID of the script, you can get this by running `ps -ef`. See the **Monitoring** section for example of `ps -ef`.

The best thing you can do is, be certain your commands are corrects. **Run your commands on a single switch doing it yourself to make sure you have them correct.**    