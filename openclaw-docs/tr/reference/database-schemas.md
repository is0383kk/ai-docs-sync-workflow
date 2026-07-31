---
read_when:
    - Daha yeni bir veritabanı şeması hatısını tanılama
    - Güncelleme veya sürüm düşürme öncesinde veritabanı uyumluluğunu denetleme
    - Daha eski bir OpenClaw sürümü için veritabanını kurtarma
summary: OpenClaw SQLite veritabanı konumları, şema sürümleri, bütünlük denetimleri ve sürüm düşürme kurtarması
title: Veritabanı şemaları
x-i18n:
    generated_at: "2026-07-26T23:00:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 73993e2c593ba460784108aedef70bbfb499e525c709d6d6bdd956ccf93e0ddc
    source_path: reference/database-schemas.md
    workflow: 16
---

OpenClaw, denetim düzlemi durumunu genel bir SQLite veritabanında, agent verilerini ise agent başına bir SQLite veritabanında depolar. Şema geçişleri, bir veritabanı açıldığında ileri yönde çalıştırılır. Eski OpenClaw derlemeleri, daha yeni bir şema tarafından yazılmış veritabanlarını reddeder.

## Veritabanı düzeni

| Kapsam                | Varsayılan yol                                               | İçerik                                                                                              |
| -------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Genel denetim düzlemi | `~/.openclaw/state/openclaw.sqlite`                        | Paylaşılan yapılandırma durumu, kayıtlar, onaylar, plugin durumu ve paylaşılan çalışma zamanı durumu             |
| Agent başına veri düzlemi | `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` | Oturumlar, transkriptler, bellek dizinleri, kimlik doğrulama durumu, konuşma durumu ve agent kapsamlı çalışma zamanı durumu |

Görev kaydı ve yörünge verileri dâhil olmak üzere yüksek hacimli veya yaşam döngüsüne özgü birkaç özellik, ayrılmış SQLite depoları kullanır.

## Sürümleme sözleşmesi

Her veritabanı şemasını iki yerde kaydeder:

- `PRAGMA user_version`, SQLite şema sürümüdür.
- Birincil `schema_meta` satırı; `role`, `agent_id`, `schema_version` ve `app_version` değerlerini kaydeder. `app_version`, şema meta verilerini en son yazan OpenClaw derlemesidir.

OpenClaw, desteklenen eski bir veritabanını açarken yalnızca ileri yönlü geçişleri uygular. `user_version` değeri çalışan derlemeden daha yeni olan bir veritabanını reddeder ve `newer schema version` hatası bildirir. Gateway, başlatılmadan önce kayıtlı tüm veritabanlarını denetler. `openclaw update` ayrıca, bildirilen şema desteği diskteki bir veritabanından daha eski olan bir paketi veya kaynak hedefini reddeder. Şema meta verileri eklenmeden önce yayımlanan hedef paketlerde ön denetim yapılamaz.

OpenClaw'u npm aracılığıyla elle yüklemek, güncelleyici korumasını atlar. Veritabanı açma denetimleri uyumsuz bir derlemeyi yine de reddeder.

## Agent şema geçmişi

| Sürüm | Değişiklik                                                                                                                                                                                                                                                         | İlk sürüm                                   |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| 1       | İlk agent başına depo ([#88349](https://github.com/openclaw/openclaw/pull/88349))                                                                                                                                                                            | `v2026.5.30-beta.1`, `v2026.7.1` sürümüne kadar kararlı |
| 2       | Bellek dizini kimliği ([#104449](https://github.com/openclaw/openclaw/pull/104449))                                                                                                                                                                            | `v2026.7.2-beta.1`                              |
| 4       | Oturumlar ve transkriptler SQLite'a taşındı ([#98236](https://github.com/openclaw/openclaw/pull/98236))                                                                                                                                                         | `v2026.7.2-beta.1`                              |
| 5-6     | Terminal güncelliği ve durum yaşam döngüsü ([#104859](https://github.com/openclaw/openclaw/pull/104859))                                                                                                                                                           | `v2026.7.2-beta.1`                              |
| 7       | Girdi başına yaşam döngüsü durumu izdüşümü ([#106151](https://github.com/openclaw/openclaw/pull/106151))                                                                                                                                                            | `v2026.7.2-beta.1`                              |
| 8       | Transkript başına oturum kökeni ([#106766](https://github.com/openclaw/openclaw/pull/106766))                                                                                                                                                                | `v2026.7.2-beta.2`                              |
| 9       | `STRICT` tabloları ([#108663](https://github.com/openclaw/openclaw/pull/108663))                                                                                                                                                                                  | `v2026.7.2-beta.2`                              |
| 10      | Gerçekleştirilmiş etkin transkript yolları ([#108851](https://github.com/openclaw/openclaw/pull/108851))                                                                                                                                                             | Yayımlanmadı                                      |
| 11      | Kiralamalar, kalıcı teslimat, konuşma adresleri ve Heartbeat sonuçları ([#109636](https://github.com/openclaw/openclaw/pull/109636), [#95838](https://github.com/openclaw/openclaw/pull/95838), [#109999](https://github.com/openclaw/openclaw/pull/109999)) | Yayımlanmadı                                      |

Sürüm 3, sürüm 4'e dâhil edilen, yayımlanmamış bir geliştirme adımıydı.

## Durum şeması geçmişi

| Sürüm | Değişiklik                                                                                                   | İlk sürüm       |
| ------- | -------------------------------------------------------------------------------------------------------- | ------------------- |
| 1       | İlk paylaşılan durum veritabanı                                                                            | `v2026.5.30-beta.1` |
| 2       | Yalnızca meta veri içeren ileti denetim olayları ([#103903](https://github.com/openclaw/openclaw/pull/103903))         | `v2026.7.2-beta.1`  |
| 3       | `STRICT` tabloları ve şema sapmasına karşı sağlamlaştırma ([#108663](https://github.com/openclaw/openclaw/pull/108663)) | `v2026.7.2-beta.2`  |
| 4       | Oturum izleme kökeni, kodlanmış gözcü satırlarının yerini alır                                                  | Yayımlanmadı          |

## Bütünlük denetimleri

| Zaman                                        | Denetim                                                           |
| ------------------------------------------- | --------------------------------------------------------------- |
| Her açılışta                                  | `schema_meta` tablosunu ve birincil meta veri satırını doğrula       |
| Bekleyen bir geçişten önce                  | Tam bütünlük, yabancı anahtar, rol, şema ve dizin taraması çalıştır |
| Gateway arka plan doğrulayıcısı                 | Tam taramayı yaklaşık günde bir kez çalıştır ve sonuçları günlüğe kaydet              |
| Doctor, yedek doğrulaması ve Compaction | Veritabanını kabul etmeden veya yeniden yazmadan önce tam taramayı çalıştır    |

Gateway ön denetimi yalnızca şema üstbilgilerini okur. Geçiş gerektirmeyen veritabanlarının daha yavaş tam taramasından arka plan doğrulayıcısı sorumludur.
Karantina kararları yalnızca ayrılmış bir `openclaw-quarantine.sqlite` deposunda tutulur; böylece karantinaya alınan veritabanları hasar görse bile korunur. Doğrulama sonuçları günlüğe kaydedilir.

## Sorun giderme

### 2026.7.2 sürümüne güncelledikten sonra neden geri dönemezsiniz?

`v2026.7.1` dâhil olmak üzere bu sürüme kadarki her sürüm, agent şeması 1'i ve durum şeması 1'i kullandı. 2026.7.2 sürüm serisi (`v2026.7.2-beta.1` ile başlayarak), ilk başlatmada veritabanlarınızı ileri taşır. Bu geçiş tek yönlüdür: veriler daha yeni şemaya göre yeniden yazılır ve sonrasında daha eski bir OpenClaw yüklemek bu işlemi geri almaz. Eski derleme, veritabanının sahibi olan derlemeyi belirten bir `newer schema version` hatasıyla başlatmayı reddeder.

İkili dosyanın sürümünü düşürmek, verilerin sürümünü hiçbir zaman düşürmez. Güncellemeden sonra 2026.7.2'den eski bir sürümü çalıştırmanız gerekiyorsa üç seçeneğiniz vardır:

1. Güncellemeden önce alınmış bir yedeği geri yükleyin. Büyük güncellemelerden önce [yedekleri oluşturun ve doğrulayın](/tr/cli/backup).
2. Eski derlemeyi ayrı bir durum diziniyle (`OPENCLAW_STATE_DIR`) çalıştırın. Temiz bir başlangıç yapar; daha yeni derlemeye döndüğünüzde kullanabilmeniz için taşınmış verileriniz değiştirilmeden kalır.
3. Aşağıdaki elle sürüm düşürme prosedürünü izleyin. Bu prosedür desteklenmez ve doğrulanmış bir yedek olmadan veri kaybı riski taşır.

2026.7.2'den itibaren `openclaw update`, mevcut veritabanlarınızı açamayan bir sürümü yüklemeyi reddeder; dolayısıyla güncelleyici sizi bu duruma sokmaz. Daha eski bir sürümü npm aracılığıyla elle yüklemek bu korumayı atlar; veritabanları eski ikili dosyayı yine reddeder, ancak bunu yalnızca dosya yüklendikten sonra yapar.

### Gateway, daha yeni şema sürümü hatasıyla başlamayı reddediyor

Veritabanlarınızı daha yeni bir OpenClaw derlemesi yazmıştır ve çalışan derleme daha eskidir. Hata ve Gateway başlatma günlüğü, veritabanının sahibi olan derlemeyi (`app_version`) belirtir. Bu sürümü veya daha yenisini yükleyin ya da yukarıdaki seçeneklerden birini kullanın. Hatayı susturmak için veritabanını düzenlemeyin.

### Bütünlük doğrulaması başarısız olduktan sonra bir veritabanı karantinaya alındı

Arka plan doğrulayıcısı dosyanın bozuk olduğunu kanıtlamıştır ve artık her açılışta yeniden tarama yapmak yerine işlem hızla başarısız olur. Veritabanını bir yedekten geri yükleyin veya onarın, ardından karantina kaydını temizlemek için `openclaw doctor --fix` komutunu çalıştırın. Karantina kaydının kendisi temizlenemiyorsa Doctor açık bir hata bildirir; temiz olduğunu bildirene kadar komutu yeniden çalıştırın.

## Sürüm düşürmeler desteklenmez

Elle şema sürümü düşürme işlemleri, riski kabul eden agent'lar ve operatörler içindir. Herhangi bir veritabanını düzenlemeden önce [bir yedek oluşturun ve doğrulayın](/tr/cli/backup). Gateway'i ve veritabanını açabilecek tüm işlemleri durdurun.

Genel prosedür şöyledir:

1. Hedef sürümün şemasını ve geçişlerini okuyun.
2. Tek bir işlem içinde, hedef sürümden sonra eklenen tüm tabloları, dizinleri, tetikleyicileri ve sütunları kaldırın.
3. `PRAGMA user_version` ve `schema_meta.schema_version` değerlerini hedef sürüme ayarlayın.
4. Gateway'i başlatmadan önce hedef sürümün tam veritabanı doğrulamasını çalıştırın.

### Örnek: agent şeması 11'den 9'a

Şema 10, etkin transkript izdüşümünü ekledi. Şema 11; kiralamaları, kalıcı teslimatı, konuşma adresi durumunu ve Heartbeat sonuçlarını ekledi. QMD koordinasyonu `state_leases` içindeki satırları kullanır; korunması gereken ayrı bir QMD tablosu yoktur.

Tam olarak hangi şemanın yazdığını inceledikten sonra, etkilenen her agent başına veritabanında eşdeğer SQL'i çalıştırın:

```sql
BEGIN IMMEDIATE;

DROP TABLE IF EXISTS heartbeat_outcomes;
DROP TABLE IF EXISTS conversation_deliveries;
DROP TABLE IF EXISTS state_leases;
DROP TABLE IF EXISTS session_transcript_active_events;

ALTER TABLE session_transcript_index_state DROP COLUMN active_event_count;
ALTER TABLE session_transcript_index_state DROP COLUMN active_message_count;
ALTER TABLE conversations DROP COLUMN delivery_target;

PRAGMA user_version = 9;
UPDATE schema_meta
SET schema_version = 9,
    updated_at = unixepoch('now') * 1000
WHERE meta_key = 'primary';

COMMIT;
```

Bu işlem; devam eden teslimat işlemleri, kiralamalar, Heartbeat sonuçları ve türetilmiş etkin transkript izdüşümü dâhil olmak üzere sürüm 10-11 durumunu atar. Hatalı bir sürüm düşürme durumunda doğrulanmış yedekten geri yükleyin.
