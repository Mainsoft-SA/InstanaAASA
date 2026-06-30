

```
-------------------------------------------
sudo su -
hostnamectl set-hostname instana-0

#10.250.0.7 instana-0
#10.250.0.6 instana-1
#10.250.0.8 instana-2

lsblk

for disk in vdd; do
   echo "make filesystem for $disk"
   mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/$disk
done

mkdir -p /mnt/instana/stanctl/

blkid /dev/vdd

/dev/vdd: UUID="c1da93f1-a95b-43d9-80dd-4688a4ba32b9" TYPE="ext4"

vi /etc/fstab

UUID=c1da93f1-a95b-43d9-80dd-4688a4ba32b9    /mnt/instana/stanctl/       ext4    discard,defaults,nofail        0 0

systemctl daemon-reload
mount -a

lsblk /dev/vdd

instana-0
mkdir -p /mnt/instana/stanctl/objects

instana-1
mkdir -p /mnt/instana/stanctl/data
mkdir -p /mnt/instana/stanctl/metrics
mkdir -p /mnt/instana/stanctl/analytics


sh -c 'echo vm.swappiness=0 >> /etc/sysctl.d/99-stanctl.conf' && sysctl -p /etc/sysctl.d/99-stanctl.conf
sh -c 'echo fs.inotify.max_user_instances=8192 >> /etc/sysctl.d/99-stanctl.conf' && sysctl -p /etc/sysctl.d/99-stanctl.conf
grubby --args="transparent_hugepage=never" --update-kernel ALL
systemctl reboot

sudo su -
cat /sys/kernel/mm/transparent_hugepage/enabled
 
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
source ~/.bashrc
echo $PATH


systemctl status firewalld


firewall-cmd --permanent --add-port=22/tcp
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=8443/tcp
firewall-cmd --new-zone=internal-access --permanent
firewall-cmd --permanent --zone=internal-access --add-source=10.250.0.7
firewall-cmd --permanent --zone=internal-access --add-source=10.250.0.6
firewall-cmd --permanent --zone=internal-access --add-source=10.250.0.8
firewall-cmd --permanent --zone=internal-access --add-port=22/tcp
firewall-cmd --permanent --zone=internal-access --add-port=6443/tcp
firewall-cmd --permanent --zone=internal-access --add-port=10250/tcp
firewall-cmd --permanent --zone=internal-access --add-port=2379/tcp
firewall-cmd --permanent --zone=internal-access --add-port=2380/tcp
firewall-cmd --permanent --zone=internal-access --add-port=5001/tcp
firewall-cmd --permanent --zone=internal-access --add-port=8472/udp
firewall-cmd --permanent --zone=internal-access --add-port=9443/tcp
firewall-cmd --permanent --zone=internal-access --add-port=53/udp
firewall-cmd --permanent --zone=internal-access --add-port=53/tcp
firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16
firewall-cmd --permanent --zone=trusted --add-interface=lo
firewall-cmd --reload
systemctl restart firewalld
systemctl stop firewalld

chmod 600 /etc/ssh/sshd_config

vi /etc/ssh/sshd_config

PasswordAuthentication yes
PermitRootLogin yes
Port 22

systemctl restart sshd
 
###solo en el instana-0

sudo su -
ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): /root/.ssh/instana

cat /root/.ssh/instana.pub 

Copiar contenido de la llave publica en $HOME/.ssh/authorized_keys de los nodos instana-1 e instana-2

vi $HOME/.ssh/authorized_keys  --- Agregar contenido del archivo /root/.ssh/instana.pub (instana-0)

ssh root@10.249.128.37
ssh root@10.249.128.35
 
https://ibm.seismic.com/Link/Content/DCQfMjDgqWT2B87FWXPhGXQbJGDP

AgentKey:
• RlWYfpi2RACQ7b4YEfAubg
SalesId:
• 5NOxPmT9Q26bo6QbzdBGPg
Expires June 30, 2026

sales_key = "WB6cJcexTpiYMSnL9etr8w"
download_key = "lYXhsTslRHacNL7TY_EXDQ"

export DOWNLOAD_KEY=lYXhsTslRHacNL7TY_EXDQ


cat << EOF > /etc/yum.repos.d/Instana-Product.repo
[instana-product]
name=Instana-Product
baseurl=https://_:$DOWNLOAD_KEY@artifact-public.instana.io/artifactory/rel-rpm-public-virtual/
enabled=1
gpgcheck=0
gpgkey=https://_:$DOWNLOAD_KEY@artifact-public.instana.io/artifactory/api/security/keypair/public/repositories/rel-rpm-public-virtual
repo_gpgcheck=1
EOF

yum clean expire-cache -y
yum update -y
#yum --showduplicates list available stanctl
#yum install -y stanctl-1.12.1-1.x86_64
#yum remove stanctl
yum install -y stanctl
yum versionlock add stanctl
#yum versionlock delete stanctl
stanctl --version

Confirmar versión a instalar

stanctl versions identify

stanctl up \
  --instana-version="3.319.465-0" \
  --install-type="production" \
  --multi-node-enable \
  --multi-node-ips="10.249.128.36,10.249.128.37,10.249.128.35" \
  --core-base-domain="lab.mainsoft.com" \
  --core-tls-generate-cert \
  --timeout="60m0s" \
  --download-key="lYXhsTslRHacNL7TY_EXDQ" \
  --sales-key="WB6cJcexTpiYMSnL9etr8w" \
  --unit-agent-key="lYXhsTslRHacNL7TY_EXDQ" \
  --unit-initial-admin-password="Mainsoft.123" \
  --unit-tenant-name="aasa" \
  --unit-unit-name="sandbox" \
  --volume-objects="/mnt/instana/stanctl/objects" \
  --volume-metrics="/mnt/instana/stanctl/metrics" \
  --volume-analytics="/mnt/instana/stanctl/analytics" \
  --volume-data="/mnt/instana/stanctl/data"  

----------------------------------------------------------------
En caso se presente un error durante la instalación, validar si llegan a levantar los pods

kubectl -n instana-core get pods
kubectl -n instana-core get core

kubectl -n instana-unit get pods
kubectl -n instana-unit get unit

kubectl get pods -A
kubectl get nodes

Registrar en archivo hosts
163.75.81.112   lab.mainsoft.com
163.75.81.112   sandbox-aasa.lab.mainsoft.com

Validar ingresando a Instana
https://sandbox-aasa.lab.mainsoft.com/
admin@instana.local
Mainsoft.123


stanctl backend apply --core-feature-flags feature.synthetics.enabled=true
stanctl backend apply --core-feature-flags feature.automation.enabled=true
stanctl backend apply --core-feature-flags feature.logging.enabled=true
----------------------------------------------------------------
vi /etc/systemd/system/k3s.service

TimeoutStartSec=600
TimeoutStopSec=600


stanctl auth reset-idp

stanctl auth reset-password --email=soportemainsoft@mef.gob.pe --password=Mainsoft.123

stanctl backend apply \
  --install-type="production" \
  --core-base-domain="apm.mineco.gob.pe" \
  --core-tls-crt="/mnt/instana/stanctl/apm.mineco.gob.pe.ssl.crt" \
  --core-tls-key="/mnt/instana/stanctl/apm.mineco.gob.pe.pem.key" \
  --download-key="obLIi27lSoKNFIp0iZdh8A" \
  --sales-key="12Uxtb2FT6WkEZG06wVIhQ" \
  --core-use-tu-url-path=false \
  --core-acceptors-agent-port=443 \
  --core-acceptors-agent-host=agent-acceptor.apm.mineco.gob.pe \
  --core-acceptors-otlp-http-port=443 \
  --core-acceptors-otlp-http-host=otlp-http.apm.mineco.gob.pe \
  --core-acceptors-otlp-grpc-port=443 \
  --core-acceptors-otlp-grpc-host=otlp-grpc.apm.mineco.gob.pe \
  --core-acceptors-eum-port=443 \
  --core-acceptors-eum-host=eum.apm.mineco.gob.pe \
  --core-acceptors-synthetics-port=443 \
  --core-acceptors-synthetics-host=synthetics.apm.mineco.gob.pe \
  --core-acceptors-opamp-port=443 \
  --core-acceptors-opamp-host=opamp-acceptor.apm.mineco.gob.pe \
  --core-acceptors-serverless-port=443 \
  --core-acceptors-serverless-host=serverless.apm.mineco.gob.pe
 
stanctl backend apply --core-feature-flags feature.synthetics.enabled=true
stanctl backend apply --core-feature-flags feature.automation.enabled=true
stanctl backend apply --core-feature-flags feature.logging.enabled=true
  
cd $HOME/.stanctl/values/instana-core/

vi custom-values.yaml

------------------------------------------------------------------------------------
core:
  properties:
    - name: config.tag.processor.readiness.min.storage.hit.rate
      value: "0.5"
    - name: config.appdata.shortterm.retention.days
      value: "7"
    - name: config.appdata.longterm.retention.days
      value: "31"
    - name: retention.metrics.rollup5
      value: "86400" # 24 hour (5-min rollup)
    - name: retention.metrics.rollup60
      value: "86400" # 1 day (1-hour rollup)
    - name: retention.metrics.rollup300
      value: "604800" # 7 days (5-min rollup)
    - name: retention.metrics.rollup3600
      value: "2678400" # 31 days (1-hour rollup)
    - name: retention.metrics.rawPayload
      value: "86400" # 24 hour (raw data)
    - name: config.synthetics.retention.days
      value: "7"
------------------------------------------------------------------------------------

stanctl backend apply

stanctl dump config-data

stanctl down

yum versionlock delete stanctl

yum --showduplicates list available stanctl

yum install -y stanctl-1.13.1-1.x86_64

stanctl --version

stanctl versions identify

yum versionlock add stanctl

stanctl up


curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

helm install synthetic-pop \
	--repo "https://agents.instana.io/helm" \
	--namespace popmef \
	--create-namespace \
	--set downloadKey="obLIi27lSoKNFIp0iZdh8A" \
	--set controller.location="POPMEF;POPMEF;Lima;Lima;0;0;POPMEF" \
	--set controller.clusterName="POPMEF" \
	--set controller.instanaKey="obLIi27lSoKNFIp0iZdh8A" \
	--set controller.instanaSyntheticEndpoint="https://synthetics.apm.mineco.gob.pe/synthetics" \
	--set redis.tls.enabled=false \
	--set redis.password="Mainsoft.123" \
	synthetic-pop

```
