---
tags:
  - Backend
  - Project
---
V tomhle projektu jsem začal s tím, že jsem chtěl poprvé namočit své prsty do umění [[Backend]].
Na začátku jsem si stáhl potřebné věci jako byli VSCode, Docker, Node.js a Express. Ve VSCode jsem inicializoval projekt pomoci `npm init -y` a nainstaloval express pomocí `npm install express --save`. Vytvořila se pro nás zajímavá file jménem **package.json**, ve které se ukládá takové nastavení ohledně našeho programu. Po nainstalování express se do této file přidala nová část jménem dependencies, kde se napíší, všechny externí programy, na kterých náš program závisí.
*package.json:*
```json
{
  "name": "chapter_2",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "type": "commonjs",
  "dependencies": {
    "express": "^5.2.1"
  }
}
```

Poté jsem si vytvořil novou file jménem **server.js** a napsal do něj
```JS
const express = require('express') //Ujistíme se, že máme express a uložíme ho do proměné
const app = express() //Inicializujeme backend aplikaci
const PORT = 8383

app.listen(PORT, () => console.log(`Server začal běžet na: ${PORT}`))
```
a následně ho spustil za pomocí `node server.js` s odpovědí `Server začal běžet na: 8383`.

Rozhodl jsem se, že to takhle spouštět není to co chci a šel jsem upravit package.json a to hlavně část **scripts**.
```JSON
 "scripts": {
    "dev": "node server.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
```
Kde dev je po *dev* je název scriptu a to potom je to příkaz co se spustí. Teď už mohu zadat příkaz `npm run dev` a dostanu odpověď
```shell
> chapter_2@1.0.0 dev
> node server.js

Server začal běžet na: 8383
```

Tohle stáhle ale není zbůsob jak nakonec budu spouštět tyto scripty. A proč? protože kdybych přišel a do původního server.js bych dodal třeba `console.log('Ahoj světe')`, tak bych musel vypnout a zapnout script, abych změnu viděl. To není chování co od toho požaduji.
Z tohoto důvodu nainstaluji **nodemon**, tohle ale není něco co budu potřebovat, když to budu pusovat do produkce a je to potřeba jen pro mě jako developera. Pro to použiji jiný způsob instalace `npm install nodemon --save-dev nodemon`, kde to pak v package.json vypadá následovně:
```JSON
  "dependencies": {
    "express": "^5.2.1"
  },
  "devDependencies": {
    "nodemon": "^3.1.11"
  }
}
```
A upravil jsem `"dev": "node server.js"` na `"dev": "nodemon server.js"`. Teď když znovu spustíme script za pomoci ``npm run dev``, tak kdykoliv uložíme server.js tak se nám v konzoli obnoví script a vidíme aktuální output.
Taky si můžeme ozkoušet, že ten server opravdu běží. Když půjdeme do webového prohlížeče a zadáme jeho URL **http://localhost:8383** nebo jeho IP **127.0.0.1:8383**, tak se nám na obrazovce objeví *Cannot GET /*.
Teď víme, že budeme specifikovat **HTTPS VERBS**([[Metody]]), **Routes**(trasy) **Paths**(cesty). VERBS jsou třeba GET, POST a HEAD, zatímco Routes určuje co aplikace udělá, když někdo přijde na danou path(kombinace: path + HTTP metoda + logika) a Path je konkrétní adresa kam uživatel/klient posílá požadavek. Do server.js jsem si dopsal následující
```JS
// HTTP VERBS (neboli method) && Routes (nebo paths)

// Metoda nám řekne o jaký typ požadavku se jedná a trasa je taková podsložka (praktaicky přesměrujeme požadavek na ten kus kódu, aby jsme odpovědeli tím co po nás požaduje. Těmto lokacím a trasám říkáme endpointy)

app.get('/', (req, res) => {
    // Tohle je endpoint #1 - /
    console.log('Jupí našel jsem endpoint')
})
```
Kde jsem specifikoval metodu pomocí `.get`, pomocí `'/'` cestu a následovně blok kódu co se spustí když se na tento endpoint dostaneme. Když teď uložim file a v prohlížeči otevřu http://127.0.0.1:8383/, tak do server konzole dostanu
```Shell
[nodemon] restarting due to changes...
[nodemon] starting `node server.js`
Server začal běžet na: 8383
Jupí našel jsem endpoint
```
Tak a teď co jsou ty *req* a *res*? to jsou dvě zkráceniny pro [[request]] a [[response]], které mohu použít na různé věci. **Req** mi slouží na získání toho co posílá klient serveru a **Res** mi slouží na reprezentaci toho co pošle server zpět klientovi. Pro vyzkoušení jsem v server.js změnil `app.get()` následovně
```JS
app.get('/', (req, res) => {
    // Tohle je endpoint #1 - /
    console.log('Jupí našel jsem endpoint', req.method)
    res.sendStatus(200)
})
```
Teď když to uložím a otevřu v prohlížeči http://127.0.0.1:8383/, tak nedostanu error ale napíše mi to OK. Taky se mi v server konzoli vypíše
```Shell
[nodemon] restarting due to changes...
[nodemon] starting `node server.js`
Server začal běžet na: 8383
Jupí našel jsem endpoint GET
```
`.sendStatus()` je metoda co posílá [[Status kód]]. 200 je právě ten důvod proč se zjeví OK.

Kdybych chtěl udělat další endpoint tak mi stačí udělat následovné
```JS
app.get('/blazinek', (req,res) => {
	// Tohle je endpoint #2 - /blazinek
    console.log('Ty jsi to našel ty blázínku')
    res.send('Ahojda')
})
```
a mám nový endpoint na  http://127.0.0.1:8383/blazinek, který vrací status kód 200 a v prohlížeči se mi místo OK zobrazuje *Ahojda*.

Teď bych chtěl poslat nějaký html a upřímně udělám to tím nejlínějším způsobem co jde. `res.send('TADY DÁM HTML KÓD')` a to bude vypadat následovně
```JS
app.get('/', (req, res) => {
    // Tohle je endpoint #1 - /
    res.send('<h1>Tohle je opravdová stránka. TRUST ;) </h1><input/>')
})
```
A teď se mi do html body dá h1 s textem *Tohle je opravdová stránka. TRUST ;* a pod tím input pole.