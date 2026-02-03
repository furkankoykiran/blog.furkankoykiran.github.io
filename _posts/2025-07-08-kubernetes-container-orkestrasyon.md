---
layout: post
title: "Kubernetes Production Survival Guide: YAML Cehenneminden Çıkış"
date: 2025-07-08 15:20:00 +0300
categories: [DevOps, System Design]
description: "QoS sınıfları, Liveness/Readiness farkı, CrashLoopBackOff debug rehberi ve Helm anti-pattern'leri üzerine kıdemli mühendis notları."
image: assets/img/posts/kubernetes-architecture-diagram.png
---

Kubernetes (K8s) öğrenme eğrisi bir dağ gibidir. Zirveye çıktığınızda manzara harikadır (otomatik ölçeklenen, self-healing sistemler), ancak tırmanırken oksijensiz kalıp `CrashLoopBackOff` çukuruna düşmeniz çok olasıdır.

Bu yazıda, tutoriallarda anlatılmayan ama production'da gece 3'te sizi uyandıran konulara odaklanacağız.

## 1. Kaynak Yönetimi ve QoS Sınıfları

"Podlarım rastgele ölüyor (OOMKilled)" diyorsanız, muhtemelen **Quality of Service (QoS)** sınıflarını bilmiyorsunuzdur. K8s, podlara 3 sınıf atar:

1.  **Guaranteed:** `requests` ve `limits` değerleri birbirine eşittir.
    *   *Senaryo:* Kritik veritabanı veya ödeme servisi. Node sıkışırsa en son bunlar öldürülür.
2.  **Burstable:** `limits` > `requests` durumudur.
    *   *Senaryo:* Web sunucuları. Ani trafik gelince limitine kadar çıkar, sonra düşer.
3.  **BestEffort:** Hiçbir kaynak tanımlanmamıştır.
    *   *Senaryo:* Node sıkışırsa **ilk ölecek** olanlardır. Production'da asla kullanmayın.

```yaml
resources:
  requests:
    memory: "64Mi" # K8s bu kadar RAM'i GARANTİ eder
    cpu: "250m"
  limits:
    memory: "128Mi" # Pod bu değeri geçerse OOMKilled olur
    cpu: "500m"     # Pod bu değeri geçerse Throttling yer
```

## 2. Probes: Liveness vs Readiness

Bu ikisini karıştırmak, zincirleme kazaya sebep olur.

*   **Liveness Probe:** "Uygulama çöktü mü/kilitlendi mi?" diye sorar. Cevap hayırsa podu **RESTART** eder.
*   **Readiness Probe:** "Trafik almaya hazır mısın?" diye sorar. Cevap hayırsa podu **Service Endpoints'ten çıkarır** (restart etmez).

**Production Hatası:**
Liveness probe'u veritabanına bağlamayın! Veritabanı yavaşlarsa, tüm web podlarınız "Veritabanına bağlanamıyorum, o zaman ben hastayım" der. Liveness restart atar. Yeni açılan pod da bağlanamaz, o da ölür. Tüm sistem `CrashLoop`'a girer. Liveness sadece ve sadece "web server süreci (PID) çalışıyor mu"ya bakmalıdır.

## 3. CrashLoopBackOff Debug Rehberi

Yeni deployment yaptınız ve podlar sürekli ölüyor. Ne yapacaksınız?

1.  `kubectl get pods` -> Restart count'a bakın.
2.  `kubectl logs <pod-name>` -> Genelde "App started" mesajını görürsünüz çünkü pod restart yemiştir.
3.  `kubectl logs <pod-name> --previous` -> **Altın Komut.** Ölen (önceki) container'ın son çığlığını (stack trace) gösterir.
4.  `kubectl describe pod <pod-name>` -> "OOMKilled", "MountFailed" veya "ImagePullBackOff" gibi altyapısal hataları burada görürsünüz.

## 4. Networking: ClusterIP, NodePort, Ingress

Mantığı basit kurun:
*   **ClusterIP:** Sadece cluster içi (Backend <-> DB).
*   **NodePort:** Node'un portunu dışarı açar (Genelde test için veya LoadBalancer yoksa).
*   **Ingress:** HTTP/HTTPS trafiğini yöneten router. "domain.com/bireysel" trafiğini X servisine, "/kurumsal"ı Y servisine yönlendirir. SSL Termination burada yapılır.

Ingress Controller olarak Nginx kullanıyorsanız, `client_max_body_size` gibi ayarları Annotation ile verebilirsiniz:
```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "50m"
```

## 5. Helm: Dost mu Düşman mı?

Helm ("K8s için apt-get") harikadır ama **Over-Templating** bir anti-pattern'dir. `values.yaml` dosyanız 500 satırsa ve her şey `if/else` ile şarta bağlandıysa, o chart'ı kimse yönetemez.

**Tavsiye:** ArgoCD veya FluxCD kullanarak GitOps yapısına geçin. `kubectl apply` komutunu CI/CD pipeline'ından değil, Git reposundaki değişiklikten tetikleyin. Bu sayede "Kim neyi değiştirdi?" sorusu tarih olur.

## 6. Secret Yönetimi

ConfigMap'ler şifresizdir. Secret'lar ise sadece Base64 ile encode edilmiştir (yani şifresizdir!).
Secret'larınızı Git'e nasılatacaksınız?

1.  **SealedSecrets (Bitnami):** Public key ile şifrelersiniz, Git'e atarsınız. Cluster içindeki controller private key ile açar.
2.  **External Secrets Operator:** AWS Secrets Manager veya HashiCorp Vault'taki veriyi çeker, K8s Secret nesnesine dönüştürür. En güvenli yöntemdir.

## 7. Production Checklist

*   [ ] **Resource Limitleri Var mı?** Limitsiz pod, saatli bombadır.
*   [ ] **HPA Tanımlı mı?** Trafik artınca pod sayısı artıyor mu? (Horizontal Pod Autoscaler).
*   [ ] **Affinity Kuralları:** Tüm podların aynı node'a doluşmasını engellediniz mi? (`podAntiAffinity`).
*   [ ] **PDB (Pod Disruption Budget):** Node bakımı sırasında "En az 2 pod ayakta kalsın" diyor musunuz?


## 8. StatefulSet vs Deployment: Kimdir Bu "Pet"ler?

Yeni başlayanların kafası "StatefulSet"te karışır.
*   **Deployment:** "Cattle" (Sığır) mantığıdır. Podlar isimsizdir (web-5f9b-zkq1), herhangi biri silinse de yerine yenisi gelir, kimse üzülmez. Stateless uygulamalar (API, Web) içindir.
*   **StatefulSet:** "Pet" (Evcil hayvan) mantığıdır. Podların ismi sabittir (db-0, db-1). `db-0` ölürse, yerine yine `db-0` gelir. Sıralı açılır, sıralı kapanır. Veritabanları (PostgreSQL, Kafka, Redis) için zorunludur.

Eğer veritabanını Deployment ile kurmaya çalışıyorsanız, veri kaybı yakındır.

## 9. Depolama: PV, PVC ve StorageClass

Docker'daki `-v /host:/container` mantığını unutun. K8s'te depolama soyuttur.

1.  **StorageClass (SC):** "Bana AWS gp3 ver" veya "NFS ver" diyen sınıftır.
2.  **PersistentVolumeClaim (PVC):** "Bana 10GB yer lazım" diyen haktır.
3.  **PersistentVolume (PV):** Fiziksel diskin kendisidir.

**Kritik Kavram: Access Modes**
*   `ReadWriteOnce` (RWO): Diski (EBS gibi) sadece TEK bir node mount edebilir.
*   `ReadWriteMany` (RWX): Diski (NFS/EFS gibi) aynı anda ÇOK node mount edebilir.

API podlarınızın hepsi aynı diske yazacaksa RWX (NFS) gerekir. Block Storage (EBS) RWO'dur, yani sadece 1 pod'a takılabilir. "Neden 2. podum diski göremiyor?" sorusunun cevabı budur.

## 10. Güvenlik Duvarları: Network Policies

K8s varsayılan olarak "Flat Network"tür. Her pod, her pod ile konuşabilir. Bu, güvenlik için felakettir.
Eğer bir saldırgan Web Pod'unuzu ele geçirirse, oradan Veritabanı Pod'una sınırsız erişim sağlar.

Bunu engellemek için **NetworkPolicy** kullanılır.
"DB Pod'una sadece Backend Pod'undan gelen trafiği kabul et, gerisini reddet (Deny-All)" demelisiniz.

```yaml
kind: NetworkPolicy
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
```

## 11. Scheduling: Taints ve Tolerations

Her node eşit değildir. Bazı node'lar "GPU"ludur, bazıları "Spot Instance"dır.
Kritik ödeme servisinizi, her an kapanabilecek Spot Instance'da çalıştırmak istemezsiniz.

*   **Taint:** Node'a "Bana leke (taint) sürdüler, kimse gelmesin" der.
*   **Toleration:** Pod'a "Ben o lekeyi tolere ederim, oraya konabilirim" yetkisi verir.

Örneğin, Master node'larında varsayılan olarak `NoSchedule` taint'i vardır. Bu yüzden sizin podlarınız Master'a konmaz.

## 12. Service Mesh: Gerekli mi?

İnternet "Istio kurun" diyen makalelerle dolu.
Cevap: %95 ihtimalle **HAYIR**.

Service Mesh (Istio, Linkerd), 50+ mikroservisiniz yoksa size sadece karmaşıklık, CPU maliyeti ve latency (gecikme) katar. K8s'in kendi Service/Ingress yapısı çoğu senaryo için yeterlidir. "Resume Driven Development" yapmayın.


## 13. Yetkilendirme: RBAC (Role Based Access Control)

K8s'e herkes "admin" olarak girmemeli. İki kavramı birbirinden ayırın:
*   **Role:** Sadece belirli bir namespace (örn: development) içinde yetki verir.
*   **ClusterRole:** Tüm cluster'da (örn: node'ları listeleme) yetki verir.

Sık yapılan hata: Bir geliştiriciye yanlışlıkla `ClusterAdmin` yetkisi vermek. Bunun yerine, sadece kendi namespace'inde podları restart edebileceği bir `Role` tanımlayın ve `RoleBinding` ile kullanıcıya bağlayın.

## 14. Operasyonel Hazırlık: Drain ve Cordon

Bir node'a bakım yapacaksanız (Kernel update vs.), sakın çat diye kapatmayın. K8s'in kibar yolları vardır:

1.  `kubectl cordon <node-name>`: "Bu node'a yeni pod atama" (Karantina).
2.  `kubectl drain <node-name> --ignore-daemonsets`: "Bu node'daki mevcut podları nazikçe diğer node'lara tahliye et".

Bu işlem bitince node boşalır, bakımı yaparsınız ve `uncordon` ile tekrar trafiğe açarsınız. Production'da "Graceful Shutdown" budur.

## 15. Metrics Server ve HPA

"Podlar otomatik ölçeklensin" (HPA) demek için `metrics-server`'ın kurulu olması şarttır. HPA, CPU/RAM kullanımını bu servisten çeker.

Eğer `kubectl top pods` komutu hata veriyorsa, HPA çalışmaz. Mutlaka kontrol edin:
```bash
kubectl top nodes
kubectl top pods -n kube-system
```

## Özetle

Kubernetes karmaşıktır çünkü çok güçlüdür. Onu "Docker'ın biraz daha büyüğü" olarak görmeyin. Kendi networking, storage ve yetkilendirme katmanları olan bir işletim sistemi gibidir.

İlk kural: "Simple is better". İhtiyacınız olmayan (Service Mesh gibi) katmanları eklemeyin. Stabilite, özellikten önce gelir.


