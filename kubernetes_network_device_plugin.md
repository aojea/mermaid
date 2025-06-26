sequenceDiagram
    title Pod Startup Device Plugin
    autonumber


    participant APIServer as API Server
    participant Kubelet
    participant ContainerRuntime as Container Runtime (CRI)
    participant CNIPlugin as CNI Plugin
    participant DevicePlugin as Device Plugin

    Kubelet->>DevicePlugin: ListAndWatch()
    Kubelet->>APIServer: Publish Device Plugin extended Resources

    Kubelet->>APIServer: Watch for Pods assigned to its node
    APIServer-->>Kubelet: Notification: New Pod scheduled requesting an extended Resource
    note over Kubelet: Admission control<br>before starting the Pod

    Kubelet->>+ContainerRuntime: RunPodSandbox
    note over ContainerRuntime: Creates a new network namespace<br>for the Pod.

    ContainerRuntime->>+CNIPlugin: Execute CNI ADD command
    note right of ContainerRuntime: Passes NetNS path, ContainerID, and CNI_ARGS

    CNIPlugin->>CNIPlugin: Configure Pod Networking<br/>(e.g., create veth, assign IP via IPAM, set routes)
    CNIPlugin-->>-ContainerRuntime: Return CNI Result (JSON)
    note right of ContainerRuntime: { "ips": [{"address": "10.244.1.5/24", ...}], "interfaces": [...] }

    
    ContainerRuntime-->>-Kubelet: RunPodSandboxResponse(SandboxID)

    Kubelet->>+ContainerRuntime: PodSandboxStatus()
    ContainerRuntime-->>-Kubelet: PodSandboxStatusResponse
    note over ContainerRuntime: PodSandboxStatusResponse.Network: Ip, AdditionalIPs


    loop for every container
        Kubelet->>DevicePlugin: Allocate()
        note over DevicePlugin: run device specific operations
        DevicePlugin-->>Kubelet: AllocateResponse

        Kubelet->>+ContainerRuntime: CreateContainer()
        note right of ContainerRuntime: The application container is started<br/>within the configured network namespace.
        ContainerRuntime-->>-Kubelet: Containers started successfully
    end

    Kubelet->>APIServer: PATCH /api/v1/pods/{name}/status
    note over Kubelet, APIServer: Update Pod status with the assigned IP address<br/>and set phase to 'Running'.