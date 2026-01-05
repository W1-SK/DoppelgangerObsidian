# Accessibility report – shop.zigilab.cz
Tento dokument shrnuje zjištěné problémy v oblasti webové přístupnosti (accessibility) na webu **shop.zigilab.cz**.
Hodnocení vychází z principů a doporučení standardu **WCAG 2.x (úroveň AA)**, který je dnes považován za základní technický rámec pro přístupné weby.

---

## 1. Nedostatečný kontrast textu vůči pozadí
**WCAG: Contrast (Minimum) – AA**

Na webu se vyskytují případy, kdy **barevný kontrast textu vůči pozadí nesplňuje doporučené minimální hodnoty**. To může způsobovat zhoršenou čitelnost, zejména pro uživatele se zrakovým omezením nebo při horších světelných podmínkách.

Problém se aktuálně objevuje přibližně na **19 místech**, například u textů:
```html
<span class="x-text-content-text-primary">od 799,00 Kč</span>
<span class="x-anchor-text-primary">Zásady ochrany osobních dat</span>
<span class="x-anchor-text-primary">Platby</span>
<span class="x-anchor-text-primary">Doprava</span>
<span style="font-family: inherit; font-size: 1em; font-weight: inherit; letter-spacing: 0em;">+420 607 012 369<br></span>
<span style="font-family: inherit; font-size: 1em; font-weight: inherit; letter-spacing: 0em;">Adresa: Čimická 442/33, Praha 8 - 182 00</span>
<div class="x-text x-content e18-e28 mi-8 mi-9 mi-a mi-c">Zigilab © 2026</div>
```

### Proč je to problém
Text s nízkým kontrastem je pro část uživatelů obtížně nebo vůbec nečitelný. To se netýká jen lidí s diagnózou, ale i běžných uživatelů (slunce na displej, horší monitor, únava očí).

### Doporučení
* Pro **běžný text** dodržet kontrastní poměr alespoň **4,5 : 1**
* Pro **větší text** (cca 18 px nebo 14 px tučný) postačuje poměr **3 : 1**
* Kontrast lze snadno ověřit např. pomocí nástroje:
  👉 [https://webaim.org/resources/contrastchecker/](https://webaim.org/resources/contrastchecker/)

---

## 2. Hierarchie nadpisů (semantika)
**WCAG: Headings and Labels**

Na stránce se místy používají nadpisy bez jasné hierarchie (např. přeskočené úrovně). Nejde o kritickou chybu, ale o **doporučenou best practice**, která výrazně pomáhá uživatelům asistenčních technologií.

Příklad úpravy:
```html
<h6 class="x-text-content-text-primary">O nákupu</h6>
```

### Proč je to problém
Uživatelé čteček obrazovky se často pohybují po stránce právě pomocí nadpisů. Pokud nejsou v logickém pořadí (`h1 → h2 → h3…`), struktura stránky je pro ně matoucí.

### Doporučení
* Používat **sekvenční úrovně nadpisů**
* Nepřeskakovat úrovně (např. z `h2` rovnou na `h5`)
* `h1` používat pro hlavní nadpis stránky, nižší úrovně pro podsekce

---

## 3. Obrázky bez alternativního textu (`alt`)

**WCAG: Non-text Content**

Na webu se nachází obrázky, které **nemají vyplněný atribut `alt`**, například:

```html
<img src="//shop.zigilab.cz/wp-content/uploads/2025/11/background_zl.jpg" ...>
```

### Proč je to problém

Bez alternativního textu nemají uživatelé čteček obrazovky žádnou informaci o tom, co obrázek představuje – nebo zda má vůbec nějaký význam.

### Doporučení

* U **obsahových obrázků** použít smysluplný `alt` popis
* U **čistě dekorativních obrázků** použít prázdný atribut `alt=""`
* V případě potřeby lze využít:

  * `aria-label`
  * `aria-labelledby` (napojení na existující text nebo nadpis)

---

## 4. Odkazy bez srozumitelného textu

**WCAG: Link Purpose (In Context)**

Na webu se nachází odkazy, které jsou tvořeny pouze ikonou a **nemají žádný čitelný nebo programově dostupný text**, například:

```html
<a class="x-anchor x-anchor-button has-graphic" href="#">…</a>
```

Typicky se jedná o ikony sociálních sítí (Facebook, Instagram, LinkedIn).

### Proč je to problém

Čtečka obrazovky bez popisu neví, kam odkaz vede. Pro uživatele se pak jedná o „nepojmenovaný odkaz“, což je velmi matoucí.

### Doporučení

* Každý odkaz musí mít **srozumitelný popis účelu**
* U ikonických odkazů použít:

  * `aria-label` (např. „Instagram – Zigilab“)
  * nebo `aria-labelledby`
* Zachovat:

  * možnost fokusování klávesnicí
  * viditelný stav focusu
* Vždy používat skutečný `<a href="">` prvek, ne náhrady přes JavaScript

---

## 5. Nesprávně strukturované seznamy

**WCAG: Info and Relationships**

Na některých místech nejsou seznamy vytvořeny pomocí sémantických HTML prvků, případně je struktura nejednoznačná.

### Proč je to problém

Čtečky obrazovky se spoléhají na správnou strukturu (`<ul>`, `<ol>`, `<li>`). Pokud jsou seznamy „jen vizuální“, ztrácí se informace o vztazích mezi položkami.

### Doporučení

* Používat:

  * `<ul>` pro nečíslované seznamy
  * `<ol>` pro číslované seznamy
* Každou položku vkládat do `<li>`
* Nepoužívat `<div>` nebo `<span>` jako náhradu seznamů

---

## 6. Použití `<li>` mimo nadřazený seznam

**WCAG: Info and Relationships**

Byly nalezeny případy, kdy se `<li>` prvky nachází **mimo** rodičovský `<ul>` / `<ol>` prvek.

### Proč je to problém

Samostatný `<li>` bez seznamu nedává asistenčním technologiím žádný smysl a může vést k dezorientaci uživatele.

### Doporučení

* Každý `<li>` musí být vždy přímým potomkem:

  * `<ul>`
  * `<ol>`
  * nebo `<menu>`

---

## 7. Obsah mimo landmark regiony

**WCAG / ARIA: Landmarks**

Části obsahu (např. pozadí stránky) nejsou obaleny žádnými **landmark regiony**, například:

```html
<div class="backstretch" style="position: fixed;">
```

### Proč je to problém

Landmarky umožňují uživatelům čteček obrazovky rychle se orientovat na stránce (hlavička, navigace, hlavní obsah, patička).

### Doporučení

* Preferovat **nativní HTML5 prvky**:

  * `<header>`
  * `<nav>`
  * `<main>`
  * `<footer>`
* Pokud to není možné, použít **ARIA role**:

  * `role="banner"`
  * `role="navigation"`
  * `role="main"`
  * `role="contentinfo"`

---

## Závěr

Výše uvedené body nepředstavují „rozbitý web“, ale **konkrétní oblasti, kde lze výrazně zlepšit přístupnost** pro část uživatelů. Většina doporučení je:

* technicky relativně jednoduchá
* bez dopadu na vizuální design
* a zároveň zvyšuje celkovou kvalitu a použitelnost webu

Z pohledu WCAG se jedná o **typické a řešitelné problémy**, které je vhodné postupně odstranit, ideálně při běžném vývoji nebo redesignu.
