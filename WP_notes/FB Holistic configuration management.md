# FB Holistic configuration management 

Article [link](http://sigops.org/s/conferences/sosp/2015/current/2015-Monterey/printable/008-tang.pdf)

- Configuration as a code - much simpler to manage. After all configuration provides control points for application to make decisions or create assumptions. This can be a code once gets complicated enough.

- Configuration depedency tree has to be built to know about other affected subsystems

- Tenants of configuration management 

  - Configuration as a code - [Thrift](https://thrift-tutorial.readthedocs.io/en/latest/intro.html
    )
  - Gating new product features rollout
  - Conducting experiemnts through AB testing
  - Traffic control - at the edge duplication, load balacing and various other testing.
  - Topology setup and load balancing - FB social graph DB TAO and its bucketing stategy to channel user traffic.
  - Monitoring, alerts and remediation 
  - Machine learned Data models update which are larger in size.
  - Controlling applications internal behaviour like use of cache, memory foot print, may be cgroup configuration.

- Architectural components 

  - Configurator 

    - allows config as a code, compilation and serialization of configs for local and canary testing.
    - tests through various spec files (ruby terminology) for canary environment.
    - Config distribution - push is preffered over pull
    - Pull model is simple to implement but we need to know what to pull?  When to pull
    - Push model gives command and control where it will decide on when change is support to happen in what?
    - Git throuput limitation over serialization

  - gatekeeper

    - application rollout restraints 
    - also written in the programming laguage 
    - [DNF](https://en.wikipedia.org/wiki/Disjunctive_normal_form 
      ) type decesion making
    - deploy often reduces the use of branching which is easier to manage.

  - PackageVessel

    - P2P transfer for larger datasets

  - Sitevars

    - key value pair data (something like range)

  - Mobile config

    - optimized way to distribute configs on the mobile devices

    - Limits

      - network b/w is limited
      - fragmentation of mobile OSes
      - Legacy OSes

    - Just Push is not efficient due to bandwidth limitation instead compare config based of hash and then send a diff to device.

      