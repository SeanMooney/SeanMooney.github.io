===============================================================================
CPU Performance Service Levels using CPU Shares
===============================================================================

Problem Description
-------------------

Users currently lack a standardized and reliable method within Nova to request and guarantee specific levels of CPU performance for their instances. While existing mechanisms like CPU pinning or NUMA topology awareness provide some control, they are often complex to manage and do not offer a simple, quantifiable way to express desired performance tiers across a shared infrastructure. This leads to unpredictable performance, especially in oversubscribed environments, making it difficult for users to run performance-sensitive workloads with confidence.

The goal is to introduce a mechanism that allows users to request CPU performance service levels based on CPU shares, providing a more predictable and manageable approach to CPU resource allocation and performance guarantees.

Use Cases
~~~~~~~~~

*   A user wants to launch a database server instance and ensure it receives a guaranteed minimum level of CPU performance, even under high load from other instances on the same host.
*   A user is running a batch processing job that requires a consistent amount of CPU time to complete within a specific timeframe. They need to request a performance tier that guarantees this.
*   A cloud administrator wants to offer different tiers of compute service (e.g., "Bronze", "Silver", "Gold") based on guaranteed CPU performance, allowing users to choose the level that best suits their workload and budget.
*   A user needs to temporarily boost the CPU performance of an existing instance to handle a peak load, and then revert to the standard performance level.

Proposed Change
---------------

Introduce a new mechanism in Nova that leverages CPU shares (using control groups - cgroups) to define and enforce CPU performance service levels. Users will be able to specify a desired CPU share value when launching or resizing an instance, which will translate to a relative proportion of available CPU resources on the host.

This approach provides a flexible and granular way to manage CPU performance, allowing for oversubscription while still providing a mechanism for performance guarantees based on the allocated shares.

CPU Share Management
~~~~~~~~~~~~~~~~~~~~

*   **Allocation:** CPU shares will be allocated to instances based on a new flavor extra spec or image metadata. The value will represent a relative weight, with higher values indicating a greater proportion of CPU resources. A default share value will be applied if none is specified.
*   **Tracking:** Nova will track the allocated CPU shares for each instance and the total shares allocated on a compute host. This information will be used by the scheduler and compute drivers.
*   **Adjustment:** Users will be able to adjust the CPU share allocation for a running instance via a resize operation.
*   **Conflicts and Scaling:** The system will need to handle potential conflicts when the total requested shares exceed the available CPU resources on a host. This will involve prioritization based on share values and potentially admission control. Scaling will be handled by the underlying cgroup mechanism.

Performance Guarantee Enforcement
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

*   **Mechanism:** CPU shares, managed via cgroups, provide a mechanism for proportional sharing of CPU time. While not a strict reservation, higher share values translate to a greater opportunity to consume CPU cycles when the system is contended.
*   **Hypervisor Limitations:** The implementation will need to consider how different hypervisors (e.g., KVM, VMware) expose and manage CPU shares or similar concepts. Abstraction layers may be required.
*   **Oversubscription:** In oversubscribed environments, the performance guarantee is relative to the total shares allocated. Instances with higher shares will receive a larger portion of the available CPU time.
*   **Workload Characteristics:** The actual performance experienced by an instance will still depend on its workload characteristics. The system guarantees a proportion of CPU time, not a specific throughput or latency.
*   **Mitigation Strategies:** Monitoring CPU utilization and performance metrics will be crucial. If performance degradation is detected for instances with guaranteed shares, mitigation strategies could include workload migration, reducing oversubscription, or alerting administrators.

User Interaction
~~~~~~~~~~~~~~~~

*   **RESTful API:**
    *   Extend the server creation and resize APIs to accept a `cpu_shares` parameter (or similar) in the request body.
    *   Add a new API endpoint to view the current CPU share allocation for an instance.
    *   Potentially add an API endpoint to view the total allocated CPU shares on a compute host (for administrators).

    Example API Request (Create Server):

    .. code-block:: json

        {
            "server": {
                "name": "my-performance-instance",
                "imageRef": "...",
                "flavorRef": "...",
                "networks": [...],
                "cpu_shares": 2048
            }
        }

    Example API Response (Show Server Details):

    .. code-block:: json

        {
            "server": {
                ...,
                "cpu_shares": 2048,
                ...
            }
        }

*   **UI Interfaces:**
    *   Update the Horizon dashboard to include an input field for CPU shares during instance launch and resize.
    *   Display the allocated CPU shares on the instance details page.

Nova Integration
~~~~~~~~~~~~~~~~

*   **Scheduler:** The scheduler will need to be aware of the requested `cpu_shares` and the available CPU resources and allocated shares on compute hosts to make informed placement decisions. A new scheduler filter or weigh
    er might be required.
*   **Compute:** The compute driver will be responsible for configuring the hypervisor (e.g., libvirt) to set the appropriate CPU shares for the instance's cgroup.
*   **API:** The API layer will handle the user requests for specifying and viewing CPU shares, performing validation, and interacting with the scheduler and compute components.
*   **Database:** The Nova database schema will need to be updated to store the `cpu_shares` value for each instance.
*   **Impacts and Required Modifications:** This feature will require modifications to the Nova API, scheduler, compute drivers, and database schema. Careful consideration of backward compatibility and rolling upgrades will be necessary.

Oversubscription Handling
~~~~~~~~~~~~~~~~~~~~~~~~~

*   **Strategy:** The primary strategy for oversubscription handling will rely on the proportional sharing provided by CPU shares. Instances with higher share values will receive a proportionally larger amount of CPU time when the system is oversubscribed.
*   **Share Prioritization:** The cgroup mechanism inherently prioritizes instances based on their share values.
*   **Performance Impact under High Load:** Under extreme oversubscription, even instances with high share values may experience performance degradation. Monitoring and alerting will be crucial to identify such situations. Administrators may need to adjust oversubscription ratios or migrate workloads.

Testing Procedures
------------------

*   **Unit Tests:** Unit tests will be required for new or modified code in the API, scheduler, compute drivers, and database layer.
*   **Integration Tests:** Integration tests will verify the interaction between different Nova components (API, scheduler, compute) and the hypervisor when creating, resizing, and deleting instances with specified CPU shares.
*   **Performance Tests:** Performance tests will be essential to validate that the allocated CPU shares translate to the expected relative performance under various load conditions and oversubscription levels. These tests should measure metrics like CPU utilization, throughput, and latency for instances with different share values. Tempest tests should be developed to cover the user-facing API interactions and basic functionality.

Diagrams and Examples
----------------------

*   **Share Allocation Workflow Diagram:** An ASCII diagram illustrating the flow of a request to create an instance with specified CPU shares, showing how the request is handled by the API, scheduler, and compute components, and how the CPU shares are configured on the compute host via cgroups.
*   **Performance Guarantee Enforcement Diagram:** An ASCII diagram illustrating a scenario with multiple instances on a single host with different CPU share allocations, showing how CPU time is distributed among them under contention.
*   **API Usage Examples:** Include detailed examples of the RESTful API requests and responses for creating, showing details of, and resizing instances with CPU shares.
*   **Configuration Examples:** Provide examples of how to configure flavors or image metadata to include default or allowed CPU share values.

Alternatives
------------

(Placeholder for analysis of alternative approaches, e.g., strict CPU reservation, percentage-based allocation, etc.)

Impacts
-------

*   **Data Model Impact:** Add a `cpu_shares` column to the instances table.
*   **REST API Impact:** Modify the create server and resize server API requests and responses. Add new API endpoints for viewing CPU share information. This requires the `APIImpact` flag.
*   **Security Impact:** (Placeholder for security considerations, e.g., potential for denial-of-service if not properly validated).
*   **Notifications Impact:** Versioned notifications may be required for changes to instance CPU share allocations.
*   **Performance Impact:** The overhead of cgroup management should be minimal. The primary performance impact will be on instance performance based on their allocated shares and the overall host load.

Upgrade Considerations
----------------------

Careful consideration is needed for rolling upgrades to ensure compatibility with existing instances and APIs. The new `cpu_shares` field should be nullable in the database initially, and default values applied for instances created before the feature is fully deployed.

Implementation Plan
-------------------

*   Phase 1: Database schema changes and API modifications to accept and store `cpu_shares`.
*   Phase 2: Scheduler integration to consider `cpu_shares` during instance placement.
*   Phase 3: Compute driver implementation to configure cgroups with the specified CPU shares.
*   Phase 4: Add API endpoints for viewing CPU share information.
*   Phase 5: Implement Horizon UI changes.
*   Phase 6: Develop comprehensive unit, integration, and performance tests.
*   Phase 7: Update documentation.

Testing Requirements
--------------------

See "Testing Procedures" section above. Tempest tests are required for API coverage.

Documentation Impact
--------------------

Update the Nova API reference, the Nova administrator guide (for configuration), and the Nova user guide (for requesting CPU shares).