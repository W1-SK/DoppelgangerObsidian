---
tags:
  - Backend
  - Project
---
V tomhle projektu jsem začal s tím, že jsem chtěl poprvé namočit své prsty do umění backend.
Na začátku jsem si stáhl potřebné věci jako byli VSCode, Docker, Node.js, Express. V VSCode jsem inicializoval projekt pomoci `npm init -y` a nainstaloval express pomocí `npm install express --save`. Vytvořila se pro nás zajímavá file jménem **package.json**, ve které se ukládá takové nastavení ohledně našeho programu. Po nainstalování express se do této file přidala nová část jménem dependencies, kde se napíší, všechny externí programy, na kterých náš program závisí.

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

Poté jsem si vytvořil novou file jménem **server.js** 