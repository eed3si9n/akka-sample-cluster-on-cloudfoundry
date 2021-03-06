This repo contains sample Akka Cluster app along with instructions how to deploy it to CloudFoundry.

# Game plan:

* Install bosh-lite locally https://github.com/cloudfoundry/bosh-lite
* (Optional) Deploy Standalone (non cluster) Akka app
* Install networking plugin https://github.com/cloudfoundry-incubator/netman-release
* (Optional) Deploy 'Cats and Dogs' sample app to verify the installation and learn how to enable inter-container communication: https://github.com/cloudfoundry-incubator/netman-release/tree/master/src/example-apps/cats-and-dogs
* Deploy Service Registry software amalgam8 - instructions are at https://github.com/cloudfoundry-incubator/netman-release/tree/develop/src/example-apps/tick
* Deploy akka-sample-cluster app, see instruction [below](#deploy-akka-cluster)

## Note: the rest of README are instructions from links above as I executed them along with potential gotchas that I've encountered. I'd advise to follow the official links and read the instructions below to compare if needed.

# Install bosh cli
```
gem install bosh_cli --no-ri --no-rdoc
```

# Install bosh lite:

## Install vagrant
https://www.vagrantup.com/downloads.html

## Clone bosh lite
```
git clone https://github.com/cloudfoundry/bosh-lite
```

## Install virtualbox
https://www.virtualbox.org/wiki/Downloads

## virtual box create
```
cd bosh-lite
vagrant up --provider=virtualbox
```

### Note: if you encounter `default: Warning: Authentication failure. Retrying...` that goes on forever, 
then use the following fix: 

```
When I log onto the VM manually, I noticed the updated authorized_keys file permission that vagrant installed is incorrect. When I 'chmod 0600 ~/.ssh/authorized_keys' that file, it then starts to work.
```

[More info](https://github.com/mitchellh/vagrant/issues/7610) and don't forget to restart the vagrant.

I followed it like this: 
```
vagrant ssh # (default password: vagrant)

vagrant@agent-id-bosh-0:~$ ls -la ~/.ssh/authorized_keys
-rw-rw-r-- 1 vagrant vagrant 389 Sep 21 15:27 /home/vagrant/.ssh/authorized_keys
vagrant@agent-id-bosh-0:~$ chmod 0600 /home/vagrant/.ssh/authorized_keys
vagrant@agent-id-bosh-0:~$ ls -la ~/.ssh/authorized_keys
-rw------- 1 vagrant vagrant 389 Sep 21 15:27 /home/vagrant/.ssh/authorized_keys

vagrant suspend
vagrant up
```

## Login into bosh director
```
bosh target 192.168.50.4 vagrant # credentials: admin/admin
```

## Adding routes
```
bin/add-route
```

# Install CF 

## Follow the instructions [deploy cloud foundry](https://github.com/cloudfoundry/bosh-lite/blob/master/README.md#deploy-cloud-foundry)

```
cd .. # be at the same level as bosh-lite
git clone https://github.com/cloudfoundry/cf-release
./bin/provision_cf
```

## It will complain about bundler
```
gem install bundler
./bin/provision_cf
```

## It will complain about spiff
For Mac install Darwin [here](https://github.com/cloudfoundry-incubator/spiff/releases)
```
unzip spiff_darwin_amd64.zip
mkdir bin
mv spiff binvi ~/.bash_profile # add $pwd/bin to PATH
source ~/.bash_profile
```

## Here should be success
```
./bin/provision_cf
```

# Install CF CLI 
https://github.com/cloudfoundry/cli#downloads
```
curl -L "https://cli.run.pivotal.io/stable?release=macosx64-binary&source=github" | tar -zx
mv cf bin/
```

### Comment: check if cluster is running properly
```
bosh cck cf-warden
```

# Build / deploy applications

## set cf api point
```
cf api --skip-ssl-validation https://api.bosh-lite.com
```

## cf login
```
cf login # credentials: admin/admin
```

## Create and target org
```
cf create -org lightbend
cf target -o lightbend
```

## Create space
```
cf create-space development
```

## Target org and space
```
cf target -o "lightbend" -s "development"
```

## deploy non clustered akka app -- Optional
```
cd akka-sample-cluster
sbt assembly
cf push sample-akka-non-cluster -p target/scala-2.11/akka-sample-cluster-assembly-0.1-SNAPSHOT.jar -b https://github.com/cloudfoundry/java-buildpack.git
```

# Network plugin installation

## Install Go
```
brew install go
```

## Set env var for go build and go to network plugin folder -- CHANGE $workspace to where it really is
```
export GOPATH=$workspace/netman-release
cd netman-release/src/cli-plugin
```

## Build and install plugin
```
go build -o /tmp/network-policy-plugin
chmod +x /tmp/network-policy-plugin
cf install-plugin -f /tmp/network-policy-plugin
```

## From bosh-lite folder
```
vagrant ssh -c 'sudo modprobe br_netfilter'
```

## From workspace folder
```
curl -L -o bosh-lite-stemcell-latest.tgz https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent
bosh upload stemcell bosh-lite-stemcell-latest.tgz


git clone --recursive https://github.com/cloudfoundry/diego-release
git clone --recursive https://github.com/cloudfoundry/cf-release
git clone --recursive https://github.com/cloudfoundry-incubator/netman-release
git clone --recursive https://github.com/cloudfoundry/garden-runc-release

cd garden-runc-release
git checkout develop
git submodule update --init --recursive

cd diego-release
git checkout develop
git submodule update --init --recursive

bosh target vagrant && bosh create release && bosh upload release

vagrant up
```

## set alias to `lite` so scripts will work
```
bosh target 192.168.50.4 lite
```

## set go variables
```
export GOROOT=/usr/local/opt/go/libexec
```

##  Deploy
```
cd netman-release
./scripts/deploy-to-bosh-lite
```

## Notes:
### If you get 'the master was dirty', check releases folders: diego, cf, netman and do
```
git submodule update --init --recursive
```

### If you get duplicate releases then you you need to remove `dev_releases` folder in `netman-release` and call 
```
bosh delete release netman 0.2.0+dev.1 # it maybe 1 or 2 or whatever at the end
```

#### Optional: CF sample app
```
cd netman-release/src/example-apps/cats-and-dogs/frontend
cf push frontend
cd netman-release/src/example-apps/cats-and-dogs/backend
cf push backend
cf set-env backend CATS_PORTS "5678,9876"
cf restage backend
cf access-allow frontend backend --port 9876 --protocol tcp
``` 

# Service Discovery needed for proper seed functioning 

## For how to install amalgam8 on CF follow the instructions for amalgam8 [here](https://github.com/cloudfoundry-incubator/netman-release/tree/develop/src/example-apps/tick)

# Deploy Akka Cluster

## Deploy backend as service with no routes and health checks, allow backend talk one to each other
```
cd akka-sample-cluster
sbt backend:assembly
cf push --no-route --health-check-type sample-akka-cluster-backend -p target/scala-2.11/akka-sample-backend.jar -b https://github.com/cloudfoundry/java-buildpack.git
cf access-allow sample-akka-cluster-backend sample-akka-cluster-backend --port 2551 --protocol tcp
```

### Note: `cf scale sample-akka-cluster-backend -i 2` will scale the backend to 2 instances but you need to wait with this command till first seed node registers with amalgam8, otherwise you might get split clusters.

## Deploy frontend
```
sbt frontend:assembly
cf push sample-akka-cluster-frontend -p target/scala-2.11/akka-sample-frontend.jar -b https://github.com/cloudfoundry/java-buildpack.git
cf access-allow sample-akka-cluster-frontend sample-akka-cluster-backend --port 2551 --protocol tcp
```

## Don't forget to `vagrant suspend` before you're leaving otherwise you'll have to recreate VMs