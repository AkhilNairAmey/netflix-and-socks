# Routing web traffic through a different country using SSH

_This has been written for educational purposes of learning about net
traffic routing._

_NOTE: Yes, this, and more, all can be achieved through a private VPN or VPN
provider. SSH is quicker to set up than a private VPN and I'd rather go private
than use a provider to route my webtraffic_

This basic guide will outline the steps to set up a **SOCKS5** proxy over `ssh`
on an **Amazon Web Services** (AWS) virtual machine instance in the US to
forward net traffic from a machine with a foreign IP address to the host
machine.  This guide assumes a linux host, though most of the set up is done on
AWS. The only steps to change occur at the end, where putty will be needed to
set up the tunnels.

The service is free for a year provided you don't run any other AWS instances on
the account that is used to set up the AWS instance. The AWS 'free tier'
provides you with 750 hours a month (`31 * 24 = 744`) of time on a linux machine
for 12 months, after which it is `$0.013` an hour. Or you can just set up
another AWS account.

The basic steps are outlined below
 - Sign up for an AWS account
 - Launch a box in a US availability zone, or any other country
 (`Ubuntu 16.04` image)
 - Allocate the machine an **elastic IP**
 - Add aliases to:
  - Easily connect to the server via `ssh`
  - Set up a background `ssh` tunnel using the instance as a proxy
  - Run a browser which connects to the web through the proxy
  - Kill the background `ssh` tunnel

At this point the set up works, and you can check your ip address is from the US
by googling 'what's my IP'.

### AWS

AWS have availability zones in the US available to everyone, i.e. a customer can
provision server space in the US by setting up a machine instance in a US
availability zone.  An example of why this might be used is to minimise latency
when sending data back to the majority of clients if the user base is in the US.

If, then, this machine makes a request for us, the data is requested by a US
client.  We can read the data from the virtual machine, and so 'forward' the
traffic to the UK.

 - Sign up to AWS [here](https://aws.amazon.com/console/)
 - Follow the wizard to launch a free EC2 instance in a US region
   - AWS is fairly generous with marking out exactly which options are free
 - Generate a `.pem` key to set the instance up with and download it
   - If you're familiar with ssh keys, feel free to continue on to the next
   section.

#### Bonus `ssh`

AWS servers do not have paswords, and can only be accessed via **ssh** (a
**s**ecure **sh**ell). When creating an AWS instance, the creator is also given
a **.pem** file, which contains a key to log into the machine.  To log into the
machine, you can either continue to use the pem key, or use/generate your own
public/private keypair, after using the pem key to place your public key on the
server.

 - Generate an ssh keypair using [this](http://www.linuxproblem.org/art_9.html)
 quick quide
 - Log into the into the AWS instance using `ssh` with the syntax `ssh -i /path/to/pem user@host`
  - `host` is the public IP address of the machine, found by clicking the
  instance in the AWS console and finding 'Public IP' in the details
   - The default amazon `user` is `ubuntu`
   - `/path/to/pem` should point to your pem key
   - Hence an example command is `ssh -i ~/.ssh/key.pem ubuntu@54.192.170.23`
 - Once on the remote machine, place your normal public key (`key.pub`) in `~/.ssh/authorized_keys`.

Alternatively, always use the pem key!

### Elastic IP

An elastic IP is a static IP address provided by AWS that is attached to an
instance, such that networking protocols can easily point to the host.  *You
pay for an elastic IP adresses for all the time the address is allocated, but
not attached to a running machine*.  If an address is allocated and attached to
an instance that is always on, the elastic IP is free.

## On the Host (assuming a Ubuntu host)

We add an `ssh` alias for ease of using `ssh`. Instead of the lengthy command that
was necessary previously, after adding an alias, we can log on to our remote
machine through `ssh` simply by typing `ssh <alias>`.  Aliases are added by
adding new hosts (and configuring them) to the `~/.ssh/config` file. Copy and paste
the below block into the host terminal, with the URL you chose for your remote
machine through the DDNS.

```
# Add an ssh alias on host
echo "Host US-Jumpbox" >> ~/.ssh/config
echo "  Hostname yupimaki.ddns.net" >> ~/.ssh/config
echo "  User ubuntu" >> ~/.ssh/config
```

This will set up an alias such that `ssh US-Jumpbox` routes to
`ssh ubuntu@yupimaki.ddns.net`, to `ssh ubuntu@XX.XXX.XXX.XX`, where the IP
address will have been recently updated by the docker container running on the
remote instance.  If you didn't add a public key to the remote machine,
and wish to keep using the `.pem` file, add an extra line to `~/.ssh/config`
with `IdentityFile path/to/key.pem`.

## Final Aliases on Host

_NOTE: If your host machine isn't running linux, 1) wot r u doin, 2) please
refer to the link in the references to crack these last steps on other OSes_

Finally, we need an easy way to set up our ssh tunnel to the proxy server and
launch a browser which is configured to connect to the web through the proxy.

I find the easiest way (by no means the correct way) to do this is to add some
lines to my `~/.bashrc` file.  

```
# Easy way to kill a process if running on an unknown port
port-kill () {
   # Find offending process's port
   {
      # Truthy hack to handle error if no process with that name is found
      ps -C $1 -o pid= &> /dev/null && port=$(ps -C $1 -o pid=)
   } || {
      echo "No process found with name $1"
      return
   }

   # Get process name by port ID
   process_name=$(ps -p ${port} -o comm=)

   # Option to kill found process
   read -r -p "Are you sure you want to kill ${process_name}? [y/N] " response
   response=${response,,}    # tolower
   if [[ $response =~ ^(yes|y)$ ]]
   then
      kill ${port}
   fi
}

## Run and kill commands
# Must not have a google-chrome session open.
# Seems to be a bug in chrome that you can't force start with a new session
alias netflix-us="ssh -D 1089 -f -C -q -N US-Jumpbox && google-chrome --proxy-server='socks5://localhost:1089'"
alias kill-ssh="port-kill ssh"
```

Breaking down the final aliases, `netflix-us` is composed of two statements.

```
ssh -D 1089 -f -C -q -N US-Jumpbox
```

We already know what `ssh US-Jumpbox` does. Explaining the other flags;
 - `-D 1089` sets up [D]ynamic port forwarding on the local port `1089` through
 `ssh`
 - `-f` [f]orks the process and runs it in the background
 - `-C` [C]ompresses the data before sending it through the `ssh` pipe
 - `-q` sends all `STDOUT` and `STDERR` messages to `/dev/null`, i.e. it is
 [q]uiet

```
google-chrome --proxy-server='socks5://localhost:1089'"
```

This command simply starts google chrome, however specifies that it connects to
the web through a proxy server.  The server is our `localhost` at port `1089`,
which we know is dynamically routing our web traffic through the US.

Now all you have to do to watch US Netflix (whilst respecting copyright,
[provided you live in the US]) is to run the command `us-netflix` in the terminal.
Unfortunately there seems to be a small bug in `google-chrome`, where you cannot
force the browser to run in a new session, so for the routing to work, `chrome`
cannot already be open. To get around this, use one browser reading traffic
through the proxy, and other for general surfing.

When done, tear down the tunnel using `kill-ssh`, or simply leave it running.

Thanks for reading,

Akhil

### References
 - Route traffic using SOCKS tunnel
   - https://www.digitalocean.com/community/tutorials/how-to-route-web-traffic-securely-without-a-vpn-using-a-socks-tunnel
