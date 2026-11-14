# OpenNCCM ZyXEL Port Mapper

ZyXEL anahtarlarda fiziksel kablo takibi yapmadan LLDP komutlarıyla port ve cihaz tespiti yapmayı amaçlayan saha rehberi.

## 💻 Kullanılan Temel Komutlar ve Amaçları

* **`show lldp info remote`**: Komşu cihazların MAC adresini, cihaz adını ve karşı portunu görüntüler.
* **`show lldp config`**: Protokolün aktif olup olmadığını kontrol eder.
* **`show interfaces status` & `show power inline`**: LLDP desteği olmayan cihazların durumunu ve PoE beslemesini inceler.

## 🔍 Saha Analizi

Çıktılardaki MAC adresi ön ekleri (OUI), port açıklamaları ve karşı port tipleri (örneğin Access Point için `eth0` veya anahtar uplinkleri için sayısal portlar) analiz edilerek ağ topolojisi çıkarılır.

## 🤝 Katkıda Bulunma
Katkı sağlamak için **Issue** veya **Pull Request** açabilirsiniz.
