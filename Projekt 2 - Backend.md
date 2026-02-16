---
tags:
  - Backend
  - Project
---
V tomto projektu začneme tím, že si stáhneme *nvm* neboli *Node Version Manager*. Tahle toolka nám umožnuje, stahovat a následně používat různé verze node.
V VSCode jsme si vytvořili složku chapter_3 a do ní zase pomocí `npm init -y` vložíme **package.json** a pak si vytvoříme následující věci:
``` Tree
└── chapter_3/
    ├── .env
    ├── package.json
    ├── todo-app.rest
    └── src/
        ├── db.js
        ├── server.js
        ├── routes/
        │   ├── authRoutes.js
        │   └── todoRoutes.js
        └── middleware/
            └── authMiddleware.js

```
Pak si nainstalujeme všechny potřebné packages pomocí `npm install express bcryptjs jsonwebtoken`. V package.json si přidáme script, který nám to bude vše zapínat.
```JSON
"scripts": {
    "dev": "node --watch --env-file=.env ./src/server.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  }
```
a když už jsme tam tak změníme z `"type": "module"` 