# ipi-installation-bastion
Download installer

```bash
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.18.1/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.18.1/openshift-install-linux.tar.gz
tar -xvf openshift-client-linux.tar.gz -C /usr/bin/
tar -xvf openshift-install-linux.tar.gz -C /usr/bin/
```
Instal packages
```bash
dnf install -y dnsmasq haproxy
```
Edit /etc/haproxy/haproxy.cfg file.
```bash

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    maxconn     20000
    log         /dev/log local0 info
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# Defaults
#---------------------------------------------------------------------
defaults
    log                     global
    mode                    http
    option                  httplog
    option                  dontlognull
    option http-server-close
    option redispatch
    option forwardfor       except 127.0.0.0/8
    retries                 3
    maxconn                 20000
    timeout http-request    10s
    timeout http-keep-alive 10s
    timeout check           10s
    timeout connect         40s
    timeout client          300s
    timeout server          300s
    timeout queue           50s

#---------------------------------------------------------------------
# HAProxy stats page
#---------------------------------------------------------------------
listen stats
    bind :9000
    stats uri /stats
    stats refresh 10s

#---------------------------------------------------------------------
# Kubernetes API Server (6443)
#---------------------------------------------------------------------
frontend k8s_api_frontend
    bind :6443
    mode tcp
    default_backend k8s_api_backend

backend k8s_api_backend
    mode tcp
    balance source
    server bootstrap 192.168.122.139:6443 check
    server master1   192.168.122.140:6443 check
    server master2   192.168.122.141:6443 check
    server master3   192.168.122.142:6443 check

#---------------------------------------------------------------------
# Machine Config Server (22623)
#---------------------------------------------------------------------
frontend ocp_machine_config_frontend
    bind :22623
    mode tcp
    default_backend ocp_machine_config_backend

backend ocp_machine_config_backend
    mode tcp
    balance source
    server bootstrap 192.168.122.139:22623 check
    server master1   192.168.122.140:22623 check
    server master2   192.168.122.141:22623 check
    server master3   192.168.122.142:22623 check

#---------------------------------------------------------------------
# Ingress (80/443)
#---------------------------------------------------------------------
frontend ocp_http_ingress_frontend
    bind :80
    mode tcp
    default_backend ocp_http_ingress_backend

backend ocp_http_ingress_backend
    mode tcp
    balance source
    server master1   192.168.122.140:80 check
    server master2   192.168.122.141:80 check
    server master3   192.168.122.142:80 check

frontend ocp_https_ingress_frontend
    bind :443
    mode tcp
    default_backend ocp_https_ingress_backend

backend ocp_https_ingress_backend
    mode tcp
    balance source
    server master1   192.168.122.140:443 check
    server master2   192.168.122.141:443 check
    server master3   192.168.122.142:443 check
```
dnsmasqfile - /etc/dnsmasq.d/ocp.conf
```bash
# ======================================
# dnsmasq for OpenShift IPI (static IPs)
# ======================================

# Listen on bastion's main NIC (adjust if different)
interface=ens192
bind-interfaces

# Your OpenShift cluster domain
domain=ocp.lab.local
expand-hosts
domain-needed
bogus-priv

# Forward all unknown queries upstream
server=8.8.8.8

# --- DNS entries ---

# API + Ingress VIPs (adjust if you pick different ones)
address=/api.ocp.lab.local/192.168.122.138
address=/api-int.ocp.lab.local/192.168.122.138
address=/.apps.ocp.lab.local/192.168.122.138

# Bootstrap + Masters
address=/bootstrap.ocp.lab.local/192.168.122.139
address=/master1.ocp.lab.local/192.168.122.140
address=/master2.ocp.lab.local/192.168.122.141
address=/master3.ocp.lab.local/192.168.122.142

# etcd hostnames (required for control-plane only)
address=/etcd-0.ocp.lab.local/192.168.122.140
address=/etcd-1.ocp.lab.local/192.168.122.141
address=/etcd-2.ocp.lab.local/192.168.122.142

# etcd SRV records
srv-host=_etcd-server-ssl._tcp.ocp.lab.local,etcd-0.ocp.lab.local,2380,0,10
srv-host=_etcd-server-ssl._tcp.ocp.lab.local,etcd-1.ocp.lab.local,2380,0,10
srv-host=_etcd-server-ssl._tcp.ocp.lab.local,etcd-2.ocp.lab.local,2380,0,10
```
Start haproxy and dnsmasq services.
```bash
systemctl disable firewalld --now
setsebool -P haproxy_connect_any on
systemctl enable haproxy --now
systemctl enable --now dnsmasq
```
- Download the vcenter CA cert and import into anchors.

open vcneter console and download the zip file.

extract the zip and import in anchors.
```bash
unzip download.zip
cp certs/lin/1e08b361.0 /etc/pki/ca-trust/source/anchors/vcenter-root-ca.crt
update-ca-trust extract
curl https://vsphere.local
```

Create this install-config file.
```bash
apiVersion: v1
baseDomain: lab.local
metadata:
  name: ocp

platform:
  vsphere:
    vcenter: vsphere.local
    username: administrator@vsphere.local
    password: <REPLACE_WITH_VCENTER_PASSWORD>
    datacenter: Datacenter
    cluster: cluster1
    defaultDatastore: datastore1
    network: "VM Network"
    apiVIP: 192.168.122.138
    ingressVIP: 192.168.122.137
    cacert: |-
      -----BEGIN CERTIFICATE-----
      MIIEEzCCAvugAwIBAgIJAORtUGHjkczNMA0GCSqGSIb3DQEBCwUAMIGUMQswCQYD
      VQQDDAJDQTEXMBUGCgmSJomT8ixkARkWB3ZzcGhlcmUxFTATBgoJkiaJk/IsZAEZ
      FgVsb2NhbDELMAkGA1UEBhMCVVMxEzARBgNVBAgMCkNhbGlmb3JuaWExFjAUBgNV
      BAoMDXZzcGhlcmUubG9jYWwxGzAZBgNVBAsMElZNd2FyZSBFbmdpbmVlcmluZzAe
      Fw0yNTEwMTgwNjA1NDlaFw0zNTEwMTYwNjA1NDlaMIGUMQswCQYDVQQDDAJDQTEX
      MBUGCgmSJomT8ixkARkWB3ZzcGhlcmUxFTATBgoJkiaJk/IsZAEZFgVsb2NhbDEL
      MAkGA1UEBhMCVVMxEzARBgNVBAgMCkNhbGlmb3JuaWExFjAUBgNVBAoMDXZzcGhl
      cmUubG9jYWwxGzAZBgNVBAsMElZNd2FyZSBFbmdpbmVlcmluZzCCASIwDQYJKoZI
      hvcNAQEBBQADggEPADCCAQoCggEBAKdw2fe4SLZAoHpbhU2tC4UneKMCB+i8WUkT
      M5zB49XAzMVS6MEJiuRHh20ZYALE3cMPhNibzmGWvHlFArthrEzE3utISHuzGNh5
      2vYZ61R7R4pbyP4EMnXMJirEcAHnmg5cXo8eg6FlHMpJzWDxp8ZVM38BhoIxfHGS
      kYtKKleDRqV28aiV1bo838OjSrHOv35cNssl61nLPs7xs3sccktCc+9FDciKiFCW
      J1jCC1Nk8AwHFFDapRoZn1WpYtW3C6CjDwBVW5pM+JwbSpefs2xa2tSAvDIGI00d
      b1ZWCRc8EqwV345sLuJ00v8dy0ZSTPWI7qRUUoDKiA83qaNcE88CAwEAAaNmMGQw
      HQYDVR0OBBYEFOUYRgSCXFRxwNb67GgoMHXlaKVSMB8GA1UdEQQYMBaBDmVtYWls
      QGFjbWUuY29thwR/AAABMA4GA1UdDwEB/wQEAwIBBjASBgNVHRMBAf8ECDAGAQH/
      AgEAMA0GCSqGSIb3DQEBCwUAA4IBAQA09w1WLGpYqkxtd+ELIH5DiUNIPpFM/+Xs
      GUIziTfcGyU4JaqPAPrFreWRWDQQzNfRqDv7qpUWI6hHM54C5JWiAS0K85KAikAV
      Lsr8+vC8okRaWJ2NsYkeEA4sENFTicwAx5xiDhg6EqDYEzd8/Q8Vp1W3iHr3C1/y
      I2R2Y3dyeVGuuLqF+F/U0s7Aq+qXJQ2QYR0awsnZILGRcoN0YE+mxLZLks1yHg2y
      QEpKbqmnmC/APev+nmpM1Qfy8FYzYElii+85yNOlYrsUuhdk52Mp//A7OKaLtgxi
      4lh1vNum5PtQ7gwU+zJbTVramk8+fGq9rkVUZo+dpYjfQ2WtleWM
      -----END CERTIFICATE-----
    hosts:
      - role: bootstrap
        networkDevice:
          ipAddrs:
            - 192.168.122.139/24
          gateway: 192.168.122.1
          nameservers:
            - 192.168.122.86

      - role: control-plane
        networkDevice:
          ipAddrs:
            - 192.168.122.140/24
          gateway: 192.168.122.1
          nameservers:
            - 192.168.122.86

      - role: control-plane
        networkDevice:
          ipAddrs:
            - 192.168.122.141/24
          gateway: 192.168.122.1
          nameservers:
            - 192.168.122.86

      - role: control-plane
        networkDevice:
          ipAddrs:
            - 192.168.122.142/24
          gateway: 192.168.122.1
          nameservers:
            - 192.168.122.86

pullSecret: '<REPLACE_WITH_YOUR_PULL_SECRET>'
sshKey: '<REPLACE_WITH_YOUR_PUBLIC_SSH_KEY>'
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
compute:
- name: worker
  replicas: 0
```
Create the cluster with this command.
```bash
openshift-install create cluster --dir ~/ocp-install --log-level=info
```
