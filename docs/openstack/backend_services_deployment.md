# Backend services deployment

The following instructions create OpenStackControlPlane CR with basic
backend services deployed, and all the OpenStack services disabled.
This will be the foundation of the podified control plane.

In subsequent steps, we'll import the original databases and then add
podified OpenStack control plane services.

## Prerequisites

* The cloud which we want to adopt is up and running. It's on
  OpenStack Wallaby release.

* The `openstack-operator` is deployed, but `OpenStackControlPlane` is
  **not** deployed.

  For developer/CI environments, the openstack operator can be deployed
  by running `make openstack` inside
  [install_yamls](https://github.com/openstack-k8s-operators/install_yamls)
  repo.

  For production environments, the deployment method will likely be
  different.

* There are free PVs available to be claimed (for MariaDB and RabbitMQ).

  For developer/CI environments driven by install_yamls, make sure
  you've run `make crc_storage`.


## Variables

* Set the desired admin password for the podified deployment. This can
  be the original deployment's admin password or something else.

  ```
  ADMIN_PASSWORD=SomePassword
  ```

  To use the existing OpenStack deployment password:
  ```
  ADMIN_PASSWORD=$(cat ~/tripleo-standalone-passwords.yaml | grep ' AdminPassword:' | awk -F ': ' '{ print $2; }')
  ```

* Set service password variables to match the original deployment.
  Database passwords can differ in podified environment, but
  synchronizing the service account passwords is a required step.

  E.g. in developer environments with TripleO Standalone, the
  passwords can be extracted like this:

  ```
  CINDER_PASSWORD=$(cat ~/tripleo-standalone-passwords.yaml | grep ' CinderPassword:' | awk -F ': ' '{ print $2; }')
  GLANCE_PASSWORD=$(cat ~/tripleo-standalone-passwords.yaml | grep ' GlancePassword:' | awk -F ': ' '{ print $2; }')
  HEAT_AUTH_ENCRYPTION_KEY=$(cat ~/tripleo-standalone-passwords.yaml | grep ' HeatAuthEncryptionKey:' | awk -F ': ' '{ print $2; }')
  HEAT_PASSWORD=$(cat ~/tripleo-standalone-passwords.yaml | grep ' HeatPassword:' | awk -F ': ' '{ print $2; }')
  IRONIC_PASSWORD=$(cat ~/tripleo-standalone-passwords.yaml | grep ' IronicPassword:' | awk -F ': ' '{ print $2; }')
  MANILA_PASSWORD=$(cat ~/tripleo-standalone-passwords.yaml | grep ' ManilaPassword:' | awk -F ': ' '{ print $2; }')
  NEUTRON_PASSWORD=$(cat ~/tripleo-standalone-passwords.yaml | grep ' NeutronPassword:' | awk -F ': ' '{ print $2; }')
  NOVA_PASSWORD=$(cat ~/tripleo-standalone-passwords.yaml | grep ' NovaPassword:' | awk -F ': ' '{ print $2; }')
  OCTAVIA_PASSWORD=$(cat ~/tripleo-standalone-passwords.yaml | grep ' OctaviaPassword:' | awk -F ': ' '{ print $2; }')
  PLACEMENT_PASSWORD=$(cat ~/tripleo-standalone-passwords.yaml | grep ' PlacementPassword:' | awk -F ': ' '{ print $2; }')
  ```

## Pre-checks

## Procedure - backend services deployment

* Make sure you are using the OpenShift namespace where you want the
  podified control plane deployed:

  ```
  oc project openstack
  ```

* Create OSP secret.

  The procedure for this will vary, but in developer/CI environments
  we use install_yamls:

  ```
  # in install_yamls
  make input
  ```

* If the `$ADMIN_PASSWORD` is different than the already set password
  in `osp-secret`, amend the `AdminPassword` key in the `osp-secret`
  correspondingly:

  ```
  oc set data secret/osp-secret "AdminPassword=$ADMIN_PASSWORD"
  ```

* Set service account passwords in `osp-secret` to match the service
  account passwords from the original deployment:

  ```
  oc set data secret/osp-secret "CinderPassword=$CINDER_PASSWORD"
  oc set data secret/osp-secret "GlancePassword=$GLANCE_PASSWORD"
  oc set data secret/osp-secret "HeatAuthEncryptionKey=$HEAT_AUTH_ENCRYPTION_KEY"
  oc set data secret/osp-secret "HeatPassword=$HEAT_PASSWORD"
  oc set data secret/osp-secret "IronicPassword=$IRONIC_PASSWORD"
  oc set data secret/osp-secret "ManilaPassword=$MANILA_PASSWORD"
  oc set data secret/osp-secret "NeutronPassword=$NEUTRON_PASSWORD"
  oc set data secret/osp-secret "NovaPassword=$NOVA_PASSWORD"
  oc set data secret/osp-secret "OctaviaPassword=$OCTAVIA_PASSWORD"
  oc set data secret/osp-secret "PlacementPassword=$PLACEMENT_PASSWORD"
  ```

* Deploy OpenStackControlPlane. **Make sure to only enable DNS,
  MariaDB, Memcached, and RabbitMQ services. All other services must
  be disabled.**

  ```yaml
  oc apply -f - <<EOF
  apiVersion: core.openstack.org/v1beta1
  kind: OpenStackControlPlane
  metadata:
    name: openstack
  spec:
    secret: osp-secret
    storageClass: local-storage

    cinder:
      enabled: false
      template:
        cinderAPI: {}
        cinderScheduler: {}
        cinderBackup: {}
        cinderVolumes: {}

    dns:
      template:
        override:
          service:
            metadata:
              annotations:
                metallb.universe.tf/address-pool: ctlplane
                metallb.universe.tf/allow-shared-ip: ctlplane
                metallb.universe.tf/loadBalancerIPs: 192.168.122.80
            spec:
              type: LoadBalancer
        options:
        - key: server
          values:
          - 192.168.122.1
        replicas: 1

    glance:
      enabled: false
      template:
        glanceAPIs: {}

    horizon:
      enabled: false
      template: {}

    ironic:
      enabled: false
      template:
        ironicConductors: []

    keystone:
      enabled: false
      template: {}

    manila:
      enabled: false
      template:
        manilaAPI: {}
        manilaScheduler: {}
        manilaShares: {}

    mariadb:
      enabled: false
      templates: {}

    galera:
      enabled: true
      templates:
        openstack:
          secret: osp-secret
          replicas: 1
          storageRequest: 500M
        openstack-cell1:
          secret: osp-secret
          replicas: 1
          storageRequest: 500M

    memcached:
      enabled: true
      templates:
        memcached:
          replicas: 1

    neutron:
      enabled: false
      template: {}

    nova:
      enabled: false
      template: {}

    ovn:
      enabled: false
      template:
        ovnDBCluster:
          ovndbcluster-nb:
            dbType: NB
            storageRequest: 10G
            networkAttachment: internalapi
          ovndbcluster-sb:
            dbType: SB
            storageRequest: 10G
            networkAttachment: internalapi
        ovnNorthd:
          networkAttachment: internalapi
          replicas: 1
        ovnController:
          networkAttachment: tenant

    placement:
      enabled: false
      template: {}

    rabbitmq:
      templates:
        rabbitmq:
          override:
            service:
              metadata:
                annotations:
                  metallb.universe.tf/address-pool: internalapi
                  metallb.universe.tf/loadBalancerIPs: 172.17.0.85
              spec:
                type: LoadBalancer
        rabbitmq-cell1:
          override:
            service:
              metadata:
                annotations:
                  metallb.universe.tf/address-pool: internalapi
                  metallb.universe.tf/loadBalancerIPs: 172.17.0.86
              spec:
                type: LoadBalancer

    telemetry:
      enabled: false
      template: {}
  EOF
  ```

## Post-checks

* Check that MariaDB is running.

  ```
  oc get pod openstack-galera-0 -o jsonpath='{.status.phase}{"\n"}'
  oc get pod openstack-cell1-galera-0 -o jsonpath='{.status.phase}{"\n"}'
  ```
