# mysql_replication_on_same_server

Eğitimler ve makalelerin bir çoğunda replikasyon ile ilgili farklı sunucular arası master-slave konfigürasyon kaynaklar bolca bulunmaktadır, fakat bazı durumlarda planlama ihtiyaçlarına göre aynı MySQL server üzerinde instance’lar arası Databaseleri replica yapma senaryosu ile baş başa kalındığı zaman pek kaynağa ulaşılamıyor, varolanlar ise sürüm olarak güncelliğini yitirmiş olduğunu fark ettim. Bu senaryomu başarılı bir şekilde gerçekleştirdikten sonra yazıya dökmek istedim.

CentOS 7 üzerinde direk olarak “yum install mysql” komutunu yazarsanız, CentOS size MySQL yerine MariaDB yükleyecektir. Bunun önüne geçmek için aşağıdaki aşamaları uygulamanız gerekmetedir.

MySQL resmi web sayfasına giderek istediğiniz sürümdeki rpm’i sunucunuza indirin ve yükleyin

https://dev.mysql.com/downloads/repo/yum/ üzerinden ilgili sürümü belirleyin ve linkini kopyalayın.

wget https://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm
sudo rpm -ivh mysql57-community-release-el7-9.noarch.rpm


MySQL Kurulumu:
Kurulumu başlatmak için

```sudo yum install mysql-server```
Servis durumunu görüntülemek için aşağıdaki komutları çalıştırın ve Active: active (running) yazısını görün.

```
sudo systemctl start mysqld
sudo systemctl status mysqld
```

MySQL kurulumu sırasında root hesabı için geçici bir şifre üretilecektir. Bu şifreye ulaşabilmek için
```
sudo grep 'temporary password' /var/log/mysqld.log
```

Örnek sonuç:

[Note] A temporary password is generated for root@localhost: mqRsFsJd_3Xk>r


Default root parolasını değiştirmeniz için
```
sudo mysql_secure_installation
```

MySQL sunucuna bağlanmak için:
```
mysqladmin -u root -p version
```

Buraya kadar herhangi bir hata almazsanız herşey yolunda demektir. MySQL sunucusu başarılı bir şekilde kuruldu.


MySQL Database’ler Arası Replikasyon

MySQL eski sürümlerinde replikasyonun konfigürasyon aşamalarından biri mysqld_multi komutuydu fakat yeni sürümü ile birlikte bu komut kullanım maksadı kalktı çünkü MySQL o işlemleri kendisi yapabilir hale geldi nasıl olduğundan bahsedeceğim. Bunu özlellikle belirtiyorum çünkü araştırdığınız zaman mysqld_multi işlemleri yapmanız gerektiğini eski makalelerde göreceksiniz vakit kaybına sebep olmasın.


Replikasyon sürecini başlatmak için en önemli nokta /etc/my.cnf dosyasının konfigürasyonudur. Burada ilk bakıldığında default instance vardır ve port 3306’dır, port önemli çünkü aynı sunucu üzerinde çalıştığımız için 127.0.0.1 sunucunsundaki hangi instance’a bağlanacağımızı port aracılığı ile belirteceğiz.

My.cnf dosyasını vi veya nano ile editleyip içerisinde default haricinde instance’lar oluşturacağız ve bunları master-slave olarak replikasyonunu sağlayacağız.

Instance oluşturmak için [example] ile başlayacağız ve gerekli parametrelerin yerlerini belirteceğiz.

Örneğin aşağidaki konf default’a aittir :

```
[mysqld]
port=3306
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mysql.log
pid-file=/var/run/mysqld/mysqld.pid
```

Yeni sürüm ile birlikte [example@replica01] şeklinde instance oluşturursak mysql_multi işlemlerini kendisi halledecek ve instance’ı ayağı kaldıracaktır.

Port gibi önemli noktalardan bir diğeri ise server-id, replikasyon aşamasında istenen parametrelerden bir tanesidir. Bu ID’ler uniq yani birbirinden farklı olması gerekir.

O zaman Master instance’ımızı kuralım.

```
[mysqld@replica01]
server-id = 1
port=3307
datadir=/var/lib/mysql-replica01
socket=/var/lib/mysql-replica01/mysql.sock
symbolic-links=0
log-error=/var/log/mysqld-replica01.log
pid-file=/var/run/mysqld-replica01/mysqld.pid
log-bin=mysql-bin
innodb_flush_log_at_trx_commit=1
sync_binlog=1
binlog-format=ROW
Şimdi ise Slave instance kuralım.

[mysqld@replica02]
server-id = 2
port=3308
datadir=/var/lib/mysql_slave
socket=/var/lib/mysql-replica02/mysql.sock
symbolic-links=0
log-bin=mysql-bin
log-error=/var/log/mysqld-replica02.log
pid-file=/var/run/mysqld-replica02/mysqld.pid
wait_timeout = 28800
interactive_timeout = 28800
max_allowed_packet  = 256M
```

Her iki instance ile ilgili config ayarlarını yaptıktan sonra systemctl restart mysqld komutunu çalıştırın.

```
systemctl status mysqld@replica01
systemctl status mysqld@replica02
```

komutları ile sağlık durumlarını kontrol edin.

Bu aşamadan sonra instance’larımıza ayrı ayrı bağlanıp işlemlerimizi yapacağız.

İlk olarak mysqld@replica01 için bağlanıyoruz
```
mysql -u root -p --host=127.0.0.1 --port=3307
```

Replication için yeni bir kullanıcı oluştur

```
mysql> CREATE USER 'replication'@'%' IDENTIFIED BY 'replication';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
```

Instance01’de çık

Master Instance’ının dump dosyasını oluştur

```
mysqldump -u root -p --host=127.0.0.1 --port=3307 --all-databases --master-data=2 > replicationdump.sql
```

Oluşturduğumuz master dump dosyasını slave üzerine import et

```
mysql -u root -p --host=127.0.0.1 --port=3308 < replicationdump.sql
```

Slave Instace’a bağlan

```
mysql -u root -p --host=127.0.0.1 --port=3308
```

Aşağıdaki Master bilgilerini gir

```
mysql> CHANGE MASTER TO
-> MASTER_HOST='127.0.0.1',
-> MASTER_USER='replication',
-> MASTER_PASSWORD='replication',
-> MASTER_PORT='3307',
-> MASTER_LOG_FILE='mysql-bin.000001',
-> MASTER_LOG_POS=349;
```

Replikasyonu başlatmak için

```
mysql> START SLAVE;
```

Replikasyonu kontrol etmek için

```
mysql> SHOW SLAVE STATUS \G
```

Bu sayede aynı MySQL sunucumuz üzerinde farklı instance’lar oluştururarak replikasyonu başlatmış oluyoruz. Master içerisindeki tüm DB hareketleri anında Slave üzerinde de oluştuğunu görebilirsiniz.

Master üzerinde aşağıdaki komutu çalıştırıp

```
Mysql> Create database test;
```

Slave üzerinde aynı database’in oluştuğunu görebilirsiniz.

```
Mysql> Show databases;
```

Ayrıca MySQL Workbench ile de Database yönetiminizi görsel arayüz ile de sağlayabilirsiniz.
