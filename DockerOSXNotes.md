# Source:
https://gist.github.com/bradleyfrank/561a04086ee69118dbbc57bc074d85eea

# How it works

This is akin to what docker desktop does: 
  - Start a linux VM 
  - Share filesystem between OSX and linux 
  - Run dockerd in the linux VM
 
The docker version is likely more fined-tune. 


# Setup `docker-cli`

## Uninstall docker-dektop (optional) and install cli 
```bash
# uninstall docker-desktop and install docker-cli
brew uninstall --cask docker && brew install --formula docker
```

# Setup a Linux hosting dockerd

## Recommend: using docker-machine

This can use either virtualbox of VMWare fusion

```bash
brew install docker docker-machine
brew services start docker-machine
```

### VirtualBox 

#### Installation
```bash
brew install virtualbox
```

You need to approve Oracle VirtualBox in System Settings, reboot machine to activate new kernel extensions

virtualbox requires a kernel extension to work.
If the installation fails, retry after you enable it in:
  System Preferences → Security & Privacy → General

For more information, refer to vendor documentation or this Apple Technical Note:
  https://developer.apple.com/library/content/technotes/tn2459/_index.html

#### Create the VM

```bash
docker-machine create --driver virtualbox --virtualbox-no-share --virtualbox-memory 4096 --virtualbox-cpu-count 2 docker-default
```

### VMWare 

#### Installation

```bash
brew install docker-machine-driver vmware
```

#### Create the VM

```bash
docker-machine create --driver vmware --vmware-no-share --vmware-memory-size 4096 --vmware-cpu-count 2 docker-default
```

## Docker client configuration

This tells the docker client which dockerd to use

```bash
docker-machine env docker-default 
eval $(docker-machine env docker-default)
```

To make it permanent when starting a shell:
```bash
echo "eval \$(docker-machine env docker-default)" >> ~/.zshrc
```




## Manual VM creation with multipass hypervisor

Multipass provides ubuntu VM. Its younger and less stable. FileSystem Sharing is not very performant

### Installation

```bash
brew install multipass
```

#### Create the VM
```bash
multipass launch -c 4 -m 8G -d 10G -n docker
multipass exec docker -- sudo bash -c '\
  apt-get update && \
  apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release && \
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
    | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg && \
  echo \
    "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
    https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
    | sudo tee /etc/apt/sources.list.d/docker.list && \
  sudo apt-get update && \
  sudo apt-get install -y docker-ce docker-ce-cli containerd.io && \
  mkdir -p /etc/systemd/system/docker.service.d && \
  printf "[Service]\nExecStart=\nExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375" \
    > /etc/systemd/system/docker.service.d/override.conf && \
  systemctl daemon-reload && \
  systemctl restart docker
'

### Docker client configuration

This tells the docker client which dockerd to use

```bash
# configure local docker client to use the multipass vm
docker context create multipass --docker \
  "host=tcp://$(multipass list --format json \
    | jq -r '.list[]|select(.name=="docker").ipv4[0]'):2375"
docker context use multipass
```

### Running out of disk in the linux VM:

The following might help:
https://github.com/canonical/multipass/issues/62



# Testing the setup

```
docker run -i -t alpine sh 
```


# FileSystem sharing 

you must share your OSX filesystem with the linux VM 
for instance, if all your dev-data is under `/Users/user-local/projects/`

## VirtualBox 

TODO

## VMWare

TODO

## Multipass
```bash
multipass mount /Users/user-local/projects/ docker 
```

The backend uses sshfs, which should not be terribly efficient. 
If the filesystem is not exported, the docker commands may block.


## Test FS sharing
```
docker pull alpine
docker run -it -v $PWD:/tests  alpine /bin/sh
cd /tests
ls
touch i-am-from-another-namespace
echo "test" >> i-am-from-another-namespace-2
```

you should see the files



# Enterprise caveats

Your company may mandate SSL MIDM filtering, in which case you may want to do the additional steps.
  - From OSX Keychain access, export the Root CA of the SSL MIDM component and export it as crt
  - copy it to the linux vm `/usr/local/share/ca-certificates/`
  - run update-ca-certificates

That step is not required if all the servers you access are using normal CA 
that can be checked by running `openssl s_client -connect server:443` in the linux VM 
