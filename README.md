import uuid
from datetime import datetime
from abc import ABC, abstractmethod
from typing import List, Dict, Optional

# ======================== FOYDALANUVCHI SINFLARI ========================
class Foydalanuvchi:
    """Asosiy foydalanuvchi klassi (Abstraksiya va Inkapsulyatsiya)"""
    def init(self, ism: str, email: str, parol: str):
        self.id = str(uuid.uuid4())[:8]
        self.ism = ism
        self.email = email
        self._parol = parol  # Yashirinish (Encapsulation)

    def parolni_tekshir(self, kiritilgan_parol: str) -> bool:
        return self._parol == kiritilgan_parol

    def str(self):
        return f"{self.ism} ({self.email})"


class Mijoz(Foydalanuvchi):
    """Mijoz klassi (Naslik / Inheritance)"""
    def init(self, ism: str, email: str, parol: str):
        super().init(ism, email, parol)
        self.savat = Savat(self)
        self.buyurtmalar: List['Buyurtma'] = []

    def buyurtma_ber(self, tolov_usuli: 'TolovUsuli') -> Optional['Buyurtma']:
        if self.savat.boy:
            print("Savat bo'sh! Iltimos, mahsulot qo'shing.")
            return None
        
        umumiy_summa = self.savat.jami_summa()
        if tolov_usuli.tolov_amlash(umumiy_summa):
            buyurtma = Buyurtma(self, self.savat.nusxasi(), umumiy_summa)
            self.buyurtmalar.append(buyurtma)
            self.savat.tozalash()
            print(f"✅ Buyurtma qabul qilindi! ID: {buyurtma.id}")
            return buyurtma
        return None


class Admin(Foydalanuvchi):
    """Administrator klassi"""
    def init(self, ism: str, email: str, parol: str):
        super().init(ism, email, parol)

    def mahsulot_qosh(self, dokon: 'OnlaynDokon', nom: str, narx: float, kategoriya: str, miqdor: int):
        dokon.mahsulot_qoshish(nom, narx, kategoriya, miqdor)

    def mahsulot_ochir(self, dokon: 'OnlaynDokon', mahsulot_id: str):
        dokon.mahsulot_ochirish(mahsulot_id)


# ======================== MAHSULOT VA KATEGORIYA ========================
class Mahsulot:
    def init(self, nom: str, narx: float, kategoriya: str, miqdor: int = 0):
        self.id = str(uuid.uuid4())[:8]
        self.nom = nom
        self._narx = narx
        self.kategoriya = kategoriya
        self._ombordagi_miqdor = max(0, miqdor)
    @property
    def narx(self):
        return self._narx

    @property
    def ombordagi_miqdor(self):
        return self._ombordagi_miqdor
    def miqdorni_yangilash(self, ozgarish: int):
        if self._ombordagi_miqdor + ozgarish < 0:
            raise ValueError("Omborda yetarli mahsulot yo'q!")
        self._ombordagi_miqdor += ozgarish

    def str(self):
        return f"[{self.nom}] | {self._narx} so'm | Qoldiq: {self._ombordagi_miqdor}"


# ======================== SAVAT (CART) ========================
class Savat:
    def init(self, mijoz: Mijoz):
        self.mijoz = mijoz
        self._mahsulotlar: Dict[str, int] = {}  # {mahsulot_id: miqdor}
        self._mahsulotlar_obyekti: Dict[str, Mahsulot] = {}

    @property
    def boy(self):
        return len(self._mahsulotlar) == 0

    def mahsulot_qosh(self, mahsulot: Mahsulot, miqdor: int = 1):
        if miqdor <= 0:
            raise ValueError("Miqdor 0 dan katta bo'lishi kerak")
        if mahsulot.ombordagi_miqdor < miqdor:
            raise ValueError("Omborda yetarli mahsulot yo'q")

        mahsulot.miqdorni_yangilash(-miqdor)
        self._mahsulotlar[mahsulot.id] = self._mahsulotlar.get(mahsulot.id, 0) + miqdor
        self._mahsulotlar_obyekti[mahsulot.id] = mahsulot
        print(f"🛒 {mahsulot.nom} savatga qo'shildi ({miqdor} dona)")

    def jami_summa(self) -> float:
        return sum(self._mahsulotlar_obyekti[pid].narx * miqdor
                   for pid, miqdor in self._mahsulotlar.items())

    def nusxasi(self) -> Dict[str, int]:
        return dict(self._mahsulotlar)

    def tozalash(self):
        self._mahsulotlar.clear()
        self._mahsulotlar_obyekti.clear()

    def str(self):
        if self.boy:
            return "Savat bo'sh"
        qatorlar = []
        for pid, miqdor in self._mahsulotlar.items():
            m = self._mahsulotlar_obyekti[pid]
            qatorlar.append(f"  • {m.nom} x {miqdor} = {m.narx * miqdor} so'm")
        return "Savat:\n" + "\n".join(qatorlar) + f"\nJami: {self.jami_summa()} so'm"


# ======================== TO'LOV TIZIMI (Polimorfizm) ========================
class TolovUsuli(ABC):
    @abstractmethod
    def tolov_amlash(self, summa: float) -> bool:
        pass
   
class KartadanTolov(TolovUsuli):
    def init(self, karta_raqami: str):
        self.karta = karta_raqami

    def tolov_amlash(self, summa: float) -> bool:
        print(f"💳 {self.karta} kartasidan {summa} so'm yechilmoqda...")
        return True  # Simulyatsiya

class NaqdTolov(TolovUsuli):
    def tolov_amlash(self, summa: float) -> bool:
        print(f"💵 Naqd pul orqali {summa} so'm to'lanmoqda...")
        return True


# ======================== BUYURTMA ========================
class Buyurtma:
    def init(self, mijoz: Mijoz, mahsulotlar: Dict[str, int], jami_summa: float):
        self.id = str(uuid.uuid4())[:8]
        self.mijoz = mijoz
        self.mahsulotlar = mahsulotlar
        self.jami_summa = jami_summa
        self.holat = "Yangi"
        self.sana = datetime.now().strftime("%Y-%m-%d %H:%M")

    def holatni_yangilash(self, yangi_holat: str):
        self.holat = yangi_holat

    def str(self):
        return f"📦 Buyurtma #{self.id} | {self.mijoz.ism} | {self.jami_summa} so'm | Holat: {self.holat}"


# ======================== ASOSIY TIZIM (Facade) ========================
class OnlaynDokon:
    def init(self, nom: str):
        self.nom = nom
        self.mahsulotlar: Dict[str, Mahsulot] = {}
        self.foydalanuvchilar: Dict[str, Foydalanuvchi] = {}

    def royxatdan_otkaz(self, foydalanuvchi: Foydalanuvchi):
        self.foydalanuvchilar[foydalanuvchi.id] = foydalanuvchi
        print(f"👤 {foydalanuvchi.ism} ro'yxatdan o'tdi")
       
def mahsulot_qoshish(self, nom: str, narx: float, kategoriya: str, miqdor: int):
        m = Mahsulot(nom, narx, kategoriya, miqdor)
        self.mahsulotlar[m.id] = m
        print(f"📦 {nom} tizimga qo'shildi")

def mahsulot_ochirish(self, mahsulot_id: str):
        if mahsulot_id in self.mahsulotlar:
            del self.mahsulotlar[mahsulot_id]
            print("🗑️ Mahsulot o'chirildi")
        else:
            print("⚠️ Mahsulot topilmadi")

def barcha_mahsulotlar(self):
         print(f"\n {self.nom} do'konidagi mahsulotlar:")
for m in self.mahsulotlar.values():
            print(m)
            print()

def mijozni_top(self, user_id: str) -> Optional[Mijoz]:
           user = self.foydalanuvchilar.get(user_id)
return user if isinstance(user, Mijoz) else None

    # 1. Do'kon yaratish
dokon=OnlaynDokon("TechMarket.uz")

    # 2. Admin ro'yxatdan o'tish
admin = Admin("Admin", "admin@techmarket.uz", "admin123")
dokon.royxatdan_otkaz(admin)

    # 3. Mahsulotlarni qo'shish
admin.mahsulot_qosh(dokon, "iPhone 15", 15000000, "Telefon", 10)
admin.mahsulot_qosh(dokon, "MacBook Air", 22000000, "Noutbuk", 5)
admin.mahsulot_qosh(dokon, "AirPods Pro", 3500000, "Aksessuar", 20)

dokon.barcha_mahsulotlar()

    # 4. Mijoz ro'yxatdan o'tish
mijoz = Mijoz("Jasur", "jasur@mail.com", "12345")
dokon.royxatdan_otkaz(mijoz)

    # 5. Savatga mahsulot qo'shish
iphone = [m for m in dokon.mahsulotlar.values() if m.nom == "iPhone 15"][0]
airpods = [m for m in dokon.mahsulotlar.values() if m.nom == "AirPods Pro"][0]

mijoz.savat.mahsulot_qosh(iphone, 1)
mijoz.savat.mahsulot_qosh(airpods, 2)
print(mijoz.savat)

    # 6. To'lov va buyurtma
tolov = KartadanTolov("9860 1234 5678 9012")
mijoz.buyurtma_ber(tolov)

    # 7. Buyurtma holati
print("\n Buyurtmalar tarixi:")   
   for b in mijoz.buyurtmalar():
        print(b)


