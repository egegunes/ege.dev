2017'de bir etkinlikte eski Kartaca çalışanlarından Bekir Doğan'ı dinleme şansı
bulmuş[^0] ve daha 2005 yılında yönettikleri tüm servisleri OpenVZ[^1] ile
konteynerler halinde kurup dağıttıklarını dinlemiş ve çok etkilenmiştim.
Birileri gerçekten bu tip şeylere uğraşıyor muydu?

Görünüşe bakılırsa, evet. 2000'lerin başından beri sektörün büyük bir kısmı ve
Linux camiası konteynerlerin günümüzdeki hale gelmesi için uğraşıyorlarmış.
Özellikle de Google, yaptığı katkılarla konteynerlerin ana akım haline
gelmesinde öncü olmuş. Aklınıza hemen Kubernetes gelmiş olabilir ama aslında
cgroups[^2] teknolojisinden bahsediyorum.

## Kısaca cgroups

Bir konteyner, tüm sistemi sanallaştırmak yerine bir grup işlemi (process) izole
edip diğer konteynerler ve uygulamalar ile aynı çekirdek üzerinde çalıştırdığı
için tüm sistemin tek sahibi olduğu yanılgısına kolayca düşebilir. Agresif
şekilde kaynak tüketen bir konteyner de kardeş konteynerlerini kararsız hale
getirerek sistemi kararsızlaştırabilir.

Bunun önüne geçmek için sanallaştırma ile bütün bir sistemi konteynere tahsis
edebiliriz ama bu da zamanın büyük kısmında kaynak israfına yol açacaktır.
Mesela, Borg'un dizayn dokümanlarında da kaynaklardan maksimum yararlanma
projenin başlıca amaçlarından biri olarak açıklanıyor[^3].

İşte `cgroups` bu noktada devreye giriyor.

`cgroups`, 2006 yılında Google mühendisleri tarafından geliştirilmeye başlanıp
2008 yılında Linux 2.6.24 sürümüne dahil edilmiş ve yarattığı domino etkisi ile
ekosistemi alt üst etmiş bir özellik.

Kodun Linux'a dahil edilmesi ile sistem yöneticileri, sistemdeki işlemleri
(process/task) gruplayarak ortak kısıtlara tabi tutabilir hale geldi.
İşlemlerin öncelikleri, kaynak limitleri yapılandırılabilir ve muhasebesi
tutulabilir oldu. Dahası, çekirdeğin bu yeteneği LXC ve sonrasında Docker gibi
sistem yönetimini kökten değiştiren yazılımlara yol açtı.

Size küçük bir `cgroups` demosu hazırladım.

[![asciicast](https://asciinema.org/a/372532.svg)](https://asciinema.org/a/372532)

Yukarıda neler oluyor?

1. `fibtest` adında bir kontrol grubu yaratıyorum.
2. Yarattığım grup içinde yine `fibtest` adlı küçük C uygulamasını çalıştırıyorum.
   Bu uygulama sürekli bir şekilde Fibonacci dizisi üretiyor.
3. `systemd-cgtop` ile sistemdeki tüm gruplardaki kaynak tüketimlerini
   izliyorum.
4. Asıl olay burada dönüyor. Grup içinde iki değeri değiştiriyorum:
   `cpu.cfs_period_us` ve `cpu.cfs_quota_us`. `cfs_period_us` ile her CPU
   döneminin kaç mikrosaniye süreceğini (50000) belirlerken `cfs_quota_us` ile
   programın her dönemde maksimum kaç mikrosaniye CPU kullanabileceğini (1000)
   belirliyorum. Uzun lafın kısası programı boğuyorum.
5. Değiştirdiğim değerleri geri alıyorum ve `fibtest` yeniden nefes alıyor.

Ek olarak, demoda kullandığım `cgcreate` ve `cgexec` adlı araçları Ubuntu
18.04'te `cgroup-tools`, Fedora 32'de `libcgroup-tools` adlı paketi kurarak
edinebilirsiz.

## Kubernetes ve cgroups

Varsayılan olarak sistem programları ile konteynerler, makinedeki kaynaklar
üzerinde rekabet halinde çalışıyor. Sistem işlemleri için kaynak ayrılmaması
halinde konteynerlerin kaynak tüketimi, makineyi kararsız hale getirebilir.

Kubernetes de sistem ve kullanıcı işlemleri için kaynak izolasyonunu cgroups
ile sağlıyor. Her makinede eğer yoksa `kubepods` adında bir cgroup yaratılıyor.
Sistem ve Kubernetes'e ait servisler için ise cgroup kendiliğinden
yaratılmıyor. Bunun için sistem yöneticilerinin `kubelet` yapılandırmasını
değiştirmesi gerekiyor[^4].

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
systemReserved:
  cpu: 500m
  memory: 500M
kubeReserved:
  cpu: 500m
  memory: 500M
```

![Sistem ve Kubernetes servisleri için kaynak ayrılması](/images/allocatable.png)

```
$ kubectl describe node
...
Capacity:
  attachable-volumes-gce-pd:  127
  cpu:                        8
  ephemeral-storage:          47259264Ki
  hugepages-2Mi:              0
  memory:                     30879764Ki
  pods:                       110
Allocatable:
  attachable-volumes-gce-pd:  127
  cpu:                        7
  ephemeral-storage:          18858075679
  hugepages-2Mi:              0
  memory:                     29831188Ki
  pods:                       110
...
```

`Capacity` makinede Kubernetes'in gördüğü toplam kaynakları, `Allocatable` ise
kullanıcı podlarına ayrılmış toplam kaynağı gösteriyor.

## Kaynak istekleri ve limitler

Kubernetes'te kaynak yönetiminde iki kavram hemen karşımıza çıkıyor: Kaynak
talebi ve kaynak limiti.

*Kaynak talebi*, podların makinelere yerleştirilmesi için planlayıcı
(scheduler) tarafından dikkate alınıyor ve kullanıcı podlarına ayrılmış
kaynakta yeterli yer olduğu müddetçe pod bir makineye atanıyor. Pod bir
makineye atandıktan sonra `kubelet` de talep edilen kaynağı daima konteynerin
kullanabileceğini garanti ediyor. "Pending" durumu ise podun atanmayı beklediği
anlamına geliyor. Her podun yaşam döngüsünde "Pending" durumunda bulunmak var
ama podların bu durumda uzun süre geçiriyor olması halinde kaynak taleplerini
gözden geçirmekte fayda var. Bir podun kaynak talebi içindeki konteynerlerin
taleplerinin toplamından oluşuyor.

Kullanıcı podları için ayrılan toplam kaynaktan verimli şekilde yararlanmak
adına her konteyner için kaynak isteği tanımlamak önem taşıyor. Planlayıcı,
podları makinelere atarken makinedeki kullanılabilecek kapasitenin makineye
atanmış podların toplam kaynak talebinden daha yüksek olduğunu kontrol ediyor.
Pratikte, makinedeki podlar taleplerinden daha az kaynak tüketiyor olsalar bile
taleplerinin toplamı kullanılabilir kaynağa eşitse bu makineye yeni pod
atanamayacaktır. Zira, yukarıda bahsettiğim gibi konteynerin talep ettiği
kaynak `kubelet` tarafından her zaman garanti ediliyor.

*Kaynak limiti*, bir podun sistemdeki tüm kaynağı tüketmesini engellemek için
`kubelet` tarafından dikkate alınıyor.

Bir konteyner talep ettiğinden daha fazla kaynak tüketebilir. Ancak, talebinden
daha fazla hafıza (memory) tüketen bir konteyner, makinede hafızanın azalması
halinde tahliye (evict) edilecektir.

Limitinden daha fazla kaynak tüketen bir konteynere ne olacağı ise ilgili
kaynağa bağlı:

1. Hafıza limitinin aşılması durumunda konteyner sonlandırılabilir ve mümkünse
   yeniden başlatılablir.
2. CPU limitinin aşılması durumunda ise konteyner sonlandırılmaz, sadece CPU
   kullanımı daraltılır.

Pratikte, makinedeki tüm konteynerlerin toplam kaynak talebi kullanıcı
podlarına ayrılan kaynağı aşamaz ama kaynak limitlerinin toplamı
kullanılabilecek kaynağın çok üstünde olabilir. Uçak şirketlerinin nasıl olsa
birkaç fire çıkar diyerek fazladan bilet satması gibi toplam kaynak limiti için
de maksimum kaynağın üstüne çıkılabilir. Bu durumda yukarıda anlattığım şekilde
sistem ve Kubernetes işlemlerine kaynak ayırmak daha fazla önem taşıyacaktır.

## Namespace seviyesinde kaynak yönetimi

Servislerinizi veya ortamlarınızı (test, qa gibi) ayırmak için Kubernetes namespace'lerini
kullanıyorsanız, her namespace için ön tanımlı kaynak talebi ve limitleri
belirleyebilirsiniz. Böylece her konteyner için ayrı ayrı kaynak yapılandırması
yapmadan ön tanımlı değerleri kullanabiliyoruz.

Ön tanımlı kaynak yapılandırması için bir `LimitRange`[^5] tanımlamak gerekiyor:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: qa-limitrange
spec:
  limits:
  - default:
      cpu: 1
      memory: 512Mi
    defaultRequest:
      cpu: 500m
      memory: 256Mi
    type: Container
```

```
$ kubectl describe limits
Name:       qa-limitrange
Namespace:  qa
Type        Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---  ---------------  -------------  -----------------------
Container   cpu       -    -    500m             1              -
Container   memory    -    -    256Mi            512Mi          -
```

Gün içinde küçük ağlama krizleri yaşayacak kadar YAML dosyası ile uğraştığımız
için her dosyaya dört satır eksik yazmak çok çekici geliyor. Ancak,
`LimitRange` bundan daha fazlasını da yapıyor. Özellikle çok kiracılı (multi
tenancy) Kubernetes cluster'larında kaynak talep ve limitlerinin onaylanması
için de `LimitRange` tanımlayabiliyoruz.

```
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limitrange
spec:
  limits:
  - max:
      cpu: 2
      memory: 1Gi
    min:
      cpu: 1
      memory: 500Mi
    type: Container
```

```
$ kubectl describe limits
Name:       dev-limitrange
Namespace:  dev
Type        Resource  Min    Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---    ---  ---------------  -------------  -----------------------
Container   cpu       500m   2    2                2              -
Container   memory    256Mi  1Gi  1Gi              1Gi            -
```

Namespace için `LimitRange` tanımladığımızda aslında bir kabul denetçisi[^6]
(admission controller) oluşturmuş oluyoruz. Bu sayede cluster'a eklenmek
istenen her pod, kaynak yapılandırması bakımından onaylanarak içeri alınıyor.
`LimitRange` kurallarına uymayanlar ise reddediliyor. Bu sayede sistem
yöneticisinden bağımsız üçüncü kişilerin diğer servisleri kararsız hale
getirmeyecek şekilde yeni podlar kurabilmesinin önü açılmış oluyor.

Namespace üzerindeki kontrolümüz bu kadarla da kalmıyor. Bir
`ResourceQuota`[^7] tanımlayarak namespace'in toplam kaynağını da
sınırlayabiliyoruz.

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: qa-resourcequota
  namespace: qa
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 8Gi
    limits.cpu: "2"
    limits.memory: 8Gi
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: test-resourcequota
  namespace: test
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 4Gi
    limits.cpu: "1"
    limits.memory: 4Gi
```

![ResourceQuota](/images/resourcequota.png)

Yukarıdaki yapılandırma ile aşağıdaki hususları garanti etmiş oluyoruz:

1. Namespace'teki podların toplam hafıza talebi ve limiti `qa` için 8 GB,
   `test` için 4 GB'ı geçemez.
2. Namespace'teki podların toplam CPU talebi ve limiti `qa` için 2, `test` için
   1 CPU'dan fazla olamaz.

## QoS Sınıfları

Podlara ait QoS (Quality of Service) sınıfları hem planlama (scheduling) hem de tahliye
(eviction) için önem taşıyor. Kullanabileceğimiz üç QoS sınıfı mevcut:

1. **Guaranteed:** Pod içindeki tüm konteynerlerin kaynak talep ve limitleri
   eşit ise
2. **Burstable:** Pod, Guaranteed sınıfına girmiyor ve en az bir konteyner
   kaynak talebinde bulunuyorsa
3. **BestEffort:** Pod içindeki hiçbir konteyner herhangi bir kaynak talebi
   veya limitine sahip değilse

Anlayacağınız üzere bu sınıflar podların kaynak yapılandırmasına göre Kubernetes
tarafından atanıyor.

```
$ kubectl get pod <pod> -o yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  containers:
  - name: container-1
    image: ...
    resources:
      limits:
        memory: 1Gi
      requests:
        cpu: 100m
        memory: 512Mi
status:
  ...
  phase: Running
  qosClass: Burstable
```

## Tahliye

Yukarıda anlattığım tüm yapılandırmalara rağmen makinelerimizdeki kaynaklar
tükenebilir ve cluster kararsızlaşabilir. Bu durumda `kubelet` hızlıca kaynak
geri kazanmaya çalışacaktır[^8]. Çabalarının nafile kalması durumunda ise
`tahliye` süreci başlar.

Tahliye için `kubelet` podları bir sıraya koyar:

* BestEffort veya Burstable QoS sınıfına ait podlardan **kaynak talebinden
  fazlasını kullananlar** önceliklerine ve taleplerinden ne kadar fazla kaynak
  tükettiklerine göre sıralanır ve tahliye edilir.
* Guaranteed podlar ile taleplerinden daha az kaynak tüketen Burstable podlar
  en son tahliye edilir. Guaranteed podlara, adı üstünde, başka bir podun kaynak
  tüketiminden dolayı tahliyeye maruz kalmayacağı garanti ediliyor. Ancak,
  sistem veya Kubernetes'e ayırdığımız kaynakların tükenmeye başlaması durumunda
  en düşük önceliğe sahip poddan başlayarak bunlar da tahliye edilebilir.

Öncelik, yine Kubernetes yöneticileri olarak `PriorityClass` ile
ayarlayabileceğimiz bir değer. Yazıyı daha da dallandırmamak adına sizi ilgili
dokümanlara[^9] yönlendirmekle yetiniyorum.

[^0]: https://www.youtube.com/watch?v=ss6p7pjnd1U
[^1]: https://openvz.org/
[^2]: https://en.wikipedia.org/wiki/Cgroups
[^3]: Large-scale cluster management at Google with Borg, https://research.google/pubs/pub43438/
[^4]: https://kubernetes.io/docs/tasks/administer-cluster/reconfigure-kubelet/
[^5]: https://kubernetes.io/docs/concepts/policy/limit-range/
[^6]: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/
[^7]: https://kubernetes.io/docs/concepts/policy/resource-quotas/
[^8]: https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#reclaiming-node-level-resources
[^9]: https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/
