# cvop2cvaas

Migrate all devices, containers and configlets from CV on-prem to CVaaS. 
The `cvaas_migration.yaml` playbook does the following:
- collects cvp facts (devices, containers and configlets) from the on-prem CVP system and stores it in memory
- fetches the devices in the inventory and builds a new inventory file to be used later
- generates TerminAttr onboarding token from CVaaS
- uploads the generated token to all devices using scp
- configures the devices with the TerminAttr config pointing to CVaaS
- pushes all the configlets from on-prem to CVaaS
- creates the container hierarchy (learned from the on-prem facts)
- moves the devices to their target container and applies the configlets
- creates a new TerminAttr configlet pointing to CVaaS and appends it to the list of configlets for each device

## Prerequisites

- python3
- [ansible-cvp](https://cvp.avd.sh)
  - `$ ansible-galaxy collection install arista.cvp`
- [cvprac 1.0.7+](https://github.com/aristanetworks/cvprac/): `pip install cvprac>=1.0.7`
  - Recommended cvprac 1.0.8+ (speed optimizations!)
- scp (`pip install scp` or `pip3 install scp`) <-- Check with `pip --version` if it points to py2 or py3
- devices should run TerminAttr 1.15.3 (CVaaS requirement)
- devices should run EOS 4.23+ for non-prod clusters and 4.22+ for prod clusters (CVaaS requirement)
- all devices must have internet connectivity and be able to reach CVaaS, a quick ping test should be enough like below:

```shell
ping vrf MGMT apiserver.cv-staging.corp.arista.io
ping vrf MGMT www.cv-staging.corp.arista.io
```

## Steps

### Option 1 - Generate TerminAttr Config on the fly

This example is more of a faster approach for lab environments

1. Generate service account token on CVaaS ([steps](#how-to-generate-service-accounts))

The token should be copied and saved to a file that can later be referred to, in this example it's in `/tokens/cvaas.tok`.

2. Generate service account token on CV on-prem ([steps](#how-to-generate-service-accounts)), save it to a file (e.g.: `/tokens/go178.tok`)

3. Export the tokens as env vars, e.g.:

```
export CVAAS_TOKEN=`cat /tokens/cvaas.tok`
export ON_PREM_TOKEN=`cat /tokens/go178.tok`
```

> Tip: Add them to your `.bashrc` or `.zshrc` to make them persistent and source them on the current terminal (`source ~/.zshrc`) or start a new session.

4. Go to CVaaS UI and generate the TerminAttr config and update the playbook under the `"Configuring TerminAttr on {{ inventory_hostname }}"` task
and inside the [terminattr.cfg](./terminattr.cfg) file

![terminattr_config_cvaas](./media/cvaas_ta_onboarding_config.png)

5. Update the `./inventory/inventory.yaml` file with the right credentials and IPs/FQDNs

6. Update `ansible.cfg` to point to the right folders for your `collections_paths` or just install ansible-cvp using ansible-galaxy and use that instead.

7. Generate the device inventory with `ansible-playbook cvaas_migrationV2.yaml -i inventory --tags devinv`
8. Finally run it as `ansible-playbook cvaas_migrationV2.yaml -i inventory --skip-tags=debug,devinv`


### Option 2 - Stream to both on-prem and CloudVision-As-a-Service

This example is recommended for production.

1. Generate service account token on CVaaS ([steps](#how-to-generate-service-accounts))

The token should be copied and saved to a file that can later be referred to, in this example it's in `/tokens/cvaas.tok`.

2. Generate service account token on CV on-prem ([steps](#how-to-generate-service-accounts)), save it to a file (e.g.: `/tokens/go178.tok`)

3. Export the tokens as env vars, e.g.:

```
export CVAAS_TOKEN=`cat /tokens/cvaas.tok`
export ON_PREM_TOKEN=`cat /tokens/go178.tok`
```

> Tip: Add them to your `.bashrc` or `.zshrc` to make them persistent and source them on the current terminal (`source ~/.zshrc`) or start a new session.

4. Go to CVaaS UI and generate the TerminAttr config and build the new TerminAttr configuration to stream to both the existing on-prem cluster and CVaaS. An example can be found in the [TerminAttr most commonly used flags documentation](https://aristanetworks.force.com/AristaCommunity/s/article/terminattr-most-commonly-used-flags-and-sample-configurations) or in the [terminattr_multi_cluster.cfg](./terminattr_multi_cluster.cf) file

![terminattr_config_cvaas](./media/cvaas_ta_onboarding_config.png)

5. Run `ansible-playbook option2_terminattr_multi_cluster.yaml --tags build` to generate the TerminAttr onboarding tokens for both on-prem and CVaaS and to generate the `onprem_devices.yaml` inventory file

6. Run `ansible-playbook option2_terminattr_multi_cluster.yaml --tags deploy` to upload the tokens and reconfigure TerminAttr to stream to both clusters and wait for the devices to show up in the Undefined container on CVaaS UI.

7. Update the existing TerminAttr configlet with the new TerminAttr configuration on the on-prem CVP cluster, and assign it to all devices to match the running-configuration and 
make all devies compliant.

8. Update the `./inventory/inventory.yaml` file with the right credentials and IPs/FQDNs

9.  Update `ansible.cfg` to point to the right folders for your `collections_paths` or just install ansible-cvp using ansible-galaxy and use that instead.

10. Finally, run the playbook to migrate the containers, configlets, move the devices to their intended containers and assign the configlets to the containers and devices:
 `ansible-playbook option2_cvaas_migration.yaml -i inventory`

11. Later when the on-prem servers will be decommissioned the TerminAttr configuration can be changed to stream only to CVaaS.

## Disclaimer

This is a proof-of-concept demo, highly recommended to take a backup before running the playbook.


## TO-DO

- export Studio templates
- export Dashboards
- export tags
- figure out why `search_key: serialNumber` doesn't work

End to end example can be watched at [youtube](https://www.youtube.com/watch?v=rN6meAtXqss)

## Appendix

### How to generate service accounts

![serviceaccount1](./media/serviceaccount1.png)
![serviceaccount2](./media/serviceaccount2.png)
![serviceaccount3](./media/serviceaccount3.png)