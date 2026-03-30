# Kubespray ile Kubernetes Kurulumu

## Ortam Bilgileri

| Node    | IP Adresi      | Rol    |
|---------|----------------|--------|
| master  | 10.102.89.212  | Master |
| worker1 | 10.102.89.179  | Worker |
| worker2 | 10.102.89.15   | Worker |

> **Not:** Test ortamı 3 adet Ubuntu 24.04 makine ile oluşturulmuştur. Her makine 4 GB RAM ve 2 vCPU ile yapılandırılmıştır.

---

## 1. SSH Key Oluşturma ve Dağıtma

Ansible'ın tüm node'lara şifresiz bağlanabilmesi için SSH key tabanlı kimlik doğrulama kuruyoruz.

```bash
ssh-keygen -t ed25519 -C "kubespray-deploy"
```

Key'i tüm node'lara kopyala:

```bash
ssh-copy-id ubuntu@10.102.89.179
ssh-copy-id ubuntu@10.102.89.15
ssh-copy-id ubuntu@10.102.89.212
```

> Master node'a da kopyalamak gerekiyor çünkü Ansible, master'a da SSH ile bağlanır.

### Bağlantı Testi

```bash
ssh -o BatchMode=yes ubuntu@10.102.89.179 echo "OK"
ssh -o BatchMode=yes ubuntu@10.102.89.15 echo "OK"
```

`BatchMode=yes` parametresi SSH'a "şifre sorma, key ile bağlanamazsan hata ver" demektir. Bu sayede interaktif terminal açmadan bağlantıyı test etmiş oluyoruz.

---

## 2. Python ve Bağımlılıkların Kurulumu

```bash
sudo apt update
sudo apt install python3-pip python3.12-venv git
```

- `python3.12-venv` — Ubuntu 24.04'te sistem geneli pip kurulumu engellendiğinden virtual environment zorunludur.
- `git` — Kubespray reposunu klonlamak için gereklidir.

---

## 3. Kubespray Reposunu İndirme

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
```

Kubespray'in tüm Ansible playbook'ları ve konfigürasyon dosyaları bu repodadır. Bundan sonraki tüm komutlar bu dizin içinden çalışacaktır.


---

## 4. Python Virtual Environment Kurulumu

```bash
python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements.txt --timeout 120
```

Ubuntu 24.04, güvenlik gerekçesiyle sistem genelinde `pip` ile paket kurulumunu engeller. Virtual environment, Kubespray'in ihtiyaç duyduğu paketleri (Ansible dahil) izole ortamda kurmamızı sağlar. `--timeout 120` ise büyük paketler indirilirken ağ kaynaklı zaman aşımlarını önler.

<img src="images/requirements.png" width="650">

`requirements.txt` içeriğindeki bağımlılıklar:

| Paket          | Amaç                                      |
|----------------|-------------------------------------------|
| `ansible`      | Playbook'ları çalıştıran otomasyon aracı  |
| `cryptography` | SSL/TLS ve şifreleme işlemleri            |
| `jmespath`     | Jinja2 şablonlarında JSON sorgulama       |
| `netaddr`      | IP adresi ayrıştırma ve işleme            |

---

## 5. Inventory Hazırlama

Örnek inventory'i kopyalayıp kendi cluster'ımız için özelleştiriyoruz:

```bash
cp -rfp inventory/sample inventory/mycluster
```

`inventory/mycluster/inventory.ini` dosyasını aşağıdaki gibi düzenle:

```ini
[all]
master  ansible_host=10.102.89.212 ip=10.102.89.212
worker1 ansible_host=10.102.89.179 ip=10.102.89.179
worker2 ansible_host=10.102.89.15  ip=10.102.89.15

[kube_control_plane]
master

[etcd]
master

[kube_node]
worker1
worker2

[k8s_cluster:children]
kube_control_plane
kube_node
```

Bu dosya Ansible'a hangi makinenin hangi rolü üstleneceğini söyler. `ip=` parametresi Kubernetes'in node'lar arası iletişimde kullanacağı IP adresini belirler. `etcd` grubuna sadece master atanır çünkü test ortamında ayrı bir etcd cluster'ı gereksizdir.

---

## 6. Ansible Bağlantı Testi

Kuruluma başlamadan önce Ansible'ın tüm node'lara erişebildiğini doğruluyoruz:

```bash
ansible all -i inventory/mycluster/inventory.ini -m ping -u ubuntu --become --ask-become-pass
```

- `--become` — Ansible'a komutları `sudo` ile çalıştırmasını söyler. Kubernetes kurulumu root yetkisi gerektirir.
- `--ask-become-pass` — Sudo şifresini interaktif olarak sorar; şifre hiçbir dosyaya yazılmaz.

Tüm node'lardan `pong` yanıtı geliyorsa devam edilir.

---

## 7. Cluster Kurulumu

```bash
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -u ubuntu --become --ask-become-pass
```

`cluster.yml` ana playbook dosyasıdır; aşağıdakileri otomatik olarak kurar ve yapılandırır:

- **containerd** — Container runtime
- **Kubernetes bileşenleri** — kube-apiserver, kubelet, kube-proxy
- **Calico** — Pod ağı (CNI eklentisi)
- **CoreDNS** — Cluster içi DNS çözümlemesi

> Bu işlem ağ hızına bağlı olarak 20–40 dakika sürebilir.

---

## 8. kubeconfig Ayarı

Kubespray, kubeconfig dosyasını varsayılan olarak `root` kullanıcısının home dizinine yazar. `ubuntu` kullanıcısı ile `kubectl` komutlarını çalıştırabilmek için kopyalamamız gerekiyor.

```bash
mkdir -p $HOME/.kube
sudo cp /root/.kube/config $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> `ubuntu` kullanıcısı ile çalışmamızın amacı, yetkileri minimal tutarak güvenlik riskini azaltmaktır.

---

## 9. Helm Kurulumu

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Helm, Kubernetes için paket yöneticisidir. Prometheus, Grafana, Elasticsearch gibi karmaşık uygulamaları tek komutla kurmayı sağlar. Olmadan her servis için onlarca YAML dosyasını manuel yönetmek gerekir.

> Başarılı kurulumda `version.BuildInfo{Version:"v3.20.1"...}` gibi bir çıktı görmelisiniz.

---

## 10. Helm Repository'lerini Ekleme

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add elastic https://helm.elastic.co
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

Helm, chart'ları repository'lerden indirir — tıpkı `apt`'nin paket listesi gibi. `helm repo update` bu listelerin güncel halini çeker. Tüm repo'ları baştan ekliyoruz ki sonraki kurulumlar tek komutla çalışsın.

---

## 11. Namespace Oluşturma

```bash
kubectl create namespace monitoring
kubectl create namespace elastic
kubectl create namespace postgres
kubectl create namespace traefik
```

Namespace'ler Kubernetes'te servisleri mantıksal olarak birbirinden izole eder. Her servis grubunun kendi alanında çalışması yönetimi kolaylaştırır ve yanlışlıkla silme veya kaynak çakışması riskini ortadan kaldırır.

<img src="images/namespaces.png" width="600">

---

## 12. kube-prometheus-stack Kurulumu

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 80.10.0
```

Bu tek komut; **Prometheus**, **Grafana**, **Alertmanager**, **Node Exporter** ve **kube-state-metrics**'i birlikte kurar. `--version` ile sürümü sabitleme alışkanlığı önemlidir — belirtilmezse her kurulumda farklı sürüm gelebilir ve bu production'da beklenmedik değişikliklere yol açar.

<img src="images/monitoring_podlar.png" width="650">

---

## 13. Elasticsearch Kurulumu

```bash
helm install elasticsearch elastic/elasticsearch \
  --namespace elastic \
  --set replicas=1 \
  --set minimumMasterNodes=1 \
  --set resources.requests.memory=512Mi \
  --set resources.limits.memory=1Gi \
  --set resources.requests.cpu=100m \
  --set persistence.enabled=false \
  --set nodeSelector."kubernetes\.io/hostname"=worker2
```

| Parametre | Değer | Neden |
|---|---|---|
| `replicas=1` | 1 | Test ortamı için yeterli; production'da en az 3 olmalı |
| `persistence.enabled=false` | false | Test için uygun; production'da kesinlikle `true` olmalı |
| `nodeSelector` | worker2 | Yük dengesizliğini önlemek için sabit node |
| RAM limiti | 1Gi | Kısıtlı test ortamına göre ayarlandı |

Pod'un ayağa kalkmasını beklemek için:

```bash
kubectl get pods -n elastic -w
```

`1/1 Running` görününce devam edilir. Init container nedeniyle ilk başta `Init:0/1` görünmesi normaldir.

<img src="images/elastich.png" width="600">

---

## 14. PostgreSQL Kurulumu

```bash
helm install postgresql bitnami/postgresql \
  --namespace postgres \
  --set primary.resources.requests.memory=256Mi \
  --set primary.resources.limits.memory=512Mi \
  --set primary.resources.requests.cpu=100m \
  --set primary.persistence.enabled=false \
  --set primary.nodeSelector."kubernetes\.io/hostname"=worker2
```

PostgreSQL'i de `worker2`'ye sabitleme sebebimiz yük dengesizliğidir. Elasticsearch ile birlikte `worker2`'de çalışmalarına rağmen bu iki servis birlikte `worker1`'den daha az kaynak tüketmektedir.

<img src="images/postgres.png" width="600">

---

## 15. Traefik Kurulumu

```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --set ingressRoute.dashboard.enabled=true \
  --set api.insecure=true \
  --set service.type=NodePort \
  --set ports.web.nodePort=30080 \
  --set ports.traefik.expose.default=true \
  --set ports.traefik.nodePort=30900
```

Traefik, servislere dışarıdan erişimi yöneten **Ingress Controller**'dır. `api.insecure=true` dashboard'u şifresiz açar — test ortamı için kabul edilebilir, production'da kapatılmalıdır. `NodePort` ile port-forward olmadan `IP:port` üzerinden doğrudan erişim sağlanır.

<img src="images/traefik.png" width="650">

---

## 16. Servisleri NodePort'a Alma

Servisler varsayılan olarak `ClusterIP` tipindedir ve sadece cluster içinden erişilebilir. NodePort'a geçirerek her servise `master_ip:port` şeklinde dışarıdan erişim açıyoruz:

```bash
# Grafana
kubectl patch svc kube-prometheus-stack-grafana -n monitoring \
  -p '{"spec":{"type":"NodePort","ports":[{"port":80,"targetPort":3000,"nodePort":30300}]}}'

# Alertmanager
kubectl patch svc kube-prometheus-stack-alertmanager -n monitoring \
  -p '{"spec":{"type":"NodePort","ports":[{"port":9093,"targetPort":9093,"nodePort":30903}]}}'

# Elasticsearch
kubectl patch svc elasticsearch-master -n elastic \
  -p '{"spec":{"type":"NodePort","ports":[{"port":9200,"targetPort":9200,"nodePort":30920,"protocol":"TCP"}]}}'
```

Bu yöntemin port-forward'a göre avantajı, terminal kapatılsa bile erişimin sürmesidir.

NodePort'ların aktif olduğunu doğrulamak için:

```bash
kubectl get svc -A | grep NodePort
```

<img src="images/servis.png" width="650">

---

## 17. Metrics Server Kurulumu

`kubectl top nodes` komutu çalışmıyorsa Metrics Server kurulu değildir. Kubespray'in kendi addon mekanizmasıyla aktif ediyoruz — bu yöntem Kubernetes sürümüyle uyumlu versiyonu otomatik seçer:

```bash
cd ~/kubespray
source venv/bin/activate

sed -i 's/metrics_server_enabled: false/metrics_server_enabled: true/' \
  inventory/mycluster/group_vars/k8s_cluster/addons.yml

ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml \
  -u ubuntu --become --ask-become-pass --tags=apps
```

1-2 dakika sonra kontrol:

```bash
kubectl top nodes
```


<img src="images/top.png" width="600">

---

## 18. Servis Erişim Tablosu

Tüm kurulumlar tamamlandıktan sonra servislere tarayıcıdan erişim:

| Servis | URL |
|---|---|
| Grafana | `http://10.102.89.212:30300` | 
| Alertmanager | `http://10.102.89.212:30903` | 
| Elasticsearch | `https://10.102.89.212:30920` | 
| Traefik Dashboard | `http://10.102.89.212:30900/dashboard/` | 

<img src="images/grafadash.png" width="700">

---

## 19. Alertmanager E-posta Bildirimi (Gmail SMTP)

Gmail üzerinden bildirim alabilmek için önce [Google Hesap Güvenliği](https://myaccount.google.com/apppasswords) sayfasından bir **App Password** oluşturun. Uygulama adı olarak `alertmanager` yazıp oluşturulan 16 haneli şifreyi not edin.

Ardından Alertmanager Secret'ını uygulayın:

```bash
kubectl create secret generic alertmanager-kube-prometheus-stack-alertmanager \
  --namespace monitoring \
  --from-literal=alertmanager.yaml='
global:
  smtp_smarthost: smtp.gmail.com:587
  smtp_from: YOUR_EMAIL@gmail.com
  smtp_auth_username: YOUR_EMAIL@gmail.com
  smtp_auth_password: YOUR_APP_PASSWORD
  smtp_require_tls: true
route:
  receiver: gmail
  group_by: [alertname]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  routes:
  - matchers:
    - alertname="Watchdog"
    receiver: "null"
receivers:
- name: gmail
  email_configs:
  - to: YOUR_EMAIL@gmail.com
    send_resolved: true
- name: "null"
' --dry-run=client -o yaml | kubectl apply -f -
```

`--dry-run=client -o yaml | kubectl apply -f -` yöntemini kullanıyoruz çünkü secret zaten varsa üzerine yazar, yoksa oluşturur. Direkt `kubectl create` olsaydı "already exists" hatası verirdi.

Secret uygulandıktan sonra Alertmanager yeni config'i okusun diye restart ediyoruz:

```bash
kubectl rollout restart statefulset alertmanager-kube-prometheus-stack-alertmanager -n monitoring
kubectl get pods -n monitoring | grep alertmanager
```

<img src="images/alert.png" width="650">

---

## 20. Grafana Bilgiler

```bash
kubectl --namespace monitoring get secrets kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

- **Kullanıcı adı:** `admin`
- **Şifre:** Yukarıdaki komutun çıktısı

---

## 21. Elasticsearch Bilgiler

```bash
kubectl get secrets --namespace=elastic elasticsearch-master-credentials \
  -o jsonpath='{.data.password}' | base64 -d ; echo
```

- **Kullanıcı adı:** `elastic`
- **Şifre:** Yukarıdaki komutun çıktısı

