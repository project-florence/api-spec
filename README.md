# Florence

Florence, yatırım yapılabilir şirketler hakkında risk analizi ve puanlama sunan kapsamlı bir finansal analiz platformudur. Uygulama, doğrudan alım-satım işlemine (trading) izin vermeyen; bunun yerine veriye dayalı öngörüler, dinamik takipler ve kişiselleştirilmiş risk değerlendirmeleri ile kullanıcının rasyonel ve profesyonel yatırım kararları almasını sağlayan bir "gözlem ve strateji" terminali olarak kurgulanmıştır.

## 📱 Platformlar

Proje, kullanıcıların finansal verilerine her ortamdan ulaşabilmesi için çoklu platform (cross-platform) ekosistemi olarak tasarlanmıştır:
*   **Android (Mobil):** Hareket halindeyken dinamik takip ve hızlı analiz için optimize edilmiş mobil arayüz.
*   **Web Terminali:** Detaylı grafik incelemeleri ve tarayıcı tabanlı geniş ekran analizleri için tasarlanmış web yüzü.
*   **Masaüstü (Desktop):** Çoklu monitör desteği ve profesyonel düzeyde veri takibi isteyen kullanıcılar için geliştirilecek yerel (native) masaüstü yazılımı.

## 🎯 Temel Modüller ve Özellikler

*   **Scout (Analiz Motoru):** Kullanıcının girdiği "Bütçe", "Vade (Kısa, Orta, Uzun)" ve "Risk Toleransı" parametrelerini arka planda işleyerek rasyonel, puanlanmış hisse portföyü önerileri sunan akıllı filtreleme algoritması.
*   **Watchlist (Takip Listesi):** Kullanıcının mercek altına aldığı şirketlerin güncel durumlarının ve sektörel hareketlerinin tek bir ekrandan dinamik olarak takip edildiği modül.
*   **Kullanıcı Profili:** Kişisel ayarların, portföy stratejilerinin ve genel sistem davranışının yönetildiği kontrol paneli.

## 🏗️ Mimari ve Teknolojiler

Florence, kod sürdürülebilirliği ve modüler yapı odaklı sektör standartlarında inşa edilmektedir.

**Android İstemcisi:**
*   **Dil:** Java
*   **Mimari:** MVVM (Model-View-ViewModel) - Ekran tasarımı (View) ile veri ve iş mantığının (ViewModel) birbirinden tamamen izole edildiği, çökme direncine sahip modern bellek mimarisi.
*   **Ağ Katmanı:** Retrofit
*   **Arayüz (UI/UX):** Koyu antrasit tema (Dark Theme) odaklı, göz yormayan, profesyonel bir finansal arayüz tasarımı.

**Web ve Masaüstü İstemcileri:**
*   *Bu platformlar için teknoloji yığınları (stack) ve mimari detaylar projenin ilerleyen fazlarında bu dokümana eklenecektir.*

## 🚀 Kurulum

*(Not: Projenin Backend/API mimarisi geliştirme aşamasında olduğu için, ağ bağlantıları ve sunucu konfigürasyon adımları API entegrasyonu tamamlandığında buraya eklenecektir.)*

Yerel ortamda ön yüzleri test etmek için repoyu bilgisayarınıza klonlayabilirsiniz:

```bash
git clone [https://github.com/project-florence/florence.git](https://github.com/project-florence/florence.git)
