---
tags:
  - Backend
  - Project
---
V tomhle projektu jsem začal s tím, že jsem chtěl poprvé namočit své prsty do umění backend.
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
Z tohoto důvodu nainstaluji **nodemon**, tohle ale není něco co budu potřebovat, když to budu pusovat do produkce a je to potřeba jen pro mě jako developera. Pro to použiji jiný způsob instalace `npm install nodemon --save-dev nodemon`, kde to pak v package.json vypadá následovně
```JSON
  "dependencies": {
    "express": "^5.2.1"
  },
  "devDependencies": {
    "nodemon": "^3.1.11"
  }
}
```
A upravil jsem `"dev": "node server.js"` na `"dev": "nodemon server.js"`