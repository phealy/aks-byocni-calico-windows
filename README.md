# Installing Calico/VXLAN on AKS BYOCNI with Windows nodepools.

1. Deploy an AKS cluster with BYOCNI mode.
1. Configure Calico using VXLAN.
1. Add a Windows 2019 or 2022 nodepool.
1. Configure [strict affinity for the ip pools](https://projectcalico.docs.tigera.io/getting-started/windows-calico/quickstart#configure-strict-affinity-for-clusters-using-calico-networking).
1. Follow the instructions on [Installing Calico for Windows using HostProcess Containers](https://projectcalico.docs.tigera.io/getting-started/windows-calico/quickstart#install) with the following modifications:
    1. In calico-windows.yaml, make the following changes:
        1. Set K8S_SERVICE_CIDR and K8S_NAME_SERVERS appropriately for your environment.
        1. Set CNI_BIN_DIR to "C:\\k\\azurecni\\bin"
        1. Set CNI_CONF_DIR to "C:\\k\\azurecni\\netconf"
        1. Add an initContainer to the calico-node-windows DaemonSet to delete the Azure CNI configuration that's present by default:
           ```yml
           - name: delete-azure-cni
             image: calico/windows:v3.24.5
             command:
             - powershell.exe
             - -Command
             - "Get-Item -ErrorAction SilentlyContinue C:\\k\\azurecni\\netconf\\10-azure.conf | Remove-Item; exit 0"
             imagePullPolicy: Always
           ```
    1. The default installation of kube-proxy will not start without Azure CNI, so you have to bring your own kube-proxy per step 7 on the Calico install instructions, referencing [windows-kube-proxy.yaml](windows-kube-proxy.yaml) for the following modifications:
        1. In the ConfigMap, remove the lines that modify kubeConfig and set `$kubeConfigPath = "C:\k\config"`
        1. Set K8S_VERSION appropriately in the environment variables
        1. Add a tolarations block:
            ```yml
            tolerations:        
            - operator: Exists  
              effect: NoSchedule
            ```
        1. Duplicate the Daemonset, using the following configurations:
            1. Windows Server 2019
                - image: mcr.microsoft.com/windows/nanoserver:ltsc2019
                - nodeSelector: kubernetes.azure.com/os-sku: Windows2019
            1. Windows Server 2022
                - image: mcr.microsoft.com/windows/nanoserver:ltsc2022
                - nodeSelector: kubernetes.azure.com/os-sku: Windows2022
 
# Outstanding issues

`cloud-node-manager-windows` is currently unable to reach IMDS from its pod - how can we build a route to 169.254.169.254?