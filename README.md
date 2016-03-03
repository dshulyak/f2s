#How to install on fuel master?

If solar package is not available yet install solar from pip:

```
yum install gcc-c++
pip install virtualenv
virtualenv venv
source venv/bin/activate

pip install solar
```

# configure solar:
All of this things will be automated by solar eventually

```
mkdir /etc/solar
echo solar_db: sqlite:////tmp/solar.db > /etc/solar/solar.conf

# generate solar resource from library tasks
f2s t2r --dir /tmp/f2s

# create solar Resource defintions repository
mkdir -p /var/lib/solar/repositories
solar repo import /tmp/f2s -n f2s
solar repo update f2s f2s/created

# clone solar-resources and create repositories
git clone https://github.com/openstack/solar-resources.git
solar repo import solar-resources/resources
```

workflow
```
# fetch nodes from nailgun and create node resources with transports
fsclient nodes 2
# create anchors from fuel graph
fsclient alloc 2 null
# create role data and dependency from role data to pre_deployment_start
fsclient prep 2 2
# generate all fuel tasks
fsclient alloc 2 2

solar ch stage
solar ch process
solar or run-once last
```

# configure fuel node

```
# provision a node by fuel
fuel node --node 1 --provision

# run on each remote
mkdir /var/lib/astute
mkdir /etc/puppet/hieradata
```

#f2s.py

This script converts tasks.yaml + library actions into solar resources,
vrs, and events.

1. Based on tasks.yaml meta.yaml is generated, you can take a look on example
at f2s/resources/netconfig/meta.yaml
2. Based on hiera lookup we generated inputs for each resource, patches can be
found at f2s/patches
3. VRs (f2s/vrs) generated based on dependencies between tasks and roles

#fsclient.py

This script helps to create solar resource with some of nailgun data.
Note, you should run it inside of the solar container.

`./f2s/fsclient.py master 1`
Accepts cluster id, prepares transports for master + generate keys task
for current cluster.

`./f2s/fsclient.py nodes 1`
Prepares transports for provided nodes, ip and cluster id fetchd from nailgun.

`./f2s/fsclient.py prep 1`
Creates tasks for syncing keys + fuel-library modules.

`./f2s/fsclient.py roles 1`
Based on roles stored in nailgun we will assign vrs/<role>.yaml to a given
node. Right now it takes while, so be patient.

#fetching data from nailgun

Special entity added which allows to fetch data from any source
*before* any actual deployment.
This entity provides mechanism to specify *manager* for resource (or list of them).
Manager accepts inputs as json in stdin, and outputs result in stdout,
with result of manager execution we will update solar storage.

Examples can be found at f2s/resources/role_data/managers.

Data will be fetched on solar command

`solar res prefetch -n <resource name>`

# TODO

Configure hiera, without it changes in resource will not be seen. It's ok for now.
```
 :backends:
  - yaml
  #- json
:yaml:
  :datadir: /etc/puppet/hieradata
:json:
  :datadir: /etc/puppet/hieradata
:hierarchy:
  - "%{resource_name}"
  - resource
```


#basic troubleshooting

If there are any Fuel plugin installed, you should manually
create a stanza for it in the `./f2s/resources/role_data/meta.yaml`,
like:
```
input:
  foo_plugin_name:
    value: null
```

And regenerate the data from nailgun,

To regenerate the deployment data to Solar resources make
```
solar res clear_all
```

and repeat all of the fsclient.py and fetching nailgun data steps
