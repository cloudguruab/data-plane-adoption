# Heat adoption

Adopting Heat means that an existing `OpenStackControlPlane` CR, where Heat
is supposed to be disabled, should be patched to start the service with the
configuration parameters provided by the source environment.

After the adoption process has been completed, a user can expect that they
will then have CR's for `Heat`, `HeatAPI`, `HeatEngine` and `HeatCFNAPI`.
Additionally, a user should have endpoints created within Keystone to facilitate
the above mentioned servies.

This guide also assumes that:

1. A `TripleO` environment (the source Cloud) is running on one side;
2. A OpenShift environment is running on the other side.

## Prerequisites

* Previous Adoption steps completed. Notably, MariaDB and Keystone
  should be already adopted.
* In addition, if your existing Heat stacks contain resources from other services
  such as Neutron, Nova, Swift, etc. Those services should be adopted first before
  trying to adopt Heat.

## Procedure - Heat adoption

As already done for [Keystone](https://github.com/openstack-k8s-operators/data-plane-adoption/blob/main/keystone_adoption.md), the Heat Adoption follows a similar pattern.

Patch the `osp-secret` to update the `HeatAuthEncryptionKey` and `HeatPassword`. This needs
to match what you have configured in the existing TripleO Heat configuration.


You can retrieve and verify the existing `auth_encryption_key` and `service` passwords via:
```bash
[stack@rhosp17 ~]$ grep -E 'HeatPassword|HeatAuth' ~/overcloud-deploy/overcloud/overcloud-passwords.yaml
  HeatAuthEncryptionKey: Q60Hj8PqbrDNu2dDCbyIQE2dibpQUPg2
  HeatPassword: dU2N0Vr2bdelYH7eQonAwPfI3
```

And verifying on one of the Controllers that this is indeed the value in use:
```bash
[stack@rhosp17 ~]$ ansible -i overcloud-deploy/overcloud/config-download/overcloud/tripleo-ansible-inventory.yaml overcloud-controller-0 -m shell -a "grep auth_encryption_key /var/lib/config-data/puppet-generated/heat/etc/heat/heat.conf | grep -Ev '^#|^$'" -b
overcloud-controller-0 | CHANGED | rc=0 >>
auth_encryption_key=Q60Hj8PqbrDNu2dDCbyIQE2dibpQUPg2
```

This password needs to be base64 encoded and added to the `osp-secret`
```bash
❯ echo Q60Hj8PqbrDNu2dDCbyIQE2dibpQUPg2 | base64
UTYwSGo4UHFickROdTJkRENieUlRRTJkaWJwUVVQZzIK

❯ oc patch secret osp-secret --type='json' -p='[{"op" : "replace" ,"path" : "/data/HeatAuthEncryptionKey" ,"value" : "UTYwSGo4UHFickROdTJkRENieUlRRTJkaWJwUVVQZzIK"}]'
secret/osp-secret patched
```

Patch OpenStackControlPlane to deploy Heat:

```bash
oc patch openstackcontrolplane openstack --type=merge --patch '
spec:
  heat:
    enabled: true
    apiOverride:
      route: {}
    template:
      databaseInstance: openstack
      secret: osp-secret
      memcachedInstance: memcached
      passwordSelectors:
        authEncryptionKey: HeatAuthEncryptionKey
        database: HeatDatabasePassword
        service: HeatPassword
'
```

## Post-checks

Ensure all of the CR's reach the "Setup Complete" state:
```bash
❯ oc get Heat,HeatAPI,HeatEngine,HeatCFNAPI
NAME                           STATUS   MESSAGE
heat.heat.openstack.org/heat   True     Setup complete

NAME                                  STATUS   MESSAGE
heatapi.heat.openstack.org/heat-api   True     Setup complete

NAME                                        STATUS   MESSAGE
heatengine.heat.openstack.org/heat-engine   True     Setup complete

NAME                                        STATUS   MESSAGE
heatcfnapi.heat.openstack.org/heat-cfnapi   True     Setup complete
```

### Check that Heat service is registered in Keystone

```bash
 oc exec -it openstackclient -- openstack service list -c Name -c Type
+------------+----------------+
| Name       | Type           |
+------------+----------------+
| heat       | orchestration  |
| glance     | image          |
| heat-cfn   | cloudformation |
| ceilometer | Ceilometer     |
| keystone   | identity       |
| placement  | placement      |
| cinderv3   | volumev3       |
| nova       | compute        |
| neutron    | network        |
+------------+----------------+
```

```bash
❯ oc exec -it openstackclient -- openstack endpoint list --service=heat -f yaml
- Enabled: true
  ID: 1da7df5b25b94d1cae85e3ad736b25a5
  Interface: public
  Region: regionOne
  Service Name: heat
  Service Type: orchestration
  URL: http://heat-api-public-openstack-operators.apps.okd.bne-shift.net/v1/%(tenant_id)s
- Enabled: true
  ID: 414dd03d8e9d462988113ea0e3a330b0
  Interface: internal
  Region: regionOne
  Service Name: heat
  Service Type: orchestration
  URL: http://heat-api-internal.openstack-operators.svc:8004/v1/%(tenant_id)s
```

### Check Heat engine services are up
```bash
 oc exec -it openstackclient -- openstack orchestration service list -f yaml
- Binary: heat-engine
  Engine ID: b16ad899-815a-4b0c-9f2e-e6d9c74aa200
  Host: heat-engine-6d47856868-p7pzz
  Hostname: heat-engine-6d47856868-p7pzz
  Status: up
  Topic: engine
  Updated At: '2023-10-11T21:48:01.000000'
- Binary: heat-engine
  Engine ID: 887ed392-0799-4310-b95c-ac2d3e6f965f
  Host: heat-engine-6d47856868-p7pzz
  Hostname: heat-engine-6d47856868-p7pzz
  Status: up
  Topic: engine
  Updated At: '2023-10-11T21:48:00.000000'
- Binary: heat-engine
  Engine ID: 26ed9668-b3f2-48aa-92e8-2862252485ea
  Host: heat-engine-6d47856868-p7pzz
  Hostname: heat-engine-6d47856868-p7pzz
  Status: up
  Topic: engine
  Updated At: '2023-10-11T21:48:00.000000'
- Binary: heat-engine
  Engine ID: 1011943b-9fea-4f53-b543-d841297245fd
  Host: heat-engine-6d47856868-p7pzz
  Hostname: heat-engine-6d47856868-p7pzz
  Status: up
  Topic: engine
  Updated At: '2023-10-11T21:48:01.000000'

```

### Verify you can now see your Heat stacks again

We can now test that user can create networks, subnets, ports, routers etc.

```bash
❯ openstack stack list -f yaml
- Creation Time: '2023-10-11T22:03:20Z'
  ID: 20f95925-7443-49cb-9561-a1ab736749ba
  Project: 4eacd0d1cab04427bc315805c28e66c9
  Stack Name: test-networks
  Stack Status: CREATE_COMPLETE
  Updated Time: null
```

