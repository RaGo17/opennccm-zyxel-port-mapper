# OpenNCCM + ZyXEL LLDP Port Haritalama Rehberi

Bu proje, **OpenNCCM** sisteminin **ZyXEL** yönetilebilir anahtarlara (switch) sadece konfigürasyon yedeği (`running-config`) almak için değil, aynı zamanda **LLDP komşuluk ve port bilgilerini** de çekip işleyebilmesi için yapılan tüm yapılandırma ve saha adımlarını kapsar.

## 🎯 Projenin Amacı

Standart yapılandırma yedekleme araçları yalnızca cihaz ayarlarını yedekler; ancak hangi fiziksel portun ucunda hangi Access Point veya Uplink switch'in bağlı olduğunu göstermez. Bu rehber ile **OpenNCCM** tarafındaki entegrasyon ayarlarını ve **ZyXEL** CLI komutlarını birleştirerek port haritasının otomatik olarak toplanmasını ve analiz edilmesini sağlıyoruz.

---

## 🖥️ BÖLÜM 1: OpenNCCM Tarafında Yapılan Ayarlar

OpenNCCM'in ZyXEL switch'lerden port ve komşuluk verilerini sorunsuz çekebilmesi için yapılan 4 temel ayar:

### 1. Sunucu İşletim Sistemi (SSH İstemci) Ayarı
Eski ve orta segment ZyXEL switch'lerin kullandığı SSH şifreleme algoritmalarını OpenNCCM sunucusunun kabul etmesi için `/etc/ssh/ssh_config` (veya `~/.ssh/config`) dosyasına şu satırlar eklenir:

```text
Host *
    KexAlgorithms +diffie-hellman-group1-sha1,diffie-hellman-group14-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers +aes128-cbc,3des-cbc
```
* **Amacı:** OpenNCCM worker servisinin ZyXEL cihazlara bağlanırken `no matching key exchange method` hatası alıp düşmesini engeller.

### 2. OpenNCCM Cihaz Tipi (Device Type / Template) Tanımı
OpenNCCM web arayüzünden (**Admin > Device Types**) ZyXEL cihazların CLI yapısını sisteme tanıtmak için oluşturulan şablon ayarlarıdır:

* **Device Type Adı:** `ZyXEL`
* **Prompt (Komut İstemi) Regex:** `[>#]\s*$` *(Komut çıktısının bittiğini anlamasını sağlar)*
* **Sayfalama Kapatma (Disable Paging) Komutu:** `terminal length 0` *(Uzun çıktılarda `--More--` yazısında takılmaması için oturum açılır açılmaz gönderilir)*
* **Ana Yedekleme (Backup) Komutu:** `show running-config`

### 3. Özel Komut / LLDP Veri Toplama Tanımı
OpenNCCM'in standart konfigürasyon yedekleme döngüsüne **port haritası bilgisini** dahil etmek için cihaz komut listesine şu komut eklenir:

* **Eklenen Komut:** `show lldp info remote`
* **Opsiyonel Ek Komut:** LLDP desteklemeyen cihazların (kamera, yazıcı vb.) port takibi için `show mac address-table` komutu da aynı listeye eklenir.
* **Çalışma Mantığı:** OpenNCCM zamanlanmış görev (cron/worker) ile cihaza bağlandığında `show running-config` komutunun hemen ardından bu komutları çalıştırır ve dönen çıktıyı yedek/envanter dizinine kaydeder.

### 4. Çıktı Ayrıştırma (Parser) Mantığı
Çekilen ham LLDP metni şu kurallarla anlamlı port bilgisine dönüştürülür:
* **Başlık Temizleme:** Çıktıdaki `LLDP Remote Device Information:`, `LocalPort...` satırları ve boş satırlar yok sayılır.
* **Sütun Eşleştirme:** 
  * `LocalPort` -> Switch üzerindeki fiziksel port numarası
  * `ChassisId` -> Bağlı cihazın MAC adresi (İlk 6 haneden marka/OUI tespiti yapılır)
  * `PortId` -> Karşı cihazın portu (`eth0` ise Access Point, sayısal değer ise Uplink Switch)
  * `SystemName` -> Bağlı cihazın ağdaki adı

---

## ⚙️ BÖLÜM 2: ZyXEL Switch Tarafında Yapılan Ayarlar

OpenNCCM'in switch'e bağlandığında komşuları görebilmesi için cihaz üzerinde şu ayarlar aktif edilir:

```text
ZyXEL-Switch# configure terminal
ZyXEL-Switch(config)# lldp run
ZyXEL-Switch(config)# exit
ZyXEL-Switch# write memory
```

---

## 💻 BÖLÜM 3: Sahada Manuel Kontrol ve Hızlı Filtreleme Komutları

Otomatik döngüyü beklemeden sunucu üzerinden veya doğrudan switch CLI'dan anlık kontrol gerektiğinde kullanılan komutlar:

### ZyXEL Switch İçi Filtreleme (Pipe `|`)
```text
# Sadece belirli bir oda/cihaz adını LLDP tablosunda bulma
ZyXEL-Switch# show lldp info remote | include Depo

# LLDP konuşmayan cihazı (Kamera/Yazıcı) MAC adresinin son 4 hanesiyle bulma
ZyXEL-Switch# show mac address-table | include 67:48

# Sadece aktif (Up) olan portları listeleme
ZyXEL-Switch# show interfaces status | include Up
```

### OpenNCCM Sunucusu Üzerinden `grep` ve `awk` ile Hızlı Analiz
```bash
# Çekilen LLDP yedek dosyaları içinde spesifik bir AP'nin hangi switch'in hangi portunda olduğunu tüm ağda arama
grep -rn "B-1Depo-WiFi" /opt/opennccm/backups/

# Sadece Ubiquiti cihazlarını MAC ön ekinden (OUI) filtreleme
grep -E "78:8a:20|fc:ec:da|80:2a:a8" /opt/opennccm/backups/BEksiBirMutfak_lldp.txt

# Çıktıyı temizleyip sadece Port -> Cihaz şeklinde ekrana basma
grep -E "^[[:space:]]*[0-9]+" BEksiBirMutfak_lldp.txt | awk '{print "Port " \(1 " ---> " \)NF}'
```

---

## 🛠️ Sorun Giderme (Hata Olursa Nereye Bakılır?)

1. **SSH Hatası mı?** Sunucudan manuel `ssh admin@<switch_ip>` komutu dene. Algoritma hatası veriyorsa **Bölüm 1 - Adım 1**'deki `ssh_config` dosyasını kontrol et.
2. **Boş Çıktı mı Geliyor?** Switch'e girip `show lldp config` yaz. LLDP kapalıysa `lldp run` komutunu uygula.
3. **Veri Yarım mı Kalıyor?** `terminal length 0` komutu o ZyXEL modelinde farklı olabilir; switch üzerinde sayfalama kapatma komutunu doğrula.
4. **Sütunlar mı Kayıyor?** Komşu cihazın `SystemName` isminde boşluk karakteri varsa `awk '{print $5}'` yerine son sütunu alan `awk '{print $NF}'` kullan.
