sequenceDiagram
    title Pod Startup Kubernetes Network Driver
    autonumber


    participant APIServer as API Server
    participant Kubelet
    participant ContainerRuntime as Container Runtime (CRI)
    participant CNIPlugin as CNI Plugin
    participant NetworkDriver as Kubernetes Network Driver

    NetworkDriver->>APIServer: Publish ResourcesSlice with available resources

    Kubelet->>APIServer: Watch for Pods assigned to its node
    APIServer-->>Kubelet: Notification: New Pod scheduled requesting a published resource
    note over Kubelet: Admission control<br>before starting the Pod

    Kubelet-->>+NetworkDriver: NodePrepareResources()
    Note over NetworkDriver: Obtain the ResourceClaim with the<br>Request and Config
    NetworkDriver-->>-Kubelet: Resources successfully allocated (precreate network interface, ...)

    Kubelet->>+ContainerRuntime: RunPodSandbox
    note over ContainerRuntime: Creates a new network namespace<br>for the Pod.

    ContainerRuntime->>+CNIPlugin: Execute CNI ADD command
    note right of ContainerRuntime: Passes NetNS path, ContainerID, and CNI_ARGS

    CNIPlugin->>CNIPlugin: Configure Pod Networking<br/>(e.g., create veth, assign IP via IPAM, set routes)
    CNIPlugin-->>-ContainerRuntime: Return CNI Result (JSON)
    note right of ContainerRuntime: { "ips": [{"address": "10.244.1.5/24", ...}], "interfaces": [...] }

    ContainerRuntime->>NetworkDriver: NRI: RunPodSandbox()
    Note right of ContainerRuntime: Pass Pod metadata, IPs, network namespace
    NetworkDriver-->>NetworkDriver:  Allocate and configure the<br>requested network resources
    NetworkDriver-->>ContainerRuntime:  Network resources allocated to the network namespaces
    
    ContainerRuntime-->>-Kubelet: RunPodSandboxResponse(SandboxID)

    Kubelet->>+ContainerRuntime: PodSandboxStatus()
    ContainerRuntime-->>-Kubelet: PodSandboxStatusResponse
    note over ContainerRuntime: PodSandboxStatusResponse.Network: Ip, AdditionalIPs


    loop for every container
        Kubelet->>NetworkDriver: Allocate()
        note over NetworkDriver: run device specific operations
        NetworkDriver-->>Kubelet: AllocateResponse

        Kubelet->>+ContainerRuntime: CreateContainer()
        note right of ContainerRuntime: The application container is started<br/>within the configured network namespace.
        ContainerRuntime-->>-Kubelet: Containers started successfully
    end

    Kubelet->>APIServer: PATCH /api/v1/pods/{name}/status
    note over Kubelet, APIServer: Update Pod status with the assigned IP address<br/>and set phase to 'Running'.