==============================================================================
CPU Performance Service Levels using CPU Shares and Placement Resource Classes
==============================================================================

This specification proposes a mechanism within Nova to allow users to
request and receive predictable CPU performance service levels for their
instances. By leveraging CPU shares managed via control groups (cgroups)
and integrating with OpenStack Placement using standardized resource
classes, we aim to provide a standardized and quantifiable way for users
to express desired performance tiers and for cloud administrators to
enforce performance guarantees, particularly in oversubscribed
environments. This addresses the current lack of a reliable method
for users to ensure consistent CPU performance for their workloads.

Problem description
===================

Users currently lack a standardized and reliable method within Nova to
request and guarantee specific levels of CPU performance for their
instances. While existing mechanisms like CPU pinning or NUMA topology
awareness provide some control, they are often complex to manage and do not
offer a simple,
quantifiable way to express desired performance tiers across a shared
infrastructure, leading to unpredictable performance, especially in
oversubscribed environments, making it difficult for users to run
performance-sensitive workloads with confidence.

The goal is to introduce a mechanism that allows users to request CPU
performance service levels based on CPU shares, providing a more
predictable and manageable approach to CPU resource allocation and
performance guarantees.

Definitions
-----------

CPU shares
    A mechanism within the Linux kernel's control groups (cgroups) to provide
    a proportional share of CPU time to processes or groups of processes
    when the CPU is contended. Higher share values result in a greater
    proportion of CPU time.

cgroups (Control Groups)
    A Linux kernel feature that limits, accounts for, and isolates the
    resource usage (CPU, memory, disk I/O, network, etc.) of a collection
    of processes.

OpenStack Placement
    An OpenStack service that tracks resource provider inventories and
    allocations, enabling the scheduler to make informed decisions about
    where instances can be hosted.

Resource Class
    A named category of resources managed by the Placement service (e.g.,
    VCPU, MEMORY_MB, DISK_GB). Custom resource classes can be defined for
    specific needs.

Flavor Extra Spec
    Key-value pairs associated with a Nova flavor that provide additional
    configuration options or constraints for instances created from that
    flavor.

VCPU_SHARES
    A proposed standard resource class in OpenStack Placement representing
    a unit of CPU performance based on CPU shares.

ResourceProviderWeigher
    A Nova scheduler component that calculates a weight for each potential
    host based on its resource availability and other factors, influencing
    host selection during scheduling.

Over subscription
    The practice of allocating more virtual resources (like VCPUs) than the
    physical resources available on a host, relying on the assumption that
    not all allocated resources will be used simultaneously.

CFS Scheduler
    The Linux Completely Fair Scheduler (CFS)
    In virtualized environments using KVM, each virtual CPU (vCPU) operates
    as a host thread scheduled by CFS across physical cores.
    The kernel scheduler distributes these vCPU threads dynamically,
    allocating time slices based on their assigned shares. This ensures:
    * Proportional CPU access during contention periods
    * Fair distribution across all runnable threads
    * Predictable performance through share-based prioritization
      under contention

Use Cases
---------

* A user wants to launch a database server instance and ensure it receives
  a guaranteed minimum level of CPU performance, even under high load
  from other instances on the same host.
* A user is running a batch processing job that requires a consistent amount
  of CPU time to complete within a specific timeframe. They need to request
  a performance tier that guarantees this.
* A cloud administrator wants to offer different tiers of compute service
  (e.g., "Bronze", "Silver", "Gold") based on guaranteed CPU performance,
  allowing users to choose the level that best suits their workload and
  budget.
* As a cloud administrator, I want to achieve higher utilization of my
  infrastructure by allowing lower performance tier instances to use more
  resources when there is low contention but reduce them to a lower
  level of performance when higher priority workloads become active.
  For example, running lower priority batch processing jobs when higher
  priority customer workloads are idle.

Proposed change
===============

Introduce a new mechanism in Nova that leverages CPU shares (using control
groups - cgroups) to define and enforce CPU performance service levels.
Administrators will be able to specify a desired CPU share multiplier
value when defining flavors that users can select when launching or
resizing an instance. This translates to a relative proportion of
available CPU resources on the host.

To achieve this, a new ``quota:cpu_shares_multiplier`` flavor extra spec
and ``VCPU_SHARES`` resource class will be introduced. The
``cpu_shares_multiplier`` will be used to calculate the number of
``VCPU_SHARES`` allocations to request based on the number of VCPUs
requested by the flavor.

This approach provides a flexible and granular way to manage CPU
performance, allowing for oversubscription while still providing a
mechanism for performance guarantees based on the allocated shares.

The validation of the new ``quota:cpu_shares_multiplier`` flavor extra spec
will leverage the existing flavor extra spec validation framework
introduced in the `Flavor Extra Spec Validator`_ specification. This
framework allows for the definition and validation of flavor extra specs,
ensuring correct usage and providing better documentation. A validator
for ``quota:cpu_shares_multiplier`` will be added to the
``nova.api.validation.extra_specs.quota`` module, alongside other
quota-related extra spec validators.

.. _Flavor Extra Spec Validator: https://specs.openstack.org/openstack/nova-specs/specs/ussuri/implemented/flavor-extra-spec-validators.html

The primary mechanism for enforcing per-host, per-tier capacity will be through
the use of OpenStack Placement resource classes. This will be combined with
a new ``ResourceProviderWeigher``. Enforcement at the hypervisor level
will be done by the Linux `CFS scheduler`_.

.. _`CFS scheduler`: https://docs.kernel.org/scheduler/sched-design-CFS.html

CPU Share Management
--------------------

* Allocation: CPU shares will be allocated to instances based on a new
  ``quota:cpu_shares_multiplier`` flavor extra spec. The value will represent
  a relative weight, with higher values indicating a greater proportion of
  CPU resources.
* Example:
    Given the allowed range for CPU shares (1-10,000) and assuming a VM
    can use up to 100 vCPUs, the following ``cpu_shares_multiplier``
    values may be appropriate:
    Gold: 100, Silver: 50, Bronze: 25.

.. Note::

  ``quota:cpu_shares_multiplier`` and ``quota:cpu_shares`` are two ways of
  specifying the same constraint. CPU shares as defined via libvirt applies
  to the domain as a whole rather than per vCPU. As a result, a flavor with
  10 VCPUs and ``quota:cpu_shares=100`` would get equal CPU time as a flavor
  with
  1 VCPU and ``quota:cpu_shares=100`` during CPU contention. By expressing
  shares
  as a multiplier (``quota:cpu_shares_multiplier=100``), the intuitive
  expectation that a 10-core VM should receive 10 times the CPU resources
  is maintained under contention. Administrators can choose between using
  ``quota:cpu_shares`` or ``quota:cpu_shares_multiplier`` based on their
  preference. Translation to Placement allocations will be opt-in via
  configuration for upgrade compatibility.

Performance Guarantee Enforcement via Placement
-----------------------------------------------

A standardized ``VCPU_SHARES`` resource class with weighted allocations
will be used to enforce performance tiers through Placement:

* **Automatic Inventory Reporting:** The libvirt driver will automatically:

  - Report ``VCPU_SHARES`` inventory based on host capabilities.
  - Calculate capacity as ``vcpu_share_multiplier * len(cpu_shared_set)``
  - Respect cgroups v2 API limits of [1, 10000] shares.
  - If and only if enabled via a new ``report_vcpu_shares`` config option.

  Configuration options:

  .. code-block:: ini

      [libvirt]
      report_vcpu_shares = true
      # Range: 1-10000
      # min: 1
      # max: 10000
      # default: 100
      vcpu_share_multiplier = 100

  * Default multiplier provides granularity for 1% increments and up to 100
    vCPUs per VM.
  * The multiplier scales the number of VCPUs requested by the flavor into
    VCPU_SHARES units. The total VCPU_SHARES capacity on a host is calculated
    based on the number of physical cores available for sharing
    (``cpu_shared_set``)
    and the ``vcpu_share_multiplier`` (e.g., 64 cores * 100 multiplier = 6,400
    shares capacity).

* Placement Enforcement: The scheduler ensures the sum of allocated shares
  never exceeds the host's advertised capacity. Shares act as consumption
  coefficients rather than absolute limits, preserving the proportional
  guarantee model.
* Overall VCPU Coordination: The standard VCPU inventory and its
  ``cpu_allocation_ratio`` will still be used as the default CPU capacity
  mechanism. The ``cpu_allocation_ratio`` will also be applied to the
  ``VCPU_SHARES`` inventory to ensure oversubscription works as expected.

ResourceProviderWeigher
-----------------------

The ``ResourceProviderWeigher`` calculates host weights by analyzing
provider trees from Placement's allocation candidates data. This weigher
enables capacity-aware scheduling decisions based on both resource
availability and trait compatibility. The weigher behavior can be
summarized as follows: "select the most boring host" where boring is
defined as having the least number of traits or resource classes.

Input Data Structure
^^^^^^^^^^^^^^^^^^^^

The weigher processes ``provider_summaries`` data which represents a
forest of resource provider trees. Each tree is encoded as a map with:

- ``resources``: Map of resource class to (capacity, used) tuples
- ``traits``: Set of trait names
- ``parent_provider_uuid``: UUID of parent provider (null for root)
- ``root_provider_uuid``: UUID of tree root provider

Algorithm Implementation
^^^^^^^^^^^^^^^^^^^^^^^^

1. Tree Flattening
   * Group providers by root UUID
   * Aggregate resource capacities and traits across each tree

2. Resource Calculation:
  * For each available resource class in the provider tree:
  * Sum available capacity across all providers in tree
  * Calculate utilization ratio: ``1 - (requested amount of resource
    class) / (free capacity of resource class)``
  * Calculate the arithmetic mean of these ratios across all available
    resource classes.

 3. Trait Calculation:
    - Collect all traits from tree providers
   * Calculate ratio: ``(count of requested traits present on provider) /
(count of all available traits on provider)``

4. Weight Composition:
   - Combine resource and trait scores: ``(resource_score + trait_score) / 2``
   - Apply multiplier from scheduler configuration

Pseudocode Implementation
^^^^^^^^^^^^^^^^^^^^^^^^^

Here is a pseudocode example illustrating the `ResourceProviderWeigher` logic:

.. code-block:: python

    def _weigh_object(self, host_state, request_spec):
        # host_state contains provider_summaries
        provider_summaries = host_state.provider_summaries
        requested_resources = request_spec.get('resources', {})
        requested_traits = request_spec.get('traits', set())

        resource_ratios = []
        trait_ratios = []

        # 1. Tree Flattening and Aggregation (Simplified)
        # In reality, this involves iterating through the forest and
        # aggregating resources and traits for each root provider tree.
        # For pseudocode, assume we have aggregated_resources and
        # aggregated_traits for the current host's provider tree.
        aggregated_resources = {} # {resource_class: {'capacity': X, 'used': Y}}
        aggregated_traits = set() # {trait_name1, trait_name2, ...}

        # 2. Resource Calculation
        available_resource_classes = aggregated_resources.keys()
        if not available_resource_classes:
            # Handle case with no available resources
            resource_score = 0.0
        else:
            for res_class in available_resource_classes:
                capacity = aggregated_resources[res_class]['capacity']
                used = aggregated_resources[res_class]['used']
                free_capacity = capacity - used

                # Avoid division by zero if free capacity is 0
                if free_capacity > 0:
                    # Use 0 for requested amount if not in the request
                    requested_amount = requested_resources.get(res_class, 0)
                    utilization_ratio = 1.0 - (requested_amount / free_capacity)
                    resource_ratios.append(utilization_ratio)
                else:
                    # If no free capacity, this resource class contributes 0 to the score
                    resource_ratios.append(0.0)

                    # Calculate the arithmetic mean of ratios across all
                    # available resource classes
                    resource_score = sum(resource_ratios) / len(resource_ratios)
                    \
                        if resource_ratios else 0.0

        # 3. Trait Calculation
        if not aggregated_traits:
             # Handle case with no available traits.
             trait_score = 0.0
        else:
            matched_traits_count =
            len(requested_traits.intersection(aggregated_traits))
            available_traits_count = len(aggregated_traits)
            # Calculate ratio: (count of requested traits present) /
            # (count of all available traits)
            trait_score = (
                (matched_traits_count / available_traits_count)
                if available_traits_count > 0 else 0.0)

        # 4. Weight Composition
        # The goal is to select the most boring host (least traits/resources).
        # A lower resource_score (closer to 0) means higher utilization
        # ratios (closer to 1), which happens when requested amount is close
        # to free capacity. This is not what we want. We want a higher score
        # for less utilized resources. The current formula 1 - (requested /
        # available) gives a higher score for less utilized. This seems
        # correct for the resource part.

        # For traits, a higher trait_score means the requested traits and
        # available traits are closely aligned. As a host will never have fewer
        # traits than requested, we want to select a host where the trait_score
        # is as close to 1 as possible because this means there are few extra
        # traits that were not requested. We want a higher score as (more
        # boring = less unrequested traits). The current formula (matched
        # traits) / (available traits) gives a higher score for more matched
        # traits.

        # Combine resource and adjusted trait scores
        # A higher combined score indicates a more "boring" host (less
        # utilized resources, fewer requested traits present).
        combined_score = (resource_score + trait_score) / 2.0

        # Apply multiplier from scheduler configuration.
        weight = combined_score * self.weight_multiplier(host_state)

        return weight


Configuration
^^^^^^^^^^^^^^
Configure via ``nova.conf``:

.. code-block:: ini

    [filter_scheduler]
    # Weight multiplier (default: 0.0)
    resource_provider_weight_multiplier = 1.0

    # Resource classes to consider (default: all)
    resource_provider_resources = VCPU_SHARES,MEMORY_MB

    # Traits to require (default: all)
    resource_provider_traits = HW_CPU_X86_AVX2,HW_CPU_X86_SSE

Example calculation for a VM requesting 2 VCPUs and SSD trait::

  HostA resources: VCPU avail=4
  HostA traits: SSD, HW_CPU_X86_AVX2
  Resource score: 1 - (2/4) = 0.5
  Trait score: (1/2) = 0.5
  Final weight: (0.5 + 0.5)/2 = 0.5

Diagrams and Examples
---------------------

* Virtual Machine Resource Contention Diagram:

  .. code-block:: bash

    +-----------------+   +-----------------+   +-----------------+
    |   Gold Instance |   |  Silver Inst 1  |   |  Silver Inst 2  |
    |   (1 vCPU)      |   |  (1 vCPU)       |   |  (1 vCPU)       |
    |   Shares: 100   |   |  Shares: 50     |   |  Shares: 50     |
    +-----------------+   +-----------------+   +-----------------+
            |                     |                     |
            v                     v                     v
    +-----------------+   +-----------------+   +-----------------+
    |  Bronze Inst 2  |   |  Bronze Inst 3  |   |  Bronze Inst 1  |
    |  (1 vCPU)       |   |  (1 vCPU)       |   |  (1 vCPU)       |
    |  Shares: 25     |   |  Shares: 25     |   |  Shares: 25     |
    +-----------------+   +-----------------+   +-----------------+
            |                     |                     |
            +----------+----------+----------+----------+
                      |                     |
                      v                     v
            +---------------------------------+
            |     Aggregate Host CPU Pool     |
            | (Managed by CFS based on shares)|
            +---------------------------------+
                      ^     ^     ^     ^
                      |     |     |     |
            +---------------------------------+
            |    Host CPU 1   |    Host CPU 2 |
            |    Host CPU 3   |    Host CPU 4 |
            +---------------------------------+

  Total Host Capacity: 400 shared units (4 CPUs * 100 units/CPU)
  Total Allocated Shares: 100 (Gold) + 100 (Silver) + 75 (Bronze) = 275 units
  Remaining Capacity: 400 - 275 = 125 units

  .. note::
     Under contention, CPU time is allocated proportionally to shares across all
     cores.
     At any given time 2 vms can reside on the same core, but the Linux
     scheduler will
     allocate cpu time proportionally to the shares.

  .. note::
     Gold (100 sh): ~36.4% of total CPU time
     Silver (50 sh each): ~18.2% of total CPU time each
     Bronze (25 sh each): ~9.1% of total CPU time each


* Configuration Examples: Provide examples of:

  - Flavor extra specs using share multipliers:

    .. code-block:: console

        # Create flavors with different CPU share multipliers
        $ openstack flavor create --vcpus 1 --ram 1024 --disk 10 bronze
        $ openstack flavor set bronze --property
        quota:cpu_shares_multiplier=25.0

        $ openstack flavor create --vcpus 1 --ram 2048 --disk 20 silver
        $ openstack flavor set silver --property
        quota:cpu_shares_multiplier=50.0

        $ openstack flavor create --vcpus 1 --ram 4096 --disk 40 gold
        $ openstack flavor set gold --property quota:cpu_shares_multiplier=100.0

  - Runtime configuration via nova.conf:

    .. code-block:: ini

        [libvirt]
        report_vcpu_shares = true
        vcpu_share_multiplier = 100  # Default value, scales physical cores
                                     # to shares

        [compute]
        cpu_shared_set = 1-112  # Map to physical cores

Alternatives
------------

* **Reporting VCPU_SHARE via provider.yaml:**
  Initial approach required manual configuration of provider.yaml files to
  report VCPU_SHARES inventory:

  .. code-block:: yaml

      resource_provider:
        name: compute-node01
        inventories:
          VCPU_SHARES:
            total: 100000
            allocation_ratio: 1.0

  **Limitations:**
  - Static configuration unable to adapt to host changes.
  - No direct correlation with cgroups capacity.
  - Manual maintenance overhead.

* **CPU Service Level via Custom Resource Classes (Original Proposal):**
  The initial approach proposed creating multiple custom resource classes
  (e.g., ``CUSTOM_VCPU_GOLD``, ``CUSTOM_VCPU_SILVER``) to represent
  performance tiers. This required:

  - Defining separate resource classes for each service tier
  - Complex flavor extra specs combining vCPU and custom resources
  - Per-host inventory management for each tier.

  **Reason for revision:** Simplified maintenance through standardized
  resource class while maintaining Placement-based guarantees through
  share weighting.

* **Watcher-Based Enforcement (Reactive):** OpenStack Watcher could be
  used to monitor per-host, per-tier CPU usage and reactively migrate
  instances to rebalance the load and enforce policies. This would involve
  developing a custom Watcher strategy that queries Nova/Placement for
  allocations and potentially Telemetry for actual CPU utilization.
* **Pros:** Can provide ongoing compliance and potentially optimize
  placement based on real-time performance metrics. Can act as a safety
  net if initial placement is not strictly enforced.
* **Cons:** Reactive approach means potential for temporary SLO breaches
  until Watcher acts. Relies on accurate telemetry data. Adds complexity
  with an additional service to deploy and manage.
* **Reason for not choosing as primary:** While Watcher can complement
    the Nova scheduler approach by providing ongoing monitoring and
    optimization, it is not ideal for strict deterministic enforcement
    at the time of placement. The Placement-based approach prevents
    violations from occurring in the first place.

* **Static Host Aggregates:** Using host aggregates to group hosts by
  intended service tier capacity and relying on aggregate-specific
  scheduler filters.
* **Pros:** Relatively simple to implement for basic tiering.
* **Cons:** Less flexible than Placement custom resources for dynamic
inventory and fine-grained control. Does not easily support mixing
tiers on the same host with guaranteed per-tier capacity limits.
Requires manual management of host aggregate membership.

* **Percentage-Based Allocation:** Allowing users to request a percentage
  of host CPU resources instead of raw shares.
* **Pros:** Potentially more intuitive for users.
* **Cons:** Translating a percentage request to a consistent share value
  across hosts with varying total CPU capacity can be complex. May not
  align directly with the underlying cgroup share mechanism which is
  relative.


Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

* The Horizon dashboard has limited support for displaying inventories
  of resource classes to administrators. While it could be extended to
  display ``VCPU_SHARES``, that is not currently planned and is out of
  scope. Administrators will be able to query this information via the
  existing OpenStack client, so no end user impact is expected.

Performance Impact
------------------

* The overhead of cgroup management by the compute driver should be
  minimal, as it leverages existing kernel features and libvirt
functionality. As a result, it effectively amounts to reporting
inventories of ``VCPU_SHARES``, weighing hosts based on the available
resource classes in the provider summaries and generating the
appropriate libvirt xml to have it enforce cpu shares using the Linux
scheduler.

* The primary performance impact will be on instance performance based on
  their allocated shares and the overall host load. Instances with higher
  shares are expected to receive a proportionally larger amount of CPU
  time under contention.
* The scheduler's interaction with Placement to check resource inventory
  should have minimal performance impact, as Placement is designed for
  scalable resource tracking. However, the overhead of the new
  ``ResourceProviderWeigher`` will impact scheduler performance. To
  mitigate this impact, it will be disabled by default via a default
  multiplier of ``0.0``.

Other deployer impact
---------------------

None

Developer impact
----------------

None

Data model impact
-----------------
None

while the Hoststate object in the scheudler will need to be extened
to provide access to the Provider Summeries. That is an internal
change and does not affect the overall datamodel.

REST API impact
---------------

None

Upgrade impact
--------------

* Careful consideration is needed for rolling upgrades to ensure
  compatibility with existing instances.
* Following the pattern established by the pci in Placement feature, a
  new config option to report CPU shares in Placement will be added to
  the libvirt section and a corresponding config option to request CPU
  shares will be added to the filter scheduler section.

 .. code-block:: ini

   [libvirt]
   report_vcpu_shares = true|false  # Default: false
   vcpu_share_multiplier = 100  # Default value

   [filter_scheduler]
   query_placement_for_vcpu_shares = True | False (Default)

 As with pci in Placement when share reporting is enabled it should not be
 disabled again. When activated, on startup, the nova compute agent will need
 to reshape existing allocations for instances in Placement to request CPU
 shares if they use either the ``quota:cpu_shares`` or
 ``quota:cpu_shares_multiplier`` flavor extra specs.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  sean-k-mooney

Other contributors:
  deepseek-r1
  gemini-flash-2.5

Feature Liaison
---------------

Feature liaison:
  sean-k-mooney

Work Items
----------

1. Implement ``VCPU_SHARES`` resource class support in Placement.
2. Add libvirt driver integration for automatic inventory reporting.
3. Develop ``ResourceProviderWeigher`` scheduler component.
4. Create new flavor extra spec validation logic.
5. Implement configuration options for upgrade compatibility.
6. Add documentation and release notes.

Dependencies
============

* A new resource class needs to be added to os-resource classes.

Testing
=======

Comprehensive testing is required to ensure the feature functions correctly and
provides the expected performance guarantees.

* **Unit Tests:**
  Unit tests will be provided to cover the basic operations of the weigher,
  xml generation and other relevant components.
* **Functional Tests:** Verify the interaction between different Nova
  components (API, scheduler, compute) and the hypervisor/Placement.
  New functional tests will be required to validate the reshape logic.
  They can also be used to assert that shares are reported correctly
 to Placement and that the translation mechanism from flavor extra spec
 to allocations works as expected.

* **Tempest Tests:**

  In general, as this primarily builds on standard Placement logic, there
  is not much additional Tempest logic required. With that said, it
  should be possible to add a Tempest test that creates a flavor that
  uses the new flavor extra specs and assert that the instance allocations
  contain the relevant VCPU_SHARES. Alternatively, we can tweak the
  flavors in one of our existing jobs to use this feature for all tests.

Documentation Impact
====================

* Update the Nova API reference to document the changes to the server
  creation, resize, and show details APIs, including the new ``cpu_shares``
  parameter and its behavior.
* Update the Nova administrator guide to explain how to configure compute
  nodes to expose custom tier-specific VCPU resources to Placement via
  provider config files, how to define flavors with appropriate extra
  specs, and how to configure related policy rules.
* Update the Nova user guide to explain how users can request CPU
  performance service levels by selecting flavors with specific CPU
  shares or by specifying the ``cpu_shares`` parameter directly (if
  allowed by policy).

References
==========

* `OpenStack cgroup tiering README <https://github.com/gprocunier/openstack-cgroup-tiering/blob/b205e34c0c62fa6cd451a44659c648595f0d19ae/README.md>`_

History
=======

.. list-table:: Revisions
   :header-rows: 1

  * - Release Name
    - Description
  * - 2025.2 Flamingo
    - Introduced CPU performance service levels using CPU shares and
      Placement standardized resource classes.