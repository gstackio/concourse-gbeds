Concourse GBE Downstream
========================

This repository deploys a Concourse cluster behind a Træfik reverse-proxy.

It is a very first example of a _GBE downstream_ repository, which means that
it follows a _GBE upstream_ repository. In this case, the upstream is the main
[Easy Foundry repository][easyfoundry_repo].

[easyfoundry_repo]: https://github.com/gstackio/gstack-bosh-environment



Topology & Architecture
-----------------------

The Concourse Cluster from Easy Foundry has these charateristics:

- CredHub for storing pipeline secrets

- UAA for CredHub (not for autenticating users, contributions are welcome)

- Separate instance groups for CredHub and UAA (allows independent scaling of
  those components)

- Security: encryption all over the place
    - Encryption of data a the database level
    - HTTPS between all components
    - No HTTP anywhere

- Discovery (and failover) of components through BOSH DNS aliases (the current
  best practice, and most portable choice)

- Sanity tests as post-deploy hook (this fixes a missing piece, because no
  smoke tests are shipped by the Concourse BOSH Release, unfortunately)


Beyond these base characteristics, we have added here the following additions.

- Scale horizontally for limited downtime during upgrades.

    - Scale ATC (the web UI), CredHub, UAA and Concourse Worker to 2 instances,
      for limited downtime when upgrading the Concourse cluster.

    - Keep Postgres to 1 instance (because the standard and basic
      [Postgres BOSH Release][postgres_release] we use doesn't implement
      leader-follower replication, nor failover, which BTW are not trivial
      matters).

- Add the ATCs behind a [Træfik][traefik_release] reverse-proxy.

    - With [circuit breaking][traefik_circuit_breaking]: whenever those 2 ATCs
      produce more than 50% network errors, then a “Service Unavailable” error
      message is displayed by Træfik.

    - With [Dynamic Round Robin load-balancing][traefik_drr_lb] between ATC
      nodes, in case only one of them performs bad compared to the other.

    - With [health checks][traefik_health_check], so that when an ATC node is
      shut down by BOSH during an upgrade, then no traffik is served to it
      anymore.

[postgres_release]: https://github.com/cloudfoundry/postgres-release
[traefik_release]: https://github.com/gstackio/traefik-boshrelease
[traefik_circuit_breaking]: https://docs.traefik.io/basics/#circuit-breakers
[traefik_drr_lb]: https://docs.traefik.io/basics/#load-balancing
[traefik_health_check]: https://docs.traefik.io/basics/#health-check



Getting started
---------------

### Create the BOSH environment

We first Git-clone GBE next to this repo, and we name it `bosh-environment`.

```bash
git clone https://github.com/gstackio/gstack-bosh-environment.git bosh-environment
git clone https://github.com/gstackio/concourse-gbeds.git
cd concourse-gbeds/
echo "--- {}" > concourse-bosh-env/conf/secrets.yml
chmod 600 concourse-bosh-env/conf/secrets.yml
```

Then we go check the [GBE pre-requisites][gbe_bare_metal_doc]. Here the
provided `concourse-{bosh,garden}-env` GBE environments are modeled after the
[hybrid topology][hybrid_topology_doc] to create the base BOSH environment.

We [configure our BOSH environment][configure_gbe_env_doc] as detailed in the
GBE documentation. When ready, we create our environment.

```bash
export GBE_ENVIRONMENT=concourse-bosh-env
source <(./bin/gbe env)  # add 'gbe' to the $PATH
GBE_ENVIRONMENT=concourse-garden-env gbe up \
    && GBE_ENVIRONMENT=concourse-bosh-env gbe up
source <(./bin/gbe env)  # reload the updated environment variables
```

We could either create your BOSH environment with [BUCC][bucc_repo] anyway,
this would make no difference as long as the BUCC is properly targeted with
`source <(./bin/bucc env)` for the following instructions below.


### Deploy the GBE subsystems

We update the Cloud and Runtime configs of our environment.

```bash
gbe update cloud-config runtime-config
```

We configure the `external_host` property in
`deployments/concourse-standalone/conf/spec.yml` with a fully-qualified DNS
name that points to the `external_ip` we have set in the
`concourse-{bosh,garden}-env/conf/spec.yml` files. And yes, we need to set a
DNS `A` record for this in our DNS zone.

Also in `deployments/traefik-concourse/conf/spec.yml`, we can configure these
properties: `web_backend_hostname`, `acme_certs_email` and `traefik_domain`.

We are ready to converge the two GBE subsystems for Træfik and Concourse.

```bash
gbe converge -y traefik-concourse concourse-standalone
```


### Play with our production-class Concourse CI

Assuming we have set `external_ip: ci.example.com` above and the DNS has
converged, then we can open `https://ci.example.com` and use our Concourse.
The first HTTPS request will provision a new TLS certificate. By default we
use the staging Let's Encrypt API, so the certificate is reported untrusted by
web browsers (red lock).

Later, when our setup is correct, we can opt for `acme_staging: false` in
Træfik subsystem's `spec.yml` config. After this, the first request we make to
`https://ci.example.com` will generate a green-lock HTTPS certificate with the
Let's Encrypt production endpoint. We need to be careful though, because this
production endpoint is subject to very strict API limitations.

[gbe_bare_metal_doc]: https://github.com/gstackio/gstack-bosh-environment/blob/master/docs/getting-started/bare-metal.md
[hybrid_topology_doc]: https://github.com/gstackio/gstack-bosh-environment#topologies
[configure_gbe_env_doc]: https://github.com/gstackio/gstack-bosh-environment/blob/master/docs/getting-started/bare-metal.md#configure-the-bosh-environment
[bucc_repo]: https://github.com/starkandwayne/bucc



Contributing
------------

Please feel free to submit issues and pull requests.



Author and License
------------------

Copyright © 2018, Benjamin Gandon, Gstack

Like BOSH and GBE, this example GBE downstream project is released under the
terms of the [Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0).

<!--
# Local Variables:
# indent-tabs-mode: nil
# End:
-->
