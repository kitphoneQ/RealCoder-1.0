
import sys
import ast
import io
import random
import time
import subprocess
from contextlib import redirect_stdout
from pathlib import Path
from PySide6.QtCore import QMimeData
from PySide6.QtGui import QAction, QDrag, QKeySequence
from PySide6.QtGui import QColor, Qt
from PySide6.QtWidgets import (
    QApplication,
    QMainWindow,
    QGraphicsView,
    QGraphicsScene,
    QGraphicsItem,
    QGraphicsProxyWidget,
    QLineEdit,
    QPushButton,
    QPlainTextEdit,
    QVBoxLayout,
    QHBoxLayout,
    QWidget,
    QLabel,
    QListWidget,
    QAbstractItemView,
    QInputDialog,
    QFileDialog,
    QMessageBox,
    QToolBar,
)




class RealCoderMotoru:
    """RealCoder bloklarını Python koduna dönüştürür ve çalıştırır."""

    DEGER_BLOKLARI = {
        "sayi", "metin", "dogru_yanlis", "toplama", "cikarma",
        "carpma", "bolme", "kalan", "rastgele", "sayi_donustur",
        "metin_donustur", "uzunluk", "buyuk_harf", "kucuk_harf",
        "metin_birlestir", "esit_mi", "buyuk_mu", "kucuk_mu",
        "ve", "veya", "degil", "liste", "liste_uzunluk",
        "listeden_al", "dosya_var_mi"
    }

    def __init__(self, bloklar):
        self.bloklar = list(bloklar)
        self.hatalar = []
        self._aktif = set()
        self.tanimli_degiskenler = set()

    def ifade(self, metin):
        """Python ifadesini koru; tırnaksız tek kelimeyi metin kabul et."""
        metin = metin.strip()
        if not metin:
            return "None"
        try:
            ifade_agaci = ast.parse(metin, mode="eval").body
        except SyntaxError:
            return repr(metin)
        if isinstance(ifade_agaci, ast.Name) and ifade_agaci.id not in self.tanimli_degiskenler:
            return repr(metin)
        return metin

    @staticmethod
    def degisken_adi(metin):
        metin = metin.strip()

        if not metin.isidentifier() or metin in {
            "if", "else", "for", "while", "def", "return",
            "import", "True", "False", "None", "break", "continue"
        }:
            raise ValueError(f"Geçersiz değişken veya fonksiyon adı: {metin}")

        return metin

    def deger_kodu(self, blok):
        tur = blok.tur
        metin = blok.metin_alani.text().strip()

        if tur == "degisken":
            alan = getattr(blok, "degisken_adi_alani", None)
            if alan is None:
                raise ValueError("Değişken bloğunda isim alanı bulunamadı.")
            return self.degisken_adi(alan.text())

        if tur == "sayi":
            try:
                deger = ast.literal_eval(metin)
            except (ValueError, SyntaxError):
                raise ValueError(f"Geçersiz sayı: {metin}")

            if isinstance(deger, bool) or not isinstance(deger, (int, float)):
                raise ValueError(f"Geçersiz sayı: {metin}")

            return repr(deger)

        if tur == "metin":
            return repr(metin)

        if tur == "dogru_yanlis":
            if metin.casefold() == "true":
                return "True"
            if metin.casefold() == "false":
                return "False"
            raise ValueError("Değer True veya False olmalı.")

        ikili_islemler = {
            "toplama": "+",
            "cikarma": "-",
            "carpma": "*",
            "bolme": "/",
            "kalan": "%",
            "esit_mi": "==",
            "buyuk_mu": ">",
            "kucuk_mu": "<",
            "ve": "and",
            "veya": "or",
        }

        if tur in ikili_islemler:
            # Giriş alanı örneği: x > 25 veya 5 + 3
            self._ifade_dogrula(metin)
            return f"({metin})"

        if tur == "degil":
            self._ifade_dogrula(metin)
            return f"(not ({metin}))"

        if tur == "rastgele":
            parcalar = [p.strip() for p in metin.split(",")]
            if len(parcalar) != 2:
                raise ValueError("Rastgele sayı için alt, üst sınır gir.")
            return f"random.randint(int({parcalar[0]}), int({parcalar[1]}))"

        if tur == "sayi_donustur":
            return f"int({self.ifade(metin)})"

        if tur == "metin_donustur":
            return f"str({self.ifade(metin)})"

        if tur == "uzunluk":
            return f"len({self.ifade(metin)})"

        if tur == "buyuk_harf":
            return f"str({self.ifade(metin)}).upper()"

        if tur == "kucuk_harf":
            return f"str({self.ifade(metin)}).lower()"

        if tur == "metin_birlestir":
            return f"str({self.ifade(metin)})"

        if tur == "liste":
            # Virgülle ayrılmış elemanları listeye dönüştür.
            elemanlar = []
            for parca in metin.split(","):
                try:
                    elemanlar.append(ast.literal_eval(parca.strip()))
                except (ValueError, SyntaxError):
                    elemanlar.append(parca.strip())
            return repr(elemanlar)

        if tur == "liste_uzunluk":
            return f"len({self.degisken_adi(metin)})"

        if tur == "listeden_al":
            return f"liste[int({metin})]"

        if tur == "dosya_var_mi":
            return f"Path({metin!r}).is_file()"

        raise ValueError(f"{tur} bir değer bloğu olarak kullanılamıyor.")

    @staticmethod
    def _ifade_dogrula(metin):
        try:
            ast.parse(metin, mode="eval")
        except SyntaxError as exc:
            raise ValueError(f"Geçersiz ifade: {metin}") from exc

    def blok_kodu(self, blok, girinti=0):
        tur = blok.tur
        metin = blok.metin_alani.text().strip()
        bosluk = "    " * girinti

        if tur == "liste":
            return [bosluk + f"liste = {self.deger_kodu(blok)}"]

        if tur == "yazdir":
            return [bosluk + f"print({self.ifade(metin)})"]

        if tur == "kullanici_girdisi":
            ad = self.degisken_adi(metin)
            self.tanimli_degiskenler.add(ad)
            return [bosluk + f"{ad} = input('Veri gir: ')"]

        if tur == "yorum":
            return [bosluk + "# " + metin]

        if tur == "hata":
            return [bosluk + f"raise Exception({metin!r})"]

        if tur == "degisken":
            alan = getattr(blok, "degisken_adi_alani", None)
            if alan is None:
                raise ValueError("Değişken bloğunda isim alanı yok.")
            ad = self.degisken_adi(alan.text())
            deger = self.ifade(metin)
            self.tanimli_degiskenler.add(ad)
            return [bosluk + f"{ad} = {deger}"]

        if tur == "liste":
            return [bosluk + f"liste = {self.deger_kodu(blok)}"]

        if tur == "liste_ekle":
            return [bosluk + f"liste.append({self.ifade(metin)})"]

        if tur == "kosul":
            self._ifade_dogrula(metin)
            return [bosluk + f"if {metin}:"]

        if tur == "degilse":
            return [bosluk + "else:"]

        if tur in {"dongu", "tekrar", "tekrar_sayisi"}:
            return [bosluk + f"for _ in range(int({metin})):"]

        if tur == "while":
            self._ifade_dogrula(metin)
            return [bosluk + f"while {metin}:"]

        if tur == "durdur":
            return [bosluk + "break"]

        if tur == "atla":
            return [bosluk + "continue"]

        if tur == "fonksiyon":
            ad = self.degisken_adi(metin)
            return [bosluk + f"def {ad}():"]

        if tur == "fonksiyon_cagir":
            ad = self.degisken_adi(metin)
            return [bosluk + f"{ad}()"]

        if tur == "parametre":
            return [bosluk + "# Parametre: " + metin]

        if tur == "geri_dondur":
            return [bosluk + f"return {self.ifade(metin)}"]

        if tur == "bekle":
            return [bosluk + f"time.sleep(max(0, float({metin})))"]

        if tur == "dosyaya_yaz":
            return [
                bosluk + f"Path({metin!r}).write_text('', encoding='utf-8')"
            ]

        if tur == "dosyadan_oku":
            return [
                bosluk + f"print(Path({metin!r}).read_text(encoding='utf-8'))"
            ]

        # Değer blokları tek başına duruyorsa değerlerini çıktıda göster.
        if tur in self.DEGER_BLOKLARI:
            return [bosluk + f"print({self.deger_kodu(blok)})"]

        raise ValueError(f"Bilinmeyen blok türü: {tur}")

    def _blok_sirasi(self):
        """Önce yönlü bağlantıları izle; bağlantısız blokları da kaybetme."""
        bilinen = set(self.bloklar)
        gelen = set()

        for blok in self.bloklar:
            sonraki = getattr(blok, "sonraki", None)
            if sonraki in bilinen:
                gelen.add(sonraki)

        kokler = [b for b in self.bloklar if b not in gelen]
        kokler.sort(key=lambda b: (b.pos().y(), b.pos().x()))

        sonuc = []
        gorulen = set()

        def izle(blok):
            while blok is not None and blok in bilinen and blok not in gorulen:
                gorulen.add(blok)
                sonuc.append(blok)
                blok = getattr(blok, "sonraki", None)

        for kok in kokler:
            izle(kok)

        # Henüz bağlantısı olmayan bloklar da görünür kalsın.
        kalan = [b for b in self.bloklar if b not in gorulen]
        kalan.sort(key=lambda b: (b.pos().y(), b.pos().x()))
        for blok in kalan:
            izle(blok)

        return sonuc

    def kod_uret(self):
        satirlar = []
        self.hatalar.clear()
        self.tanimli_degiskenler.clear()
        sirali_bloklar = self._blok_sirasi()
        sonraki_girinti = 0
        govde_bekleniyor = False

        govde_basliklari = {"kosul", "degilse", "dongu", "tekrar", "tekrar_sayisi", "while", "fonksiyon"}

        for blok in sirali_bloklar:
            # Bir başlıktan sonra gelen ilk bağlı blok gövde olur.
            girinti = sonraki_girinti
            if govde_bekleniyor and blok.tur == "degilse":
                satirlar.append("    " * girinti + "pass")
                girinti = max(0, girinti - 1)
            elif not govde_bekleniyor:
                girinti = 0

            try:
                satirlar.extend(self.blok_kodu(blok, girinti))
            except Exception as exc:
                self.hatalar.append(f"{blok.baslik}: {exc}")

            govde_bekleniyor = blok.tur in govde_basliklari
            sonraki_girinti = girinti + 1 if govde_bekleniyor else 0

        if govde_bekleniyor:
            satirlar.append("    " * sonraki_girinti + "pass")

        if self.hatalar:
            raise ValueError("\n".join(self.hatalar))

        return "\n".join(satirlar) or "pass"

    def calistir(self):
        kod = self.kod_uret()
        derlenmis = compile(kod, "<RealCoder>", "exec")

        ortam = {
            "__builtins__": __builtins__,
            "random": random,
            "time": time,
            "Path": Path,
        }

        exec(derlenmis, ortam, ortam)
        return kod


class KodDizini(QGraphicsItem):
    DIZINLER = {
        # TURUNCU: Çıktı ve giriş
        "yazdir": ("Ekrana yazdır", "#f0b429", "Merhaba Dünya!", "Yazdırılacak metin..."),
        "kullanici_girdisi": ("Kullanıcıdan veri al", "#f0b429", "İsim", "Değişken adı..."),
        "yorum": ("Açıklama ekle", "#f0b429", "Bu bir açıklama", "Yorum yaz..."),
        "hata": ("Hata mesajı", "#f0b429", "Bir hata oluştu", "Hata metni..."),

        # MAVİ: Değişkenler ve veri
        "degisken": ("Değişken oluştur", "#3498db", "10", "Değişken değeri..."),
        "sayi": ("Sayı oluştur", "#3498db", "42", "Bir sayı yaz..."),
        "metin": ("Metin oluştur", "#3498db", "Merhaba", "Metin yaz..."),
        "dogru_yanlis": ("Doğru / Yanlış", "#3498db", "True", "True veya False"),
        "liste": ("Liste oluştur", "#3498db", "1, 2, 3", "Liste elemanları..."),
        "liste_ekle": ("Listeye eleman ekle", "#3498db", "4", "Eklenecek eleman..."),
        "listeden_al": ("Listeden eleman al", "#3498db", "0", "Eleman indeksi..."),
        "liste_uzunluk": ("Liste uzunluğu", "#3498db", "liste", "Liste adı..."),

        # AÇIK MAVİ: Matematik
        "toplama": ("Toplama", "#16a6d9", "5 + 3", "İşlem..."),
        "cikarma": ("Çıkarma", "#16a6d9", "5 - 3", "İşlem..."),
        "carpma": ("Çarpma", "#16a6d9", "5 * 3", "İşlem..."),
        "bolme": ("Bölme", "#16a6d9", "10 / 2", "İşlem..."),
        "kalan": ("Bölümden kalan", "#16a6d9", "10 % 3", "İşlem..."),
        "rastgele": ("Rastgele sayı", "#16a6d9", "1, 100", "Alt ve üst sınır..."),
        "sayi_donustur": ("Sayıya dönüştür", "#16a6d9", "12", "Dönüştürülecek değer..."),

        # PEMBE: Metin işlemleri
        "metin_donustur": ("Metne dönüştür", "#e84393", "123", "Değer..."),
        "uzunluk": ("Uzunluğu bul", "#e84393", "Merhaba", "Metin veya liste..."),
        "buyuk_harf": ("Büyük harfe çevir", "#e84393", "merhaba", "Metin..."),
        "kucuk_harf": ("Küçük harfe çevir", "#e84393", "MERHABA", "Metin..."),
        "metin_birlestir": ("Metinleri birleştir", "#e84393", "Merhaba Dünya", "Birleştirilecek metin..."),

        # MOR: Koşullar ve karşılaştırmalar
        "kosul": ("Eğer", "#9b59b6", "10 > 5", "Koşul ifadesi..."),
        "degilse": ("Değilse", "#9b59b6", "aksi durum", "Alternatif durum..."),
        "esit_mi": ("Eşit mi?", "#9b59b6", "5 == 5", "Karşılaştırma..."),
        "buyuk_mu": ("Büyük mü?", "#9b59b6", "10 > 5", "Karşılaştırma..."),
        "kucuk_mu": ("Küçük mü?", "#9b59b6", "3 < 8", "Karşılaştırma..."),
        "ve": ("Ve", "#9b59b6", "True and False", "İki koşul..."),
        "veya": ("Veya", "#9b59b6", "True or False", "İki koşul..."),
        "degil": ("Değil", "#9b59b6", "not True", "Koşul..."),

        # YEŞİL: Döngüler
        "dongu": ("Döngü", "#2ecc71", "5", "Tekrar sayısı..."),
        "tekrar": ("Tekrar et", "#2ecc71", "5", "Tekrar sayısı..."),
        "tekrar_sayisi": ("Belirli sayıda tekrarla", "#2ecc71", "10", "Tekrar sayısı..."),
        "while": ("Koşul doğruyken tekrarla", "#2ecc71", "x < 10", "Döngü koşulu..."),
        "durdur": ("Döngüyü durdur", "#2ecc71", "", "Döngüden çık"),
        "atla": ("Sonraki tekrara geç", "#2ecc71", "", "Sonraki adıma geç"),

        # PEMBE-MOR: Fonksiyonlar
        "fonksiyon": ("Fonksiyon tanımla", "#d66efd", "selamla", "Fonksiyon adı..."),
        "fonksiyon_cagir": ("Fonksiyonu çağır", "#d66efd", "selamla", "Fonksiyon adı..."),
        "parametre": ("Parametre oluştur", "#d66efd", "isim", "Parametre adı..."),
        "geri_dondur": ("Değer döndür", "#d66efd", "sonuc", "Döndürülecek değer..."),

        # GRİ: Zaman ve dosya işlemleri
        "bekle": ("Bekle", "#7f8c8d", "1", "Saniye..."),
        "dosyaya_yaz": ("Dosyaya yaz", "#7f8c8d", "veri.txt", "Dosya adı..."),
        "dosyadan_oku": ("Dosyadan oku", "#7f8c8d", "veri.txt", "Dosya adı..."),
        "dosya_var_mi": ("Dosya mevcut mu?", "#7f8c8d", "veri.txt", "Dosya adı...")
    }

    def __init__(self, x, y, tur="yazdir"):
        super().__init__()

        from PySide6.QtCore import QRectF

        self.setPos(x, y)
        self.setFlags(
    QGraphicsItem.GraphicsItemFlag.ItemIsMovable
    | QGraphicsItem.GraphicsItemFlag.ItemIsSelectable
    | QGraphicsItem.GraphicsItemFlag.ItemSendsGeometryChanges
)

        self.tur = tur
        self.bagli_dizinler = []
        self._build_tasiniyor = False

        # Gerçek program bağlantıları
        self.sonraki = None
        self.govde = []
        self.else_govde = []
        self.girdiler = {}

        bilgi = self.DIZINLER.get(tur, self.DIZINLER["yazdir"])
        self.baslik, self.renk, varsayilan, ipucu = bilgi

        yukseklik = 115 if tur == "degisken" else 85
        self._rect = QRectF(0, 0, 250, yukseklik)

        self.metin_alani = QLineEdit()
        self.metin_alani.setPlaceholderText(ipucu)
        self.metin_alani.setText(varsayilan)
        self.metin_alani.setFixedWidth(220)

        self.metin_proxy = QGraphicsProxyWidget(self)
        self.metin_proxy.setWidget(self.metin_alani)
        self.metin_proxy.setPos(10, 45)

        # Değişken için ayrı isim alanı
        self.degisken_adi_alani = None
        self.degisken_adi_proxy = None

        if tur == "degisken":
            self.degisken_adi_alani = QLineEdit()
            self.degisken_adi_alani.setPlaceholderText("Değişken adı")
            self.degisken_adi_alani.setText("sayi")
            self.degisken_adi_alani.setFixedWidth(220)

            self.degisken_adi_proxy = QGraphicsProxyWidget(self)
            self.degisken_adi_proxy.setWidget(
                self.degisken_adi_alani
            )
            self.degisken_adi_proxy.setPos(10, 72)

    def boundingRect(self):
        return self._rect

    
    
    def itemChange(self, change, value):
        if (
            change == QGraphicsItem.GraphicsItemChange.ItemPositionChange
            and not self._build_tasiniyor
        ):
            fark = value - self.pos()

            if fark.x() != 0 or fark.y() != 0:
                grup = []
                ziyaret_edilen = set()
                bekleyen = [self]

                while bekleyen:
                    oge = bekleyen.pop()

                    if oge in ziyaret_edilen:
                        continue

                    ziyaret_edilen.add(oge)
                    grup.append(oge)
                    bekleyen.extend(oge.bagli_dizinler)

                for oge in grup:
                    oge._build_tasiniyor = True

                try:
                    for oge in grup:
                        if oge is not self:
                            oge.setPos(oge.pos() + fark)
                finally:
                    for oge in grup:
                        oge._build_tasiniyor = False

        return super().itemChange(change, value)

    def build_uyesini_ekle(self, dizin):
        if dizin is self:
            return

        # Görsel grubu birlikte taşıma bağlantıları
        if dizin not in self.bagli_dizinler:
            self.bagli_dizinler.append(dizin)

        if self not in dizin.bagli_dizinler:
            dizin.bagli_dizinler.append(self)

        # Programın yönlü akışı
        self.sonraki = dizin

    def mousePressEvent(self, event):
        self._undo_baslangic = None
        if self.scene():
            for gorunum in self.scene().views():
                ana = getattr(gorunum, "ana_pencere", None)
                if ana:
                    self._undo_baslangic = ana.durum_kaydet()
                    break
        super().mousePressEvent(event)

    def mouseReleaseEvent(self, event):
        super().mouseReleaseEvent(event)

        if self.scene():
            for gorunum in self.scene().views():
                if hasattr(gorunum, "dizini_bagla"):
                    gorunum.dizini_bagla(self)
                    ana = getattr(gorunum, "ana_pencere", None)
                    if ana and self._undo_baslangic is not None:
                        ana.islemi_gecmise_ekle(self._undo_baslangic)
                    break

    def python_ifadesi(self):
        metin = self.metin_alani.text().strip()

        if self.tur == "degisken":
            return self.degisken_adi_alani.text().strip()

        if self.tur == "sayi":
            return str(float(metin)) if "." in metin else str(int(metin))

        if self.tur == "metin":
            return repr(metin)

        if self.tur == "dogru_yanlis":
            deger = metin.casefold()

            if deger == "true":
                return "True"
            if deger == "false":
                return "False"

            raise ValueError("Doğru/Yanlış değeri True veya False olmalı.")

        ifadeler = {
            "toplama": lambda: f"({metin})",
            "cikarma": lambda: f"({metin})",
            "carpma": lambda: f"({metin})",
            "bolme": lambda: f"({metin})",
            "kalan": lambda: f"({metin})",
            "esit_mi": lambda: f"({metin})",
            "buyuk_mu": lambda: f"({metin})",
            "kucuk_mu": lambda: f"({metin})",
            "ve": lambda: f"({metin})",
            "veya": lambda: f"({metin})",
            "degil": lambda: f"({metin})",
            "sayi_donustur": lambda: f"int({metin})",
            "metin_donustur": lambda: f"str({metin})",
            "uzunluk": lambda: f"len({metin})",
            "buyuk_harf": lambda: f"str({metin}).upper()",
            "kucuk_harf": lambda: f"str({metin}).lower()",
            "metin_birlestir": lambda: f"str({metin})",
            "rastgele": lambda: f"__rastgele({metin})",
            "liste_uzunluk": lambda: f"len({metin})",
            "listeden_al": lambda: f"liste[{metin}]",
            "dosya_var_mi": lambda: f"__dosya_var_mi({metin})",
        }

        if self.tur in ifadeler:
            return ifadeler[self.tur]()

        return metin

    def python_satiri(self):
        metin = self.metin_alani.text().strip()

        if self.tur == "degisken":
            ad = self.degisken_adi_alani.text().strip()

            if not ad.isidentifier() or ad in {
                "if", "else", "while", "for", "def",
                "class", "return", "import", "True", "False",
                "None", "break", "continue", "pass"
            }:
                raise ValueError(f"Geçersiz değişken adı: {ad}")

            return f"{ad} = {metin or 'None'}"

        if self.tur == "yazdir":
            return f"print({metin!r})"

        if self.tur == "kullanici_girdisi":
            if not metin.isidentifier():
                raise ValueError("Girdi için geçerli bir değişken adı yaz.")

            return f"{metin} = input('Veri gir: ')"

        if self.tur == "yorum":
            return f"# {metin}"

        if self.tur == "hata":
            return f"raise Exception({metin!r})"

        if self.tur == "liste":
            elemanlar = [
                parca.strip()
                for parca in metin.split(",")
            ]
            return f"liste = {elemanlar!r}"

        if self.tur == "liste_ekle":
            return f"liste.append({metin})"

        if self.tur == "kosul":
            return f"if {metin}:"

        if self.tur == "degilse":
            return "else:"

        if self.tur in {"dongu", "tekrar", "tekrar_sayisi"}:
            return f"for _ in range(int({metin})):"

        if self.tur == "while":
            return f"while {metin}:"

        if self.tur == "durdur":
            return "break"

        if self.tur == "atla":
            return "continue"

        if self.tur == "fonksiyon":
            if not metin.isidentifier():
                raise ValueError("Geçerli bir fonksiyon adı yaz.")

            return f"def {metin}():"

        if self.tur == "fonksiyon_cagir":
            if not metin.isidentifier():
                raise ValueError("Geçerli bir fonksiyon adı yaz.")

            return f"{metin}()"

        if self.tur == "parametre":
            return metin

        if self.tur == "geri_dondur":
            return f"return {metin}"

        if self.tur == "bekle":
            return f"__bekle({metin})"

        if self.tur == "dosyaya_yaz":
            return f"__dosyaya_yaz({metin!r})"

        if self.tur == "dosyadan_oku":
            return f"__dosyadan_oku({metin!r})"

        # Diğer veri blokları bir ifade olarak kullanılabilir.
        return self.python_ifades()

    def paint(self, painter, option, widget=None):
        painter.setBrush(QColor(self.renk))
        painter.setPen(QColor("#202020"))
        painter.drawRoundedRect(self._rect, 10, 10)

        painter.setPen(QColor("#202020"))
        painter.drawText(10, 28, self.baslik)

        if self.isSelected():
            painter.setPen(QColor("#ffffff"))
            painter.drawRoundedRect(self._rect, 10, 10)




class CalismaAlani(QGraphicsView):
    def __init__(self, sahne):
        super().__init__(sahne)
        self.setAcceptDrops(True)
        self.viewport().setAcceptDrops(True)
        self.setFocusPolicy(Qt.FocusPolicy.StrongFocus)
        self.ana_pencere = None
        self.kopya_bloklar = None
        self.yapistirma_sayisi = 0

        self.izgara_araligi = 25

        # kaydırma
        self.setDragMode(QGraphicsView.DragMode.NoDrag)
        self.setTransformationAnchor(
            QGraphicsView.ViewportAnchor.AnchorUnderMouse
        )

        # Kaydırma çubuklarını gizle
        self.setHorizontalScrollBarPolicy(
            Qt.ScrollBarPolicy.ScrollBarAlwaysOff
        )
        self.setVerticalScrollBarPolicy(
            Qt.ScrollBarPolicy.ScrollBarAlwaysOff
        )

        self.setBackgroundBrush(QColor("#252525"))

    def keyPressEvent(self, event):
        odak = QApplication.focusWidget()
        metin_alani_odakta = isinstance(odak, (QLineEdit, QPlainTextEdit))
        if not metin_alani_odakta and self.ana_pencere:
            if event.matches(QKeySequence.StandardKey.Undo):
                self.ana_pencere.geri_al()
                event.accept()
                return
            if event.matches(QKeySequence.StandardKey.Redo):
                self.ana_pencere.ileri_al()
                event.accept()
                return
            if event.matches(QKeySequence.StandardKey.Cut):
                self.ana_pencere.secili_bloklari_kes()
                event.accept()
                return
            if event.key() == Qt.Key.Key_A and event.modifiers() & Qt.KeyboardModifier.ControlModifier:
                self.ana_pencere.bloklari_tumunu_sec()
                event.accept()
                return
            if event.key() == Qt.Key.Key_D and event.modifiers() & Qt.KeyboardModifier.ControlModifier:
                self.ana_pencere.bloklari_cogalt()
                event.accept()
                return
            if event.key() == Qt.Key.Key_Escape:
                self.scene().clearSelection()
                event.accept()
                return
            if event.matches(QKeySequence.StandardKey.Copy):
                self.ana_pencere.bloklari_kopyala()
                event.accept()
                return
            if event.matches(QKeySequence.StandardKey.Paste):
                self.ana_pencere.bloklari_yapistir()
                event.accept()
                return
            if event.key() == Qt.Key.Key_Delete:
                self.ana_pencere.secili_bloklari_sil()
                event.accept()
                return
        super().keyPressEvent(event)

    def drawBackground(self, painter, rect):
        aralik = self.izgara_araligi

        sol = int(rect.left()) - (int(rect.left()) % aralik)
        ust = int(rect.top()) - (int(rect.top()) % aralik)

        painter.setPen(QColor("#3b3b3b"))

        x = sol
        while x < rect.right():
            painter.drawLine(x, int(rect.top()), x, int(rect.bottom()))
            x += aralik

        y = ust
        while y < rect.bottom():
            painter.drawLine(int(rect.left()), y, int(rect.right()), y)
            y += aralik

    def wheelEvent(self, event):
        # Fare tekerleğiyle zoonm ve uzaklaştırma
        if event.angleDelta().y() > 0:
            faktor = 1.15
        else:
            faktor = 1 / 1.15

        self.scale(faktor, faktor)

    def mousePressEvent(self, event):
        if event.button() == Qt.MouseButton.MiddleButton:
            self._son_fare_konumu = event.position().toPoint()
            self.setCursor(Qt.CursorShape.ClosedHandCursor)
            event.accept()
            return

        super().mousePressEvent(event)

    def mouseMoveEvent(self, event):
        if (
            event.buttons() & Qt.MouseButton.MiddleButton
            and hasattr(self, "_son_fare_konumu")
        ):
            yeni_konum = event.position().toPoint()
            fark = yeni_konum - self._son_fare_konumu

            self._son_fare_konumu = yeni_konum

            self.horizontalScrollBar().setValue(
                self.horizontalScrollBar().value() - fark.x()
            )
            self.verticalScrollBar().setValue(
                self.verticalScrollBar().value() - fark.y()
            )

            event.accept()
            return

        super().mouseMoveEvent(event)

    def mouseReleaseEvent(self, event):
        if event.button() == Qt.MouseButton.MiddleButton:
            self.unsetCursor()
            if hasattr(self, "_son_fare_konumu"):
                del self._son_fare_konumu

            event.accept()
            return

        super().mouseReleaseEvent(event)
    
    def dragEnterEvent(self, event):
        if event.mimeData().hasText():
            event.acceptProposedAction()
        else:
            event.ignore()

    def dragMoveEvent(self, event):
        if event.mimeData().hasText():
            event.acceptProposedAction()
        else:
            event.ignore()


    
    def dizini_bagla(self, yeni_dizin):
        en_yakin = None
        en_kucuk_mesafe = 45

        yeni_ust = yeni_dizin.mapToScene(
            yeni_dizin.boundingRect().topLeft()
        )

        for dizin in self.scene().items():
            if not isinstance(dizin, KodDizini):
                continue

            if dizin is yeni_dizin:
                continue

            alt = dizin.mapToScene(
                dizin.boundingRect().bottomLeft()
            )

            dx = abs(alt.x() - yeni_ust.x())
            dy = abs(alt.y() - yeni_ust.y())

            if dx < 60 and dy < en_kucuk_mesafe:
                en_kucuk_mesafe = dy
                en_yakin = dizin

        if en_yakin is None:
            return

        alt = en_yakin.mapToScene(
            en_yakin.boundingRect().bottomLeft()
        )

        ust = yeni_dizin.mapToScene(
            yeni_dizin.boundingRect().topLeft()
        )

        yeni_dizin.setPos(
            yeni_dizin.pos().x() + alt.x() - ust.x(),
            yeni_dizin.pos().y() + alt.y() - ust.y()
        )

        en_yakin.build_uyesini_ekle(yeni_dizin)


    
    def dropEvent(self, event):
        if event.mimeData().hasText():
            onceki_durum = self.ana_pencere.durum_kaydet() if self.ana_pencere else None
            konum = self.mapToScene(
                event.position().toPoint()
            )
            ad = event.mimeData().text()

            tur = next(
                (
                    anahtar
                    for anahtar, bilgi in KodDizini.DIZINLER.items()
                    if bilgi[0] == ad
                ),
                "yazdir"
            )

            dizin = KodDizini(
                konum.x(),
                konum.y(),
                tur
            )

            self.scene().addItem(dizin)
            self.dizini_bagla(dizin)
            if self.ana_pencere and onceki_durum is not None:
                self.ana_pencere.islemi_gecmise_ekle(onceki_durum)
            event.acceptProposedAction()
        else:
            event.ignore()



class DizinListesi(QListWidget):
    def startDrag(self, supportedActions):
        secili = self.currentItem()

        if secili is None:
            return

        surukleme = QDrag(self)
        veri = QMimeData()
        veri.setText(secili.text())
        surukleme.setMimeData(veri)

        surukleme.exec(Qt.DropAction.CopyAction)

class AnaPencere(QMainWindow):
    def __init__(self):
        super().__init__()

        self.setWindowTitle("RealCoder")
        self.resize(1200, 800)

        # Çalışma alanı
        self.sahne = QGraphicsScene(self)
        self.calisma_alani = CalismaAlani(self.sahne)
        self.calisma_alani.ana_pencere = self
        self._undo_gecmisi = []
        self._redo_gecmisi = []
        self._ek_pencereler = []
        self._kopya_bloklar = None
        self._yapistirma_sayisi = 0


        # Dizin paleti
        self.dizin_listesi = DizinListesi()

        self.arama = QLineEdit()
        self.arama.setPlaceholderText("🔎 Dizin ara...")
        self.arama.setStyleSheet("""
            QLineEdit {
                background-color: #303030;
                color: white;
                border: 2px solid #555555;
                border-radius: 8px;
                padding: 8px;
                font-size: 13px;
            }
            QLineEdit:focus {
                border-color: #f0b429;
            }
        """)

        self.arama.textChanged.connect(self.dizinleri_filtrele)

        self.dizin_listesi.setFixedWidth(190)
        self.dizin_listesi.setStyleSheet("""
            QListWidget {
                background-color: #252525;
                color: white;
                border: none;
                padding: 6px;
            }
            QListWidget::item {
                background-color: #E9A23B;
                color: #34240C;
                padding: 12px;
                margin: 4px;
                border-radius: 6px;
            }
            QListWidget::item:selected {
                border: 2px solid white;
            }
        """)

        for bilgi in KodDizini.DIZINLER.values():
            self.dizin_listesi.addItem(bilgi[0])

        # ayırmak için
        self.dizin_listesi.setDragEnabled(True)
        self.dizin_listesi.setDragDropMode(
            QAbstractItemView.DragDropMode.DragOnly
        )

        # Üretilen Python kodu ve çalıştırma çıktısı
        self.python_kodu = QPlainTextEdit()
        self.python_kodu.setReadOnly(True)
        self.python_kodu.setPlaceholderText("Bloklardan üretilen Python kodu...")

        self.terminal = QPlainTextEdit()
        self.terminal.setReadOnly(True)
        self.terminal.setPlaceholderText("Program çıktıları...")

        # Çalıştır butonu
        self.calistir_butonu = QPushButton("▶ Çalıştır")
        self.calistir_butonu.clicked.connect(self.programi_calistir)

        # Orta ve alt paneller
        orta_panel = QWidget()
        orta_duzen = QVBoxLayout(orta_panel)
        orta_duzen.addWidget(self.calistir_butonu)
        orta_duzen.addWidget(self.calisma_alani)

        alt_panel = QWidget()
        alt_duzen = QVBoxLayout(alt_panel)
        alt_duzen.addWidget(QLabel("Üretilen Python kodu"))
        alt_duzen.addWidget(self.python_kodu)
        alt_duzen.addWidget(QLabel("Çalıştırma çıktısı"))
        alt_duzen.addWidget(self.terminal)

        # Sol menü + çalışma alanı
        sol_panel = QWidget()
        sol_duzen = QVBoxLayout(sol_panel)
        sol_duzen.addWidget(self.arama)
        sol_duzen.addWidget(self.dizin_listesi)

        ust_panel = QWidget()
        ust_duzen = QHBoxLayout(ust_panel)
        ust_duzen.setContentsMargins(0, 0, 0, 0)
        ust_duzen.addWidget(sol_panel)
        ust_duzen.addWidget(orta_panel, 1)

        # Ana pencere
        ana_widget = QWidget()
        ana_duzen = QVBoxLayout(ana_widget)
        ana_duzen.addWidget(ust_panel, 4)
        ana_duzen.addWidget(alt_panel, 1)

        self.setCentralWidget(ana_widget)
        self.arac_cubugunu_olustur()

    def arac_cubugunu_olustur(self):
        dosya_menu = self.menuBar().addMenu("Dosya")
        duzen_menu = self.menuBar().addMenu("Düzen")
        terminal_menu = self.menuBar().addMenu("Terminal")
        pencere_menu = self.menuBar().addMenu("Pencere")

        self.disari_aktar_aksiyonu = QAction("Python olarak dışa aktar", self)
        self.disari_aktar_aksiyonu.triggered.connect(self.python_disari_aktar)
        dosya_menu.addAction(self.disari_aktar_aksiyonu)

        self.yeni_terminal_aksiyonu = QAction("Yeni terminal", self)
        self.yeni_terminal_aksiyonu.triggered.connect(self.yeni_terminal_ac)
        terminal_menu.addAction(self.yeni_terminal_aksiyonu)

        self.yeni_pencere_aksiyonu = QAction("Yeni pencere", self)
        self.yeni_pencere_aksiyonu.triggered.connect(self.yeni_pencere_ac)
        pencere_menu.addAction(self.yeni_pencere_aksiyonu)

        self.geri_al_aksiyonu = QAction("Geri al\tCtrl+Z", self)
        self.geri_al_aksiyonu.triggered.connect(self.geri_al)
        duzen_menu.addAction(self.geri_al_aksiyonu)
        self.yinele_aksiyonu = QAction("Yinele\tCtrl+Y", self)
        self.yinele_aksiyonu.triggered.connect(self.ileri_al)
        duzen_menu.addAction(self.yinele_aksiyonu)
        duzen_menu.addSeparator()

        duzen_aksiyonlari = [
            ("Kes\tCtrl+X", self.secili_bloklari_kes),
            ("Kopyala\tCtrl+C", self.bloklari_kopyala),
            ("Yapıştır\tCtrl+V", self.bloklari_yapistir),
            ("Tümünü seç\tCtrl+A", self.bloklari_tumunu_sec),
            ("Seçimi sil\tDelete", self.secili_bloklari_sil),
            ("Seçimi kaldır\tEsc", self.secimi_kaldir),
        ]
        for yazi, islem in duzen_aksiyonlari:
            aksiyon = QAction(yazi, self)
            aksiyon.triggered.connect(islem)
            duzen_menu.addAction(aksiyon)

        arac = QToolBar("Hızlı erişim", self)
        arac.setMovable(False)
        self.addToolBar(arac)
        for aksiyon in (
            self.disari_aktar_aksiyonu,
            self.yeni_terminal_aksiyonu,
            self.yeni_pencere_aksiyonu,
        ):
            arac.addAction(aksiyon)
        arac.addSeparator()
        arac.addAction(self.geri_al_aksiyonu)
        arac.addAction(self.yinele_aksiyonu)

    def bloklari_tumunu_sec(self):
        for blok in self.sahne.items():
            if isinstance(blok, KodDizini):
                blok.setSelected(True)

    def secimi_kaldir(self):
        self.sahne.clearSelection()

    def yeni_pencere_ac(self):
        pencere = AnaPencere()
        self._ek_pencereler.append(pencere)
        pencere.show()

    def yeni_terminal_ac(self):
        try:
            subprocess.Popen(
                ["cmd.exe"],
                creationflags=subprocess.CREATE_NEW_CONSOLE,
            )
        except Exception as exc:
            QMessageBox.warning(self, "Terminal açılamadı", str(exc))

    def python_disari_aktar(self):
        bloklar = [i for i in self.sahne.items() if isinstance(i, KodDizini)]
        if not bloklar:
            QMessageBox.information(self, "Dışa aktar", "Önce çalışma alanına blok ekle.")
            return
        try:
            kod = RealCoderMotoru(bloklar).kod_uret()
        except Exception as exc:
            QMessageBox.warning(self, "Kod üretilemedi", str(exc))
            return
        dosya_adi, _ = QFileDialog.getSaveFileName(
            self,
            "Python dosyasını kaydet",
            "program.py",
            "Python dosyaları (*.py)",
        )
        if not dosya_adi:
            return
        if not dosya_adi.lower().endswith(".py"):
            dosya_adi += ".py"
        kaynak = "import random\nimport time\nfrom pathlib import Path\n\n" + kod + "\n"
        try:
            Path(dosya_adi).write_text(kaynak, encoding="utf-8")
            QMessageBox.information(self, "Dışa aktarıldı", f"Python dosyası kaydedildi:\n{dosya_adi}")
        except Exception as exc:
            QMessageBox.warning(self, "Dosya kaydedilemedi", str(exc))

    def durum_kaydet(self):
        bloklar = [i for i in self.sahne.items() if isinstance(i, KodDizini)]
        bloklar.sort(key=lambda b: (b.pos().y(), b.pos().x(), b.tur))
        indeks = {blok: i for i, blok in enumerate(bloklar)}
        return [
            {
                "tur": b.tur,
                "x": b.pos().x(),
                "y": b.pos().y(),
                "metin": b.metin_alani.text(),
                "degisken_adi": b.degisken_adi_alani.text() if b.degisken_adi_alani else None,
                "sonraki": indeks.get(b.sonraki),
                "bagli": sorted(indeks[o] for o in b.bagli_dizinler if o in indeks),
                "secili": b.isSelected(),
            }
            for b in bloklar
        ]

    def islemi_gecmise_ekle(self, onceki_durum):
        if onceki_durum is None:
            return
        simdiki = self.durum_kaydet()
        if onceki_durum != simdiki:
            self._undo_gecmisi.append(onceki_durum)
            self._undo_gecmisi = self._undo_gecmisi[-100:]
            self._redo_gecmisi.clear()

    def _durumu_yukle(self, durum):
        self.sahne.clear()
        bloklar = []
        for kayit in durum:
            blok = KodDizini(kayit["x"], kayit["y"], kayit["tur"])
            blok.metin_alani.setText(kayit["metin"])
            if blok.degisken_adi_alani and kayit["degisken_adi"] is not None:
                blok.degisken_adi_alani.setText(kayit["degisken_adi"])
            self.sahne.addItem(blok)
            bloklar.append(blok)
        for i, kayit in enumerate(durum):
            bloklar[i].sonraki = bloklar[kayit["sonraki"]] if kayit["sonraki"] is not None else None
            bloklar[i].bagli_dizinler = [bloklar[j] for j in kayit["bagli"]]
            bloklar[i].setSelected(kayit["secili"])

    def geri_al(self):
        if not self._undo_gecmisi:
            return
        self._redo_gecmisi.append(self.durum_kaydet())
        self._durumu_yukle(self._undo_gecmisi.pop())

    def ileri_al(self):
        if not self._redo_gecmisi:
            return
        self._undo_gecmisi.append(self.durum_kaydet())
        self._durumu_yukle(self._redo_gecmisi.pop())

    def secili_bloklari_kes(self):
        if not any(isinstance(i, KodDizini) for i in self.sahne.selectedItems()):
            return
        self.bloklari_kopyala()
        self.secili_bloklari_sil()

    def bloklari_kopyala(self):
        secilenler = [i for i in self.sahne.selectedItems() if isinstance(i, KodDizini)]
        if not secilenler:
            return
        secilenler.sort(key=lambda b: (b.pos().y(), b.pos().x()))
        indeks = {blok: i for i, blok in enumerate(secilenler)}
        min_x = min(b.pos().x() for b in secilenler)
        min_y = min(b.pos().y() for b in secilenler)
        self._kopya_bloklar = [
            {
                "tur": b.tur,
                "x": b.pos().x() - min_x,
                "y": b.pos().y() - min_y,
                "metin": b.metin_alani.text(),
                "degisken_adi": b.degisken_adi_alani.text() if b.degisken_adi_alani else None,
                "sonraki": indeks.get(b.sonraki),
            }
            for b in secilenler
        ]
        self._yapistirma_sayisi = 0

    def bloklari_yapistir(self):
        if not self._kopya_bloklar:
            return
        onceki = self.durum_kaydet()
        merkez = self.calisma_alani.mapToScene(self.calisma_alani.viewport().rect().center())
        yeni_bloklar = []
        kayma = 25 * (self._yapistirma_sayisi + 1)
        for kayit in self._kopya_bloklar:
            blok = KodDizini(merkez.x() + kayit["x"] + kayma, merkez.y() + kayit["y"] + kayma, kayit["tur"])
            blok.metin_alani.setText(kayit["metin"])
            if blok.degisken_adi_alani and kayit["degisken_adi"] is not None:
                blok.degisken_adi_alani.setText(kayit["degisken_adi"])
            self.sahne.addItem(blok)
            yeni_bloklar.append(blok)
        for i, kayit in enumerate(self._kopya_bloklar):
            j = kayit["sonraki"]
            if j is not None:
                yeni_bloklar[i].sonraki = yeni_bloklar[j]
                yeni_bloklar[i].bagli_dizinler.append(yeni_bloklar[j])
                yeni_bloklar[j].bagli_dizinler.append(yeni_bloklar[i])
        self.sahne.clearSelection()
        for blok in yeni_bloklar:
            blok.setSelected(True)
        self._yapistirma_sayisi += 1
        self.islemi_gecmise_ekle(onceki)

    def bloklari_cogalt(self):
        secilenler = [i for i in self.sahne.selectedItems() if isinstance(i, KodDizini)]
        if not secilenler:
            return
        onceki = self.durum_kaydet()
        secilenler.sort(key=lambda b: (b.pos().y(), b.pos().x()))
        indeks = {blok: i for i, blok in enumerate(secilenler)}
        yeni_bloklar = []
        for eski in secilenler:
            yeni = KodDizini(eski.pos().x() + 25, eski.pos().y() + 25, eski.tur)
            yeni.metin_alani.setText(eski.metin_alani.text())
            if yeni.degisken_adi_alani and eski.degisken_adi_alani:
                yeni.degisken_adi_alani.setText(eski.degisken_adi_alani.text())
            self.sahne.addItem(yeni)
            yeni_bloklar.append(yeni)
        for i, eski in enumerate(secilenler):
            sonraki_indeks = indeks.get(eski.sonraki)
            if sonraki_indeks is not None:
                yeni_bloklar[i].sonraki = yeni_bloklar[sonraki_indeks]
                yeni_bloklar[i].bagli_dizinler.append(yeni_bloklar[sonraki_indeks])
                yeni_bloklar[sonraki_indeks].bagli_dizinler.append(yeni_bloklar[i])
        self.sahne.clearSelection()
        for blok in yeni_bloklar:
            blok.setSelected(True)
        self.islemi_gecmise_ekle(onceki)

    def secili_bloklari_sil(self):
        secilenler = {i for i in self.sahne.selectedItems() if isinstance(i, KodDizini)}
        if not secilenler:
            return
        onceki = self.durum_kaydet()
        bloklar = [i for i in self.sahne.items() if isinstance(i, KodDizini)]
        for blok in bloklar:
            if blok in secilenler:
                continue
            sonraki = blok.sonraki
            while sonraki in secilenler:
                sonraki = sonraki.sonraki
            blok.sonraki = sonraki
            blok.bagli_dizinler = [o for o in blok.bagli_dizinler if o not in secilenler]
        for blok in list(secilenler):
            for view in self.sahne.views():
                if isinstance(view, CalismaAlani):
                    view.scene().removeItem(blok)
                    break
        kalanlar = [i for i in self.sahne.items() if isinstance(i, KodDizini)]
        for blok in kalanlar:
            if blok.sonraki and blok.sonraki not in blok.bagli_dizinler:
                blok.bagli_dizinler.append(blok.sonraki)
            if blok.sonraki and blok not in blok.sonraki.bagli_dizinler:
                blok.sonraki.bagli_dizinler.append(blok)
        self.islemi_gecmise_ekle(onceki)

    def dizinleri_filtrele(self, arama_metni):
        arama_metni = arama_metni.casefold()
        for i in range(self.dizin_listesi.count()):
            oge = self.dizin_listesi.item(i)
            oge.setHidden(arama_metni not in oge.text().casefold())

    def programi_calistir(self):
        self.terminal.clear()
        self.python_kodu.clear()
        dizinler = [item for item in self.sahne.items() if isinstance(item, KodDizini)]

        if not dizinler:
            self.terminal.setPlainText("Çalıştırılacak blok yok.")
            return

        try:
            motor = RealCoderMotoru(dizinler)
            kod = motor.kod_uret()
            self.python_kodu.setPlainText(kod)

            # input() çağrıları arayüz üzerinden kullanıcıya sorulur.
            def arayuz_girdisi(prompt="Veri gir:"):
                deger, tamam = QInputDialog.getText(self, "Kullanıcı girdisi", str(prompt))
                if not tamam:
                    raise RuntimeError("Kullanıcı girişi iptal edildi.")
                return deger

            cikti = io.StringIO()
            ortam = {
                "__builtins__": __builtins__,
                "random": random,
                "time": time,
                "Path": Path,
                "input": arayuz_girdisi,
            }
            with redirect_stdout(cikti):
                exec(compile(kod, "<RealCoder>", "exec"), ortam, ortam)
            self.terminal.setPlainText(cikti.getvalue() or "Program tamamlandı; çıktı üretmedi.")
        except Exception as exc:
            self.terminal.setPlainText(f"Hata: {exc}")


if __name__ == "__main__":
    uygulama = QApplication(sys.argv)
    pencere = AnaPencere()
    pencere.show()
    sys.exit(uygulama.exec())


