# Çalışma Günlüğü

**Proje:** Çok Modlu Ürün Kategori Sınıflandırma
**Öğrenci:** Nurgül Sarıtaş

Bu günlükte ödev boyunca her gün ne yaptığımı, nerede takıldığımı ve sorunları nasıl çözdüğümü yazdım. Süreler yaklaşıktır. Yapay zekâ kullanımını en sonda ayrı bir bölümde açıkladım.

---

## 22.09 · 0,5 saat · Ödevi okudum

Ödev metnini okudum ve ödevin içeriğini, hedefini detaylı inceledim. Hedefin `articleType` sütununu yalnızca ürün adı ve ürün görseliyle tahmin etmek olduğunu, diğer sütunların kullanılamayacağını not aldım.

## 23.09 · 2 saat · Plan, kurulum, veriyi ilk açış

- Görevleri sıraya koyup kendime bir plan çıkardım. Colab'da Google Drive bağlantısını kurdum.
- Veriyi `kagglehub` ile indirdim. `images` klasöründe 44.441 görsel saydım, `styles.csv` dosyasında ise 44.446 kayıt vardı. Aradaki 5 kayıtlık farkı sonra kontrol etmek üzere not ettim.
- Yasaklı sütunların neden sızıntı yarattığını gösteren küçük bir bölüm ekledim: `subCategory` değeri "Watches" olan 2.542 ürünün hepsi Watches sınıfında ve birçok alt kategori yalnızca tek bir sınıf içeriyor. Bu sütunlar cevabı doğrudan verdiği için modelde kullanmayacağım.

**Sorunlar:**
- `!ls {myntradataset}` komutu "No such file or directory" hatası verdi. Süslü parantezin içine yalnızca Python değişkeni yazılabildiğini öğrendim, yolu `{veri_yolu}/myntradataset` olarak düzelttim.
- `pd.read_csv` "Expected 10 fields in line 6044, saw 11" hatası verdi. Bunu ertesi güne bıraktım.

## 24.09 · 1 saat · Bozuk satırı inceledim

- `sed -n '6044p'` ile hatalı satıra baktım. Ürün adında tırnak içine alınmamış bir virgül var: "Boss Men Perfume, After Shave Balm and Shower Gel Set". Pandas bu virgülü yeni sütun sandığı için satır 11 parçaya bölünüyor.
- Bu satırları silmek yerine **onarmaya** karar verdim, çünkü veri kaybı olmasını istemedim. `csv` modülüyle onarım koduna başladım.


## 26.09 · 2,5 saat · Onarım ve eksik veri kontrolleri

- `csv.reader` ile dosyayı satır satır okudum. 10'dan fazla parçaya bölünen satırlarda ilk 9 parçayı bırakıp geri kalanını virgülle birleştirdim (`satir[:9] + [",".join(satir[9:])]`). **22 satır onarıldı**, 44.446 kaydın hepsi korundu.
- Birleştirirken `", "` yerine `","` kullandım. `csv.reader` virgülden sonraki boşluğu parçanın içinde bıraktığı için orijinal ad birebir geri geliyor. 
- Boş ad, yalnızca boşluktan oluşan ad ve tekrar eden id kontrolü yaptım. Hepsi 0 çıktı.
- İlk gün pandas'ta 7 boş ad görmüştüm ama bu kontroller 0 buldu. Kısa adlara bakınca bu 7 kaydın adında düz metin olarak **"NA"** yazdığını gördüm. Pandas bu metni otomatik olarak NaN'a çeviriyormuş, `csv.reader` ise olduğu gibi bırakıyor.

**Sorun:** "NA" yazan adlar boş sayılmadığı için `isna()` ve `== ""` kontrolleri bunları yakalamadı. `pd.read_csv` bu metni kendiliğinden NaN'a çevirirdi ama hatalı satırlarda ya duruyor ya da (`on_bad_lines="skip"` ile) 22 ürünü siliyordu. Veri kaybı olmasın diye `csv.reader`'ı seçtim, bunun karşılığında da "NA" adları kendim bulmam gerekti. Bu kayıtları ertesi gün NaN'a çevirip çıkartmaya karar verdim.

## 27.09 · 3 saat · Temizlik

- "NA" adları `replace` ile NaN yaptım. Önce küçük bir deneme yapıp `replace` fonksiyonunun yalnızca tam eşleşmeyi değiştirdiğini kontrol ettim ("NAVY Blue Shirt" bozulmuyor). Tam eşleşme yaptığından emin oldum.
- Görsel yolunu `f"{veri_yolu}/images/{i}.jpg"` olarak düzeltince görseli olmayan **5 kayıt** buldum. Böylece 23.09'daki 5 kayıtlık farkın nedeni de açıklanmış oldu. Bunların 4'ü aynı markaya ait ve id'leri ardışık.
- Adı geçersiz ve görseli olmayan kayıtları çıkardım. Model iki girdiyi de kullandığı için birinin eksik olduğu kayıt işe yaramıyor. Sonra en sık 20 sınıfı seçtim ve yalnızca `id`, `productDisplayName`, `articleType` sütunlarını bıraktım.
- Temizliği en sık 20 sınıfı seçmeden **önce** yaptım. Sınıf sıralaması kullanılabilir kayıtlar üzerinden yapılmalıdır.

**Sorun: Tablom bozuldu.** Görseli olmayan kayıtları çıkarmak için `df = df["gorsel_var"]`, en sık 20 sınıfı seçmek için de `df = df["articleType"].isin(top20)` yazdım. Sonraki hücreler `KeyError` verdi. `type(df)` ile bakınca `df`'in artık bir tablo (DataFrame) değil, tek bir sütun (Series) olduğunu gördüm.
- **Neden:** `df["sütun"]` yalnızca o sütunu verir. Satırları süzmek için koşulu tablonun içine yazmak gerekiyormuş: `df = df[df["gorsel_var"]]`. Ben sütunu `df`'in üstüne yazdığım için bütün tablo kaybolmuştu.
- **Durumu daha da karıştıran şey:** Hatalı hücreyi silmek işe yaramadı çünkü çalıştırılmış bir hücrenin etkisi Colab'ın hafızasında kalıyor. Colab'ın hücreleri ekrandaki sıraya göre değil, benim tıkladığım sıraya göre çalıştırdığını da burada öğrendim. "Öncekileri çalıştır" deyince bu sefer `gorsel_var` sütunu bulunamadı çünkü görsel kontrol hücresi notebook'ta temizlik hücrelerinin altında kalmıştı.

**Çözüm:** Veri okuma, satır onarımı, "NA" dönüşümü, görsel kontrolü ve temizlik adımlarını **tek bir hücrede** topladım. Bu hücre her çalıştığında veriyi dosyadan baştan okuyor. Böylece hangi hücreyi hangi sırayla çalıştırdığımın bir önemi kalmadı ve sonuç her seferinde aynı çıkıyor. Sonraki günlerde bulduğum yeni temizlik adımlarını da bu hücreye ekledim.

## 28.09 · 5 saat · Keşifsel analiz, bölme ve TF-IDF'e giriş

**Keşifsel analiz:**
- Sınıf dağılımı grafiğini çizdim. En büyük sınıf Tshirts (7.069), en küçüğü Jeans (608). Aralarında yaklaşık **11,6 kat** fark var. Veri dengesiz olduğu için makro-F1'in doğruluktan daha anlamlı olduğunu not ettim.
- Görselleri açtım. Hepsi **60×80 piksel** yani çok küçük. Her sınıftan 1 ve 3 örnek içeren iki ızgara hazırladım. Görselleri analiz ettiğimde Flip Flops ile Sandals'ın birbirine çok benzediğini fark ettim.
- Ürün adlarının kelime sayısına baktım: ortalama 5,8 kelime, en az 1, en fazla 14.
- Tek kelimelik adları incelerken iki geçersiz ad daha buldum: "RTV_LoadTest" (test kaydı gibi duruyor, görseli spor ayakkabı ama etiketi Sandals) ve "productDisplayName" (ad yerine sütun başlığı yazılmış). Bunları da geçersiz adlar listesine ekledim. "Playboy" ise gerçek bir marka adı olduğu için tuttum. Kuralım şu oldu: **geçersiz** kayıtları çıkarıyorum, **zor** olanları değil.
- Temizlik sonrası: 44.446 → 44.437 (geçersiz adlar) → 44.432 (görselsizler) → **33.142** (en sık 20 sınıf).

**Bölme:**
- `train_test_split` ile iki adımda, tabakalı ve tohum 42 ile %70 / %15 / %15 böldüm: **23.199 / 4.971 / 4.972**. Sınıf oranlarının üç kümede aynı kaldığını kontrol ettim ve `split.csv` dosyasını kaydettim.
- Test kümesindeki farklı ürün adlarının **%31,9'u** train'de de geçiyor (kayıt bazında %38,6). Bu, test sonucunu biraz iyimser gösterebilir. Ödev bölme yöntemini açıkça tarif ettiği için bölmeyi değiştirmedim. Bunun yerine etkisini ölçmeye karar verdim.

**TF-IDF:** Küçük bir örnekle TF-IDF'in nasıl çalıştığına baktım. Varsayılan ayarın "T-shirt" içindeki tek harfli "t"yi attığını, yani "T-shirt" ile "Shirt"ü aynı kelimeye çevirdiğini fark ettim.

**Sorunlar:**
- Grafikte uzun sınıf adları kesildi, `tight_layout` ile düzelttim.
- İlk denemede kelime yerine karakter saydım (ortalama 34,7 çıktı). Ürün adı uzunluklarını hesaplarken `.str.split().str.len()` ile düzelttim.


## 30.09 · 5 saat · Metin modeli ve maskeleme

- Lojistik regresyon ve doğrusal SVM'i doğrulama kümesinde denedim. İlk sonuçta en zayıf sınıf Shirts'tü (F1 0,897). En sık hata Shirts → Tshirts (65 kez) idi. Bu, 28.09'da fark ettiğim "t" sorunuydu.
- **"t" sorunu:** TF-IDF varsayılan olarak yalnızca en az 2 harfli parçaları kelime sayıyor ve tireden bölüyor. Bu yüzden "T-shirt" -> "t" + "shirt" oluyor ve "t" atılıyor. Sonuçta Tshirts sınıfındaki "Puma Men Grey T-shirt" ile Shirts sınıfındaki "Turtle Check Men Navy Blue Shirt" model için aynı ana kelimeyi ("shirt") taşıyor. Model iki sınıfı yalnızca marka ve renk gibi diğer kelimelerle ayırmaya çalışıyordu.
- Kuralı tek harfleri de tutacak şekilde değiştirdim (`token_pattern=r"(?u)\b\w+\b"`) ve "t"nin sözlüğe girdiğini kontrol ettim. Aynı SVM modeliyle Shirts F1 0,906'dan **0,996**'ya çıktı. Yalnızca bu ayarı değiştirdiğim için iyileşmenin nedeninin "t" harfi olduğundan emin oldum. Sonra 3 TF-IDF ayarı × 2 model × 3 C değeri ile 18 deneme yaptım. En iyisi **karakter TF-IDF (2–5) + lojistik regresyon, C = 10** (val makro-F1 0,9875).
- **Maskeleme:** Sınıf adlarını oluşturan kelimeleri tekil ve çoğul biçimleriyle ürün adlarından sildim. Önce birkaç sınıfta adların nasıl yazıldığına baktım. Örneğin Tshirts için "T-shirt", "Tshirt", "T-Shirt" gibi farklı yazımlar vardı. "Flats", "jeggings" gibi eş anlamlıları bilerek eklemedim, çünkü sınıf adında geçmiyorlar. Maskeli adlarda val makro-F1 0,9875'ten **0,8206'ya** düştü. En çok düşen sınıflar Belts ve Wallets oldu, çünkü aynı markalar hem kemer hem cüzdan üretiyor.
- Test kümesini ilk kez burada, ayarları doğrulama kümesinde seçtikten sonra kullandım ve Tablo 1'in ilk 3 satırını doldurdum (makro-F1: LR 0,9821, SVM 0,9835, maskeli LR 0,8172).
- 28.09'da bulduğum ad tekrarının etkisini ölçmek için test ürünlerini ikiye ayırdım: adı train'de de geçen **1.917** ürün ve hiç geçmeyen **3.055** ürün. LR modelinde makro-F1 görülmüş adlarda 0,9944, görülmemiş adlarda 0,9750 çıktı (yaklaşık **1,9 puan** fark). Maskeli adlarda fark büyüdü: 0,8519'a karşı 0,7810 (yaklaşık **7 puan**). Yani ezber etkisi var ve sınıf adı silinince daha da önemli hale geliyor ama hiç görülmemiş adlarda da sonuç yüksek kalabiliyor.
- Görsel model ve füzyon aynı veriyi kullansın diye `hazir_veri.csv` dosyasını kaydettim. 

**Sorunlar:**
- "Flip-flops" maskelenince geride "-" kalıyordu. Tireli biçimleri listeye ekleyip kısa kelimelerden önce yazdım.
- Testte SVM az farkla öne geçti. Ama seçimi doğrulama kümesine göre yaptığım için LR'de kaldım. Ayrıca LR olasılık verdiği için füzyonda kullanabilecektim. 

## 03.10 · 5 saat · Görsel model

- GPU'lu yeni bir notebook açtım. ImageNet'te eğitilmiş **EfficientNetB0** kullandım. Görseller 60×80 olduğu için giriş boyutunu 128×128 seçtim. 224'e büyütmek yeni bilgi eklemeden işlemi yaklaşık 3 kat yavaşlatırdı. Görselleri esnetmeden büyüttüm, kenarları beyazla doldurdum.
- Veri artırma olarak yatay çevirme, hafif döndürme ve yakınlaştırma kullandım.
- Eğitimi iki aşamada yaptım: önce gövde donukken 15 tur, sonra son 30 katmanı açıp düşük öğrenme oranıyla ince ayar. EarlyStopping ve ModelCheckpoint kullandım. En iyi model **39. turda** geldi.
- Eğitim eğrilerini çizdim. Test sonucu: doğruluk 0,9204, makro-F1 0,9251 (Tablo 1, satır 4). En zayıf sınıflar ayakkabılar ve Tops. Buna karşılık metin modelinin maskeli adlarda zorlandığı Belts ve Wallets'ta görsel model çok başarılı.
- Füzyonda kullanmak için olasılıkları ve 1.280 boyutlu gömmeleri Drive'a kaydettim.

**Sorunlar:**
- Gömmeleri çıkarırken RAM doldu ve oturum çöktü. Train görsellerini ikinci kez belleğe almıştım (`cache()`). Modeli Drive'dan yükleyip train gömmelerini önbellek kullanmadan çıkardım.
- İnce ayar için ayırdığım 10 tur bittiğinde val kaybı hâlâ düşüyordu (0,2136) yani model doğrulama verisinde iyileşmeye devam ediyordu. Eğitimi erken kesmemek için 15 tur daha uzattım. ModelCheckpoint'i yalnızca 0,2136'dan daha iyi bir model bulunursa kaydedecek şekilde ayarladım. En iyi sonuç 39. turda geldi (0,2015). Son turlarda val kaybı durağanlaşırken eğitim kaybı düşmeye devam etti bunu aşırı öğrenmenin başladığının işareti olarak yorumladım ve durup en iyi tur olarak 39. turu seçtim.

## 05.10 · 4 saat · Füzyon

- Füzyon için CPU'da yeni bir notebook açtım. Önce metin ve görsel dosyalarındaki ürün sırasının aynı olduğunu kontrol ettim .
- Metin modelini aynı ayarlarla yeniden eğitip olasılıklarını aldım. İlk notebook'ta yalnızca tahminleri almıştım ama füzyon için olasılıklar gerekiyordu.
- **Geç füzyon:** İki model ayrı ayrı tahmin yapıyor, sonra olasılıkları `w × metin + (1 − w) × görsel` ile birleştiriliyor. w'yi doğrulama kümesinde 0'dan 1'e 0,05 adımla tek tek deneyip en yüksek makro-F1'i vereni seçtim: orijinal adlarda w = 0,55, maskeli adlarda w = 0,50. Maskeli adlarda metin zayıfladığı için görsele biraz daha fazla ağırlık verilmesi mantıklı. Seçilen w ile testi bir kez ölçtüm: orijinal **0,9850**, maskeli **0,9535** makro-F1.
- **Ara füzyon:** Bu sefer iki bilgiyi baştan birleştirip tek bir model eğittim. TF-IDF çok fazla sütun ürettiği için (41.680) metni TruncatedSVD ile 256 sayıya sıkıştırdım.  Sonra her ürün için görsel gömmeyi (1.280) ve metin özetini (256) yan yana koydum, StandardScaler ile ölçekledim ve küçük bir Keras MLP eğittim. Eğitimi en fazla 50 tur olarak ayarlamıştım ama EarlyStopping modeli orijinal adlarda 8., maskeli adlarda 7. turda durdurdu ve en iyi tura (3. ve 2. tur) geri döndü. MLP çok hızlı öğrenip birkaç turda ezberlemeye başladı. Test: orijinal 0,9755, maskeli 0,9445.
- **Görselin katkısı:** Her füzyon sonucundan metnin tek başına sonucunu çıkardım. Orijinal adlarda görsel neredeyse hiç katkı sağlamıyor (geç füzyonda +0,3 puan, ara füzyonda −0,7). Maskeli adlarda katkı **+13,6** (geç) ve +12,7 (ara) puana çıkıyor. Buradan çıkardığım sonuç: Ad sınıfı söylediğinde görsele pek gerek yok, ad zayıfladığında görsel açığı kapatıyor.
- Hata analizine başladım. En iyi model olan geç füzyon (orijinal ad) için normalize **karışıklık matrisini** çizdim. 4.972 test ürününde 64 hata var. 20 sınıfın 12'sinde doğru tahmin oranı 1,00'a yuvarlanıyor. Karışıklıklar çoğunlukla ayakkabı ve terlik sınıfları arasında (örneğin Sandals -> Flip Flops %7).

**Sorun:** Ara füzyon geç füzyondan yaklaşık 1 puan düşük çıktı. Test sonucunu gördüğüm için ayarları değiştirmedim. Olası nedenleri tartışmaya ekledim (SVD'de bilgi kaybı, MLP'nin çabuk ezberlemesi).

## 06.10 · 3 saat · Hata analizi

- En çok karışan 5 sınıf çiftini kodla buldum. Hepsi ayakkabı ve terlik sınıfları: 64 hatanın 44'ü (%69). En sık hata Casual -> Sports Shoes (14).
- Modelin en emin olduğu 16 hatayı görselleri ve adlarıyla gösterdim, tek tek inceledim. Bir kısmında modelin haklı, etiketin yanlış olduğunu fark ettim.
- 64 hatanın hepsini ürün adındaki sınıf kelimelerine bakan bir kuralla grupladım: %78'inde ad ürün türünü söylemiyor, %17'si olası etiket hatası.
- Yöntemi Türkçe ürün adları olan bir kataloğa uygulasak hangi sorunlarla karşılaşabileceğimizi yazdım.

**Sorun:** İlk hata görselindeki yazılar okunmuyordu. Yazıları büyüttüm, gerçek sınıfı yeşil, tahmini kırmızı yazdım. Ayrıca 16 örneğe bakınca "hataların çoğu etiket hatası" diye düşünmüştüm. 64 hatanın tamamını sayınca bu oranın %17 olduğunu gördüm. Az sayıda örneğe bakarak genelleme yapmamam gerektiğini öğrendim.

## 07.10 · 1 saat · Düzenleme

Günlüğü düzenledim. 

## 08.10 · 1,5 saat · Teslim hazırlığı

Teslim için kalan işleri planladım. İlk notebook'u temizledim: hatalı ve tekrar eden hücreleri sildim, import'ları kurulum hücresinde topladım, bölüm başlıkları ekledim.



## 10.10 · 4 saat · Kontrol ve Grad-CAM

- İlk notebook'u "Tümünü çalıştır" ile baştan çalıştırdım. Hatasız çalıştı ve bütün sayılar önceki sonuçlarla aynı çıktı.
- Diğer iki notebook'u temizledim. `requirements.txt`, `README.md` ve raporun LaTeX taslağını hazırladım.
- Füzyon notebook'unda bir başlığın kod hücresine yazıldığını fark ettim bu "Tümünü çalıştır"ı durduruyordu. Düzeltip notebook'u baştan çalıştırdım.
- Ödev sınıf bazlı F1 de istiyordu. En iyi model (geç füzyon) için test kümesinde sınıf bazlı F1 tablosunu ekledim. En düşük değerler yine ayakkabı ve terlik sınıflarında (Formal Shoes 0,938), yani hata analiziyle tutarlı.
- **Ek görev, Grad-CAM:** Ayrı bir notebook (`04_ek_gorevler`) açtım. Görsel modelin karar verirken görselin neresine baktığını gösteren ısı haritaları çizdim. Doğru tahminlerde model ürünün ayırt edici parçasına bakıyor: saatte kadran, gözlükte camlar, tişörtte gövde. Yanlış tahminlerde saplı sırt çantalarının Handbags sanıldığını ve haritanın sapı gösterdiğini gördüm. Casual <-> Sports karışmalarında iki sınıfın haritası da ayakkabının aynı bölgesini gösteriyor.

**Sorunlar:**
- Colab'daki TensorFlow 2.21.0'a güncellenmişti. Model 2.20.0 ile eğitildiği için `requirements.txt` dosyasına 2.20.0 yazdım. Görsel notebook'u baştan çalıştırırsam model yeniden eğitilip sonuçlar değişebileceği için onu mevcut çıktılarıyla bıraktım.
- Grad-CAM'de ilk denediğim cüzdan görselinde harita ürünü değil çevresini gösterdi. Önce bir hata yaptığımı düşündüm, bu yüzden 6 farklı sınıfla tekrar denedim. Diğerlerinde harita ürünün üzerindeydi. Harita yalnızca 4×4 çözünürlükte olduğu için bazen ürünün çevresi de vurgulanabiliyor. Cüzdan örneğini yöntemin sınırını göstermek için bilerek notebook'ta bıraktım.

## 11.10 · 3 saat · Gradio demosu

- **Ek görev, Gradio:** Görsel yükleyip ürün adı yazınca sınıf olasılıklarını gösteren bir demo hazırladım. Demoda geç füzyonu kullandım çünkü en iyi sonucu o veriyor ve iki model ayrı çalıştığı için kurması kolay.
- Demo üç durumda çalışıyor: yalnızca görsel (görsel model), yalnızca ad (metin modeli), ikisi birlikte (geç füzyon, w = 0,55).
- Bir tişört görseliyle denedim: Görsel tek başına Tshirts %92, "…White Top" adı tek başına Tops %100 dedi. İkisini birlikte verince sonuç Tops %58, Tshirts %42 çıktı. Bu sayıları formülle elle de hesapladım ve aynı çıktı (0,55 × 1,00 + 0,45 × 0,08 ≈ 0,58). Geç füzyonun iki model çelişince ağırlığa göre karar verdiğini burada açıkça gördüm.

**Sorun:** İlk yazdığım demo görsel olmadan çalışmıyordu. Ad boşsa görsele geçiyordu ama görsel yoksa ada geçmiyordu. Fonksiyonu üç durumu da kapsayacak şekilde düzenledim.


## 13.10 · 3 saat · Teslim hazırlığı

- Günlüğü gün gün ayrıntılı olarak yeniden düzenledim.
- `README.md` dosyasını projenin akışını anlatacak şekilde yeniden yazdım. 
- `requirements.txt` dosyasına Gradio sürümünü (6.29.0) ekledim.
- Ek görevler notebook'una başlıklar ve açıklamalar ekledim. Füzyon ve ek görevler notebook'larını baştan çalıştırdım, bütün sonuçlar önceki değerlerle aynı çıktı.



## Önemli kararlar

- **Hatalı satırları silmedim, onardım.** Veri kaybı olmasın istedim.
- **Yalnızca geçersiz kayıtları çıkardım.** "NA", "RTV_LoadTest" ve sütun başlığı yazılmış kaydı çıkardım, "Playboy" gibi zor ama gerçek adları tuttum.
- **Bölmeyi değiştirmedim.** Aynı adın train ve testte olmasının etkisini ayrıca ölçtüm.
- **Maskelemeyi yalnızca sınıf adı kelimeleriyle sınırladım.** "Flats", "polo", "slippers" gibi eş anlamlıları bilerek eklemedim.
- **Bütün seçimleri doğrulama kümesiyle yaptım:** TF-IDF ayarı, C, en iyi tur ve füzyon ağırlığı. Test sonucuna bakıp hiçbir ayarı değiştirmedim.
- **Görsel model için 128×128 giriş seçtim.** Görseller 60×80 olduğu için 224'e büyütmek bilgi eklemeden işlemi yavaşlatırdı.
- **Ek görevleri ayrı bir notebook'ta yaptım.** Böylece zorunlu kısımların sonuçları hiç etkilenmedi.

## Sonuçlar (Tablo 1, test kümesi)

| No | Model | Ürün adı | Doğruluk | Makro-F1 |
|---|---|---|---:|---:|
| 1 | TF-IDF + lojistik regresyon | Orijinal | 0,9843 | 0,9821 |
| 2 | TF-IDF + doğrusal SVM | Orijinal | 0,9849 | 0,9835 |
| 3 | En iyi metin modeli (LR) | Maskelenmiş | 0,8407 | 0,8172 |
| 4 | Keras transfer öğrenme (görsel) | Kullanılmıyor | 0,9204 | 0,9251 |
| 5 | Geç füzyon (w = 0,55) | Orijinal | 0,9871 | **0,9850** |
| 6 | Geç füzyon (w = 0,50) | Maskelenmiş | 0,9491 | **0,9535** |
| 7 | Ara füzyon | Orijinal | 0,9773 | 0,9755 |
| 8 | Ara füzyon | Maskelenmiş | 0,9395 | 0,9445 |

- **Görselin katkısı:** Orijinal adlarda neredeyse yok (geç +0,3 / ara −0,7 puan), maskeli adlarda yaklaşık 13 puan (geç +13,6 / ara +12,7).
- **Hata analizi:** 64 hatanın %69'u ayakkabı ve terlik çiftlerinde. Hataların %78'inde ad ürün türünü söylemiyor, %17'si olası etiket hatası.

## Yapay zekâ kullanımı

Claude'u  kullandım:


- **Kod:** Aşamalarda   boşluklu iskelet alıp kodu kendim tamamladım.. Her hücreyi kendim çalıştırdım, çıktıları paylaştım ve anlamadığım yerleri sordum. Birleştirilmiş veri hazırlama hücresi ve bellek çökmesinden sonraki kurtarma hücresi Claude tarafından yazıldı.
- **Ek görevler:** Grad-CAM ve Gradio kodları (`04_ek_gorevler`) ile yasaklı sütunlar bölümündeki iki kod satırı Claude tarafından satır satır açıklanarak verildi. Bu kodları kendim çalıştırdım ve sonuçlarını yorumladım.
- **Hata ayıklama:** Yol hataları, DataFrame/Series karışıklığı ve bellek çökmesi gibi sorunların nedenlerini birlikte bulduk.
- **Notebook temizliği ve kontrol:** Notebook'ların düzenlenmesinde (gereksiz hücrelerin silinmesi, bölüm başlıkları, kısa açıklamalar) yardım aldım. Hepsini Colab'da kendim çalıştırıp test ettim.
- **Teslim dosyaları:** `README.md`, `requirements.txt` ve raporun taslağı, elde ettiğim sonuçlar ve yorumlarım kullanılarak Claude ile birlikte yazıldı. 


