# Crear una aplicación con Eletron + ReactJS y configurarla para distribuirla
El objetivo de esta guía es crear desde cero la base para poder desarrollar una aplicación utilizando ReactJS y Electron.

Para evitar configuraciones complejas, utilizaremos el proyecto create-react-app, que provee una base sólida sobre la que desarrollar ReactJS.

Sobre esta base, añadiremos los componentes de ElectronJS. 

Finalmente, configuraremos el entorno para permitir construir tanto un instalable .exe y un portable de la aplicación.

## Pasos a seguir

### Instalar NodeJS descargándolo de [su página web](https://nodejs.org)

### Abrimos una terminal de comandos e instalamos yarn de manera global utilizando npm
```
npm install -g yarn
```
Una vez instalado, comprobamos que todo ha ido correctamente con `yarn --version`

### Creamos la base de ReactJS con create-react-app sobre una nueva carpeta con el nombre de la aplicación
```
npx create-react-app miApp
```
Una vez creada, entramos dentro de la carpeta con `cd miApp`

### Añadimos las dependencias del proyecto
```
yarn add electron-is-dev
yarn add electron electron-builder foreman --dev
```

[Foreman](https://strongloop.github.io/node-foreman/) permitirá ejecutar la aplicación desde la línea de comandos. Será útil más adelante.

### Creamos el fichero principal de Electron en la ruta `public/electron.js` con el código del `main.js` de [electron-quick-start](https://github.com/electron/electron-quick-start)

```javascript
const electron = require('electron');
const app = electron.app;
const BrowserWindow = electron.BrowserWindow;

const path = require('path');
const url = require('url');
const isDev = require('electron-is-dev');

let mainWindow;

function createWindow() {
  mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      nodeIntegration: true,
    },
  });
  mainWindow.loadURL(
    isDev
      ? process.env.ELECTRON_START_URL
      : url.format({
          pathname: path.join(__dirname, '../build/index.html'),
          protocol: 'file:',
          slashes: true,
        })
  );

  if (isDev) {
    // Open the DevTools.
    mainWindow.webContents.openDevTools();
  }
  mainWindow.on('closed', () => (mainWindow = null));
}

app.on('ready', createWindow);

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit();
  }
});

app.on('activate', () => {
  if (mainWindow === null) {
    createWindow();
  }
});
```

:red_circle: Del bloque anterior, prestar atención al código que carga el contenido de la ventana:
```javascript
mainWindow.loadURL(
    isDev
      ? process.env.ELECTRON_START_URL
      : url.format({
          pathname: path.join(__dirname, '../build/index.html'),
          protocol: 'file:',
          slashes: true,
        })
  );
```
Si estamos en el entorno de desarrollo, se usará la URL contenida en la variable `process.env.ELECTRON_START_URL`, que apuntará al servidor de desarrollo de webpack. En otro caso, se apuntará al fichero `../build/index.html`, que será el resultado de la construcción del código ReactJS por parte de webpack.

### Configurar Foreman y las variables de entorno

Creamos el fichero `Procfile` en la raíz de nuestra aplicación, con el siguiente contenido
```
react: yarn react-start
electron: yarn electron-start
```
Así, cuando se ejecte Foreman, se arrancarán en paralelo tanto el servidor de desarrollo de React como la aplicación Electron.

Por otro lado, crear el fichero `.env`, también en la carpeta raíz de la aplicación, con el siguiente contenido
```
BROWSER=none
```
Esto hará que cuando arranque el servidor de desarrollo, no se abra automáticamente en una pestaña del navegador, ya que el contenido se debe cargar desde Electron.

### Modificar el fichero `package.json`

Primero, cambiamos la sección de `scripts` para dejarla así:
```json
"scripts": {
    "start": "nf start -p 3000",
    "react-start": "react-scripts start",
    "react-build": "react-scripts build",
    "react-test": "react-scripts test",
    "react-eject": "react-scripts eject",
    "electron": "electron .",
    "electron-start": "node src/start-electron",
    "electron-build": "electron-builder",
    "build": "yarn react-build && yarn electron-build",
    "postinstall": "electron-builder install-app-deps"
}
```

Las órdenes `react-start`,`react-build`, `react-test` y `react-eject` vienen por defecto, ya que las crea `create-react-app`

Además de estas, se añaden las siguientes:

* `start` ejecutará `nf` (Foreman), que iniciará la aplicación en el puerto 3000. Para ello hará uso de los ficheros `Procfile` y `.env` definidos anteriormente.
* `electron` arrancará la aplicación Electron. Es ejecutado por el script de la siguiente opción.
* `electron-start` ejecutará un script .js localizado en src/start-electron.js, y que veremos en el siguiente punto. Este script se encargará de arrancar electron cuando se ejecute la aplicación en modo local. Esta orden es llamada desde el fichero `Procfile`.
* `electron-build` se encargará de construir la parte de Electron. No lo ejecutaremos nosotros directamente, por lo general.
* `build` construirá la aplicación final y genereará los `.exe` y portables que se hayan definido, para su distribución.
* `postinstall` se encarga de que las dependencias de la aplicación estén actualizadas después de instalar algo a través de Yarn.

Añadiremos también las siguientes etiquetas al fichero:
 ```
 "homepage": "./",
 "main": "public/electron.js",
 "description": "Prueba de app React+Electron",
 "author": "Javier López @lopezisere",
 ```
 * `homepage` permite que la aplicación cargue correctamente en Electron.
 * `main` indica en qué carpeta del proyecto se encuentra el script que Electron buscará para iniciar la aplicación.

### Crear el script de arranque en local de Electron en `src/start-electron.js` con el siguiente código.
```javascript
const net = require('net');
const childProcess = require('child_process');

const port = process.env.PORT ? process.env.PORT - 100 : 3000;
// Foreman will offset the port number by 100 for processes of different types. 
// (https://github.com/strongloop/node-foreman#advanced-usage)
// So, subtracts 100 to set the port number of the React dev server correctly.

process.env.ELECTRON_START_URL = `http://localhost:${port}`;

const client = new net.Socket();

let startedElectron = false;
const tryConnection = () => {
  client.connect({ port }, () => {
    client.end();
    if (!startedElectron) {
      console.log('starting electron');
      startedElectron = true;
      const exec = childProcess.exec;
      exec('yarn electron');
    }
  });
};

tryConnection();

client.on('error', () => {
  setTimeout(tryConnection, 1000);
});
```
Básicamente, este script permite enlazar Electron con ReactJS cuando se arranca en modo local, y realiza los siguientes pasos :
* Obtiene el número de puerto sobre el que arrancar
* Crea la variable `process.env.ELECTRON_START_URL` usada en la fucnión `mainWindow.LoadURL` del script `public/electron.js`
* Si el puerto local ya está accesible, arranca Electron a través de la ejecución de `yarn electron`

### Configurar cómo se construirá la aplicación.
Para ello, crearemos el fichero `electron-builder.yml` en la raíz de la aplicación, con el siguiente contenido:
```yml
appId: prueba.react
copyright: lopezishere
productName: ReacTron

directories:
  output: dist/

win:
  target:
    - target: nsis
      arch:
        - x64
    - target: portable
      arch:
        - x64
  icon: build/icon.ico
nsis:
  installerIcon: build/favicon.ico
  uninstallerIcon: build/favicon.ico
  perMachine: true
```
Esto generará tanto un `.exe` instalable como un archivo portable de la aplicación. Ambos se compilarán únicamente para arquitectura x64, aunque también es posible hacerlo para i32, si es necesario.

## Y ya está todo!
Con esto, ya está la aplicación preparada para arrancar.

La ejecutaremos en local con `yarn start`, y si queremos construir los ejecutables, lo haremos con `yarn build`

¡Eso es todo! Gracias por llegar hasta aquí :grinning:

## Referencias
Esta gúia no hubiese sido posible sin la ayuda de estas otras guías:
* https://medium.com/@xagustin93/a%C3%B1adiendo-react-a-una-aplicaci%C3%B3n-de-electron-ab3df35f48fd
* https://www.freecodecamp.org/news/building-an-electron-application-with-create-react-app-97945861647c/
* https://flaviocopes.com/react-electron/

¡Gracias!
