# Petfinder: Birlikte Onları Buluyoruz

## Demo Videosu

<a href="https://vimeo.com/user264746150/petfinder-demo?fl=ip&fe=ec">
  <img width="1691" height="680" alt="thumbnail" src="https://github.com/user-attachments/assets/8590f550-622a-4074-89bf-ae2581ebbee9" />
</a>


## Genel Bakış

**Petfinder**, kaybolan hayvanları sahipleriyle yeniden buluşturmak için toplulukları bir araya getiren, uçtan uca bir web uygulamasıdır. Laravel deneyimi 4 ay, toplam kodlama deneyimi ise bir yıldan az olan bir geliştirici tarafından tek başına **toplam 6 haftada** geliştirilmiştir.

Uygulama; derinlik yönetimli özyinelemeli yorum dizileri, Eloquent observer’larıyla soft delete desenleri, Docker konteynerleştirmesiyle Typesense üzerinden tam metin arama, Redis önbellekleme, test kapsamı ve karmaşık toplu işlemler gibi üretim ortamı düzeyinde mimari kararları sergiler. Bunların hiçbiri tipik bir ilk projede karşılaşılacak özellikler değildir.

⚠️ Önemli Notlar <br>
**Canlı Demo:** Altyapı maliyetleri nedeniyle demo şu anda canlı olarak barındırılmıyor. <br>
**Kaynak Kodu:** Kaynak kodunun tamamı **özel bir depoda** tutulmaktadır. Bu herkese açık depo, referans ve dokümantasyon merkezi olarak hizmet verir.

## Bu Projeyi Özel Kılanlar

### 1. **Derinlik Yönetimli Özyinelemeli Yorum Sistemi**

Reddit’in yapısını andıran, iş parçacıklı bir yorum bölümü. Yorumların sınırsız sayıda yanıtı olabilir; ancak kullanıcı deneyimini ve veritabanı performansını korumak için yanıtlar yapılandırılabilir derinlik sınırlarına tabidir.

<img width="1403" height="656" alt="Recursive_comments_in_action" src="https://github.com/user-attachments/assets/0ef091fc-5beb-4f65-a2f0-15e637fb0336" />

<br>

**Öne Çıkan Uygulama Ayrıntıları:**

- Yorumlar, `parent_id` kullanılarak ebeveyn-çocuk ilişkileriyle tek bir tabloda saklanır
- Sorgular, Laravel’in `with()` eager loading özelliğiyle yüklenir
- Derinliğe dayalı yanıt kısıtlamaları: kullanıcılar 4. derinlik seviyesinin ötesinde yanıt veremez; sonraki etkileşimler özel bir yorum ağacı görünümüne yönlendirilir
- Okunabilirlik için daraltılabilir yorum dizileri
- `(edited)` göstergesiyle yorum düzenleme
- Soft delete, ağacın tamamını korur; silinen yorumlar yapısal bütünlük korunarak `[deleted]` şeklinde gösterilir

```php
// ? RECURSIVE COMMENT COUNT LOGIC
    public function countTotalComments()
    {
        $count = $this->repliesRecursive->count();

        foreach ($this->repliesRecursive as $reply) {
            $count += $reply->countTotalComments();
        }

        return $count;
    }
```

**Bu Tercihin Nedeni:** Özyinelemeli bir yorum sistemi oluşturmak basit değildir. Geliştiricilerin çoğu ilk büyük projelerinde bu tür bir sistemden kaçınır. Bu yaklaşım, veritabanı tasarımı konusundaki düşünme becerisini ve kullanıcı deneyiminin dikkate alındığını gösterir.

---

### 2. **Soft Delete Mimarisi ve Observer Deseni**

<img width="1403" height="656" alt="User_observer_triggering_comment_deletion-optimize" src="https://github.com/user-attachments/assets/651f6066-3252-4fb5-8863-e637f1cf8ffa" />
<br>


Bir kullanıcı hesabını sildiğinde, konuşma dizilerini korumak için yorumları zincirleme silme yerine soft delete ile silinmelidir. Bunun için incelikli bir yarış koşulunun çözülmesi gerekti.

**Sorun:**

Veritabanı yabancı anahtarındaki `onDelete('cascade')`, Laravel observer’ları çalışmadan *önce* devreye girer. Observer çalıştığında `user_id` zaten NULL olur ve sorgu silinecek bir şey bulamaz.

**Çözüm:**

`deleted()` yerine silme işleminden önce çalışan `deleting()` olayını kullanmak:

```php
// UserObserver.php
 // ? WHY ARE WE USING DELETING
    // The issue stems from a race condition with database foreign key constraints. 
    // When a user is deleted, the database's onDelete('set null') constraint executes 
    // automatically, setting comments.user_id to NULL before the observer's deleted() 
    // event fires. By the time the observer runs and queries for comments, all records 
    // have already lost their user_id reference, resulting in zero matches.

    /**     
     * Handle the User "deleting" event.     
     * Fires BEFORE the deletion occurs.     
     */
    public function deleting(User $user): void
    {
        // ? WE ARE SOFT DELETING COMMENTS OF USER WHEN DELETION OCCURS. SO THAT THE COMMENTS DONT GET DELETED
        DB::transaction(function () use ($user) {
            // ? SOFT DELETE USER'S COMMENTS ON EVERY POST
            $comments = Comment::where('user_id', $user->id)->get();
            $posts = AnimalPost::where('user_id', $user->id)->get();
            foreach ($comments as $comment) {
                Cache::tags(["post_comments_{$comment->commentable_id}"])
                    ->flush();
                $comment->delete();
            }

            foreach ($posts as $post) {
                $post->unsearchable();
                $post->forceDelete();
            }
        });
    }
```

Observer ayrıca **Typesense indekslemesinin güncel kalmasını** sağlar. Gönderiler controller üzerinden silindiğinde observer arama indeksinin güncellenmesini tetikler ve sahipsiz kayıtların oluşmasını önler.

---

### 3. **Typesense ve Docker ile Tam Metin Arama**

Gönderiler, birden çok filtreyle birlikte **Typesense**’te (üretim ortamı düzeyinde bir arama motoru) indekslenir: 
hayvanın cinsiyeti, hayvan türünün adı, hayvan ırkının adı, durum, hayvanın adı, hayvanın rengi, il, başlık, açıklama, ülke.

**Mimari:**

- Typesense, ortamlar arasında tutarlılık sağlamak için **Docker konteynerinde** çalışır
- Gönderiler controller’lar aracılığıyla Typesense ile eşitlenir (gelecekteki bir uygulamada observer’lar kullanılacak şekilde güncellenecektir)
- Sayfalama, tüm sonuç kümeleri yüklenmeden verimli biçimde gerçekleştirilir
- Mobil cihazlar için optimize edilmiş arama filtreleri


**Bu Tercihin Nedeni:** Typesense ve Docker, bir öğrenci projesi için kurumsal düzeyde altyapıdır. Geliştiricilerin çoğu basit MySQL LIKE sorgularını tercih eder.

---

### 4. **Redis Önbellekleme ve Observer Otomasyonu**

Yorumlar önbelleğe alınır ve CRUD işlemlerinden sonra önbellek otomatik olarak geçersiz kılınır.

```php
// Comment Observer - keeps cache index in sync
 /**
     * Handle the Comment "created" event.
     */
    public function created(Comment $comment): void
    {
        $this->invalidatePostCommentCache($comment);
    }

    /**
     * Handle the Comment "updated" event.
     */
    public function updated(Comment $comment): void
    {
        $this->invalidatePostCommentCache($comment);
    }

    /**
     * Handle the Comment "deleted" event.
     */
    public function deleted(Comment $comment): void
    {
        Log::info('Comment deleted observer fired', ['comment_id' => $comment->id, 'commentable_id' => $comment->commentable_id,]);
        $this->invalidatePostCommentCache($comment);
    }

    /**
     * Handle the Comment "restored" event.
     */
    public function restored(Comment $comment): void
    {
        $this->invalidatePostCommentCache($comment);
    }

    /**
     * Handle the Comment "force deleted" event.
     */
    public function forceDeleted(Comment $comment): void
    {
        $this->invalidatePostCommentCache($comment);
    }

    private function invalidatePostCommentCache(Comment $comment)
    {
        Cache::tags(["post_comments_{$comment->commentable_id}"])
            ->flush();
    }
```

Bu sayede her sayfa yüklemesinde yorum ağaçlarının yeniden hesaplanması önlenir; bu, sık karşılaşılan bir performans darboğazıdır.

---

### 5. **Veritabanı İşlemleriyle Toplu İşlemler**

<img width="1403" height="656" alt="Bulk_operations_in_action-optimize" src="https://github.com/user-attachments/assets/a221c3fb-aba3-457c-a185-6ca10fa40745" />
<br>

Kullanıcılar kendi gönderilerini toplu olarak silebilir veya geri yükleyebilir ve kendi yorumlarını silebilir. Yönetici bu işlemleri herhangi bir kullanıcının içeriğinde gerçekleştirebilir.

- Birden fazla gönderi seçme
- İşlem güvenliğiyle soft delete
- Çöp kutusundan geri yükleme
- Herhangi bir kullanıcının içeriğinde yönetici işlemleri
- Gerçek zamanlı önbellek geçersizleştirme

```php
public function bulkRestore(Request $request)
    {
        $validated = $request->validate([
            'checkboxIds.*' => ['exists:animal_posts,id']
        ]);

        $allIds = $validated['checkboxIds'];

        DB::transaction(function () use ($allIds) {
            foreach ($allIds as $id) {
                $animalPost = AnimalPost::onlyTrashed()->find($id);
                Gate::allows('restore', $animalPost);

                $animalPost->restore();
                $animalPost->searchable(); // ? REMOVE FROM TYPESENSE INDEX
            }
        });

        if (Auth()->user()->roles->contains('name', 'user')) {
            $route = 'user.dashboard';
        } else {
            $route = 'admin.dashboard';
        }

        return redirect()->route($route)->with('posts_restored', 'Uw post(s) is/zijn succesvol hersteld!');

    }
```

**Bu Tercihin Nedeni:** Toplu işlemler, dikkatli işlem yönetimi ve önbellek geçersizleştirme gerektirir. Bunları doğru yapmak veri tutarsızlıklarını önler.

---

### 6. **Testler ve Factory’ler**

Testler kimlik doğrulamayı, soft delete işlemlerini, toplu işlemleri ve arama işlevselliğini doğrular. Factory’ler veri üretimini rastgeleleştirir:

<img width="1266" height="516" alt="Running_tests_operation" src="https://github.com/user-attachments/assets/f7d21c7e-6884-44dc-ac9f-d8b9a5871cf9" /><br>

```php
 public function test_user_can_put_one_post_in_bin(): void
    {
        $animalPostCollection = $this->createTestPosts();
        $animalPost = $animalPostCollection->first();

        $this->softDeletePost($animalPost);

        $this->deleteEverythingAfterTestOperation($animalPostCollection);
    }

    public function test_user_can_put_multiple_posts_in_bin(): void
    {
        $animalPostCollection = $this->createTestPosts(3);

        $this->bulkOperations($animalPostCollection);

        $this->deleteEverythingAfterTestOperation($animalPostCollection);
    }

    public function test_user_can_restore_one_post_in_bin(): void
    {
        $animalPostCollection = $this->createTestPosts();
        $animalPost = $animalPostCollection->first();

        $this->softDeletePost($animalPost);

        $this->bulkOperations($animalPostCollection, BulkOperation::RESTORE);

        $this->deleteEverythingAfterTestOperation($animalPostCollection);
    }
    public function test_user_can_restore_multiple_posts_in_bin(): void
    {
        $animalPostCollection = $this->createTestPosts(3);

        $this->bulkOperations($animalPostCollection);

        $this->bulkOperations($animalPostCollection, BulkOperation::RESTORE);

        $this->deleteEverythingAfterTestOperation($animalPostCollection);
    }

    public function test_user_can_permanently_delete_one_post_in_bin(): void
    {
        $animalPostCollection = $this->createTestPosts();
        $animalPost = $animalPostCollection->first();

        $this->softDeletePost($animalPost);

        $this->bulkOperations($animalPostCollection, BulkOperation::DESTROY);

        $this->deleteEverythingAfterTestOperation($animalPostCollection);
    }

    public function test_user_can_permanently_delete_multiple_posts_in_bin(): void
    {
        $animalPostCollection = $this->createTestPosts(3);

        $this->bulkOperations($animalPostCollection);

        $this->bulkOperations($animalPostCollection, BulkOperation::DESTROY);

        $this->deleteEverythingAfterTestOperation($animalPostCollection);
    }

    public function test_user_cannot_delete_another_users_post(): void
    {
        $animalPostCollection = $this->createTestPosts();
        $otherUser = $this->createTestUser();
        $post = $animalPostCollection->first();

        $response = $this
            ->actingAs($otherUser)
            ->delete(route('animal-post.delete', $post));

        $response->assertForbidden();
        $this->assertDatabaseHas('animal_posts', ['id' => $post->id, 'deleted_at' => null]);

        $this->actingAs($post->user);
        $this->deleteEverythingAfterTestOperation($animalPostCollection);

        $this->actingAs($otherUser);
        $this->deleteEverythingAfterTestOperation(user: $otherUser);

    }
```


**Rastgeleleştirme içeren Factory (+Seed işlemi):**

<img width="1265" height="554" alt="Running_db_seed_operation" src="https://github.com/user-attachments/assets/82752158-955d-4117-a0b4-20ca78c8abdc" /><br>

Proje, aşağıdakileri rastgeleleştiren factory’ler içerir:

- Gönderi başına yüklenen görsel sayısı (2–4) -> min:2 | max:4
- Yorum sayıları ve iç içe geçme derinliği
- Kullanıcı rolleri ve izinleri
- Gönderi durumu ve meta verileri

Bu, hızlı test ve demo verisi üretimi sağlar.


```php
/**
     * Run the database seeds.
     */
    public function run(): void
    {
        $animalPosts = AnimalPost::all();
        $randomAmountOfComments = rand(1, 3);
        foreach ($animalPosts as $post) {
            $rootComments = Comment::factory()->state([
                'commentable_id' => $post->id
            ])
                ->count($randomAmountOfComments)
                ->create();

            foreach ($rootComments as $comment) {
                $this->createNestedComments($comment, $post);
            }
        }

        // $randomReplyDepth = rand(0, 4);

    }

    private function createNestedComments(Comment $parentComment, AnimalPost $post, $depth = 0, $maxDepth = 5)
    {
        if ($depth > $maxDepth) {
            return;
        }

        $randomReplyAmount = rand(0, 3);

        $replies = Comment::factory()->state([
            'commentable_id' => $post->id
        ])->asReply($parentComment)
            ->count($randomReplyAmount)
            ->create();

        // ? AN EMPTY Collection can still be iterated if empty but i wrote 'return' to make the code more explicit
        // ? if the reply count for the parent comment is 0.
        // ? The branch ends and we move on to the next parent comment
        if ($replies->isEmpty()) {
            return;
        }

        // ? CREATING RECURSIVE COMMENTS 
        foreach ($replies as $reply) {
            $this->createNestedComments($reply, $post, $depth + 1, $maxDepth);
        }
    }
```

## Özelliklere Genel Bakış

| Özellik | Uygulama | Portföy Değeri |
|---------|-----------------|-----------------|
| **Kullanıcı Kimlik Doğrulaması** | Oturum/çerez güvenliğiyle Laravel Auth | Güvenlik bilinci |
| **CRUD İşlemleri** | İzinlerle tam oluşturma, okuma, güncelleme ve silme | Uçtan uca geliştirme temelleri |
| **Soft Delete** | Geri yükleme özelliğine sahip çöp kutusu işlevi | Veri bütünlüğü yaklaşımı |
| **Özyinelemeli Yorumlar** | Derinlik kısıtlamalarıyla sınırsız iç içe geçme | Karmaşık veri yapıları |
| **Tam Metin Arama** | Çok alanlı filtrelemeyle Typesense + Docker | Modern altyapı |
| **Redis Önbellekleme** | Observer’lar aracılığıyla otomatik geçersizleştirme | Performans optimizasyonu |
| **Toplu İşlemler** | Birden çok kaydı işlemsel olarak güncelleme | Veritabanı güvenilirliği |
| **Middleware ve İstek Sınırlama** | İstek sınırlama ve role dayalı erişim | Güvenlik ve ölçeklenebilirlik |
| **Mobil Uyumluluk** | Mobil öncelikli tasarımla Tailwind CSS | Platformlar arası yetkinlik |
| **Yönetici Paneli** | Analizler, kullanıcı yönetimi, içerik moderasyonu | Uçtan uca geliştirme derinliği |

## Hata Ayıklama ve Problem Çözme

### Sorun #1: Laravel’de UUID ile İlgili Gizli Oturum Hatası

**Sorun:**

Giriş başarılı görünüyordu, oturum çerezi mevcuttu ve kullanıcı kimliği doğrulanmıştı. Ancak sonraki istekte Laravel kullanıcıyı misafir olarak değerlendiriyordu. Hata yoktu. Görünür bir arıza da yoktu.

**Kök Neden:**

Oturumlar tablosunda `foreignId()` (işaretsiz tamsayı) kullanılırken User modelinde UUID kullanılıyordu. Oturum yazılırken Laravel UUID dizgesini tamsayı sütununa kaydediyor, bu da değerin `NULL` olarak kesilmesine neden oluyordu.

**Düzeltme:**

```php
// Migration fix
$table->foreignUuid('user_id')->nullable()->index();
```

Eski oturumları temizleyip yeniden kimlik doğrulaması yapınca sorun çözülür.

**Bana Öğrettikleri:**

Sistemler, yapılandırma sınırlarında sessizce başarısız olabilir. Yalnızca giriş yanıtını kontrol etmek yeterli değildir; oturum yükünü, veritabanı şemasını, kimlik doğrulama guard’ını, kullanıcı sağlayıcısını ve *sonraki* isteği incelemek gerekir. Bu hata, yalıtılmış bileşenler yerine sistemler bütünü üzerinden düşünmemi sağladı.

## Issue #2: Account Deletion & Comment Soft-Delete Race Condition

**Sorun:**

Bir kullanıcı hesabını sildiğinde, yorumlar soft-delete edilmeli (thread korunmalı). Bunun yerine, `deleted_at` NULL kalırken `user_id` NULL olmuş, en kötü ihtimal ortaya çıkmıştı.

**Kök Neden:**

Database foreign key `onDelete('set null')` Laravel observers'dan *önce* çalışıyor. Observer, `user_id` zaten NULL olduktan sonra tetiklendi, bu nedenle `Comment::where('user_id', $user->id)->get()` sıfır sonuç döndürdü.

**Çözüm:**

`deleted()` yerine `deleting()` kullan. `deleting()` olayı silme *öncesinde* tetiklenerek `user_id` referansını korur:

```php
  public function deleting(User $user): void
    {
        // ? WE ARE SOFT DELETING COMMENTS OF USER WHEN DELETION OCCURS. SO THAT THE COMMENTS DONT GET DELETED
        DB::transaction(function () use ($user) {
            // ? SOFT DELETE USER'S COMMENTS ON EVERY POST
            $comments = Comment::where('user_id', $user->id)->get();
            $posts = AnimalPost::where('user_id', $user->id)->get();
            foreach ($comments as $comment) {
                Cache::tags(["post_comments_{$comment->commentable_id}"])
                    ->flush();
                $comment->delete();
            }

            foreach ($posts as $post) {
                $post->unsearchable();
                $post->forceDelete();
            }
        });
    }
```

**Beni Öğreten Ders:**

Olay sırası önemlidir. Database kısıtlamaları ve uygulama mantığı çatışabilir. İşlemlerin yaşam döngüsünü anlamak (deleting vs. deleted, creating vs. created) doğru davranış için kritiktir.

---

## Teknoloji Yığını

| Katman | Teknoloji | Neden |
|-------|-----------|-----|
| **Backend** | Laravel 13 | Endüstri standardı PHP framework'ü zengin ekosistemle |
| **Frontend** | JavaScript + Tailwind CSS | Responsive, bakımı kolay styling + API Alma |
| **Database** | MySQL | İşlemler için ACID uyumluluğu |
| **Arama** | Typesense + Docker | Production-grade tam metin arama |
| **Caching** | Redis + Docker | Tag aracılığıyla hızlı cache geçersiz kılma |
| **Testing** | Laravel Tests (PHPUnit) | Kritik yollar için kapsam |
| **Containerization** | Docker | Tutarlı yerel ortamlar |

---

## Mobil UI

<table class="w-full">
  <tr>
    <td><img src="https://github.com/user-attachments/assets/95aa5373-6ea1-4e33-bfa3-c40407592101" alt="Petfinder stats filter" width="300"></td>
    <td><img src="https://github.com/user-attachments/assets/8565156e-d56a-42ee-a57a-528c0863a2c9" alt="Petfinder stats filter archive" width="300"></td>
    <td><img src="https://github.com/user-attachments/assets/33fb0d3a-ee38-4910-a858-d119407cb3a9" alt="Petfinder recursive comments" width="300"></td>
    <td><img src="https://github.com/user-attachments/assets/393de14a-8fe4-464f-8560-65136ab7dbbb" alt="Petfinder query" width="300"></td>
  </tr>
  <tr>
    <td><img src="https://github.com/user-attachments/assets/dbf67d5d-0694-4154-9820-1da0256899fc" alt="Petfinder post view" width="300"></td>
    <td><img src="https://github.com/user-attachments/assets/84ee742f-6b5b-41eb-9897-a97c0c08a4cc" alt="Petfinder navigation" width="300"></td>
    <td><img src="https://github.com/user-attachments/assets/e8cc556c-de46-46dd-86d1-349d01c89cf0" alt="Petfinder bulk soft deletion" width="300"></td>
    <td><img src="https://github.com/user-attachments/assets/df23bcba-c0cc-4254-a60b-67125fe1f3a8" alt="Petfinder archive" width="300"></td>
  </tr>
</table><br>

Uygulama tamamen responsive'dir, etkileşimler ve küçük ekranlar için optimize edilmiştir. Mobil'in temel özellikleri şunları içerir:

- Daraltılabilir navigasyon ve yorum threadleri
- Mobil kısıtlamaları için optimize edilmiş arama filtreleri
- Görüntü optimizasyonuyla hızlı yükleme

---

## Temel Öğrenme Çıktıları

**İlk Kez Yapılan Uygulamalar:**

- Derinlik kısıtlamalarıyla özyinelemeli yorum sistemi
- Docker ile Typesense tam metin arama
- İşlemsel güvenlikle toplu işlemler
- Otomatik senkronizasyon için Observer deseni
- Otomatik geçersiz kılmayla Redis caching
- Test kapsamı
- Soft silme desenleri
- Tür güvenliği için Enum kullanımı
- Middleware ve istek throttling

**Geliştirme Zaman Çizelgesi:** 6 hafta, solo geliştirici  
**Başlangıçtaki Deneyim Seviyesi:** 4 ay Laravel, toplam 1 yıldan az kodlama  
**Sonuç:** Şablonlardan sıfır kod yeniden kullanımıyla production-grade mimari

---

## Bu Proje Benim Kariyer İçin Neden Önemli

Bu proje şunları gösterir:

1. **Hızlı Öğrenme:** 1 yıldan az kodlama deneyimiyle 6 hafta içinde karmaşık full-stack bir uygulama yayınlandı
2. **Full-Stack Yetkinliği:** Backend mimarisi (observers, işlemler, caching) + frontend (responsive UI, mobil optimizasyonu)
3. **Production Düşüncesi:** Enterprise araçlar seçildi (Typesense, Redis, Docker) ve ilk denemede uygulandı
4. **Problem Çözme:** İnce ısmarlı race conditions ve mimari uyuşmazlıklar bağımsız olarak debuglandı
5. **Kod Kalitesi:** Kapsamlı testler, factories ve migrations, hack proje değil
6. **Solo İcra:** Her özellik rehberlik olmadan oluşturuldu, test edildi ve debuglandı

---

## Sonrasında Ne Var

Planlanan iyileştirmeler:

**Proje daha ileri götürülürse:**
- Özyinelemeli yorum sistemi üzerinde iyileştirme. API isteği aracılığıyla on-demand yükleme ekleme.
- Görüntü optimizasyonları
- WebSockets ile gerçek zamanlı bildirimler
- Kayıp hayvan konumları için harita entegrasyonu
- Güven oluşturmak için kullanıcı derecelendirme sistemi
- Yeni eşleşmeler için e-posta bildirimleri
