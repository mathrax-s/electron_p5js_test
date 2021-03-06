# Electron Fiddle+p5jsでデスクトップアプリをつくる例（Electron v15）

v11までは「renderer.js」に、p5jsを使うため <strong>require('p5')</strong> と書けていたのに、v12以降はこれができなくなってしまいました。次の画面が実際のエラーです。
![スクリーンショット 2021-11-01 14.51.15.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/668579/e4abe53d-7864-6e1c-c0ca-6b66917a9c49.png)

いまのところ解決する方法は、いまいちスッキリしない気がしますが、大きく3つ見つけました。もっといい方法があると思うので、何かわかる方がいらっしゃたらご教授いただけるとありがたいです。先に手軽な方法を2つあげて、3つ目に自分の採用した方法をあげます。4つ目は、試したけどうまくいかなかった方法です。


## sample_1：nodeIntefrationをtrue、contextIsolationをfalseにする
1つめは、Electron v12以降でも、v11以前のように書いて動作できる方法です。Electronのセキュリティ設定を甘くすることになるので、おすすめではないです。

v12から「nodeIntegration」という設定が、デフォルトで「false」になっていて、同じく「contextIsolation」という設定はデフォルトで「true」になっています。これらはmain.jsの「webPreference」というところで設定できます。

- nodeIntegrationをtrue
- contextIsolationをfalse

とするとv11のときのように「renderer.js」にも<strong>require('p5')</strong>と書けます。

手っ取り早くコンパイルしたい場合や、自分のローカルな環境で自己責任で使う場合にはよいと思います。

~~~main.js
function createWindow () {
  // Create the browser window.
  const mainWindow = new BrowserWindow({
    width: 800,
    height: 600,
    webPreferences: {
      preload: path.join(__dirname, 'preload.js'),
      nodeIntegration: true,  // nodeIntegrationをtrueにして、
      contextIsolation: false // contextIsolationをfalseにする
    }
  })

  // and load the index.html of the app.
  mainWindow.loadFile('index.html')

  // Open the DevTools.
  // mainWindow.webContents.openDevTools()
}
~~~

## sample_2：preload.jsに全部書く
2つめは、プログラムを書くファイルを「renderer.js」ではなく、「preload.js」に書く方法です。「preload.js」なら <strong>require('p5')</strong>  が使えるので、ここに全て書くと一応動作できました。しかしセキュリティ的にどうなるのか、よく理解できていないのでわからないのですが、せっかく用意された「renderer.js」の意味がなくなるので、おすすめではないだろうと思います。

~~~preload.js
const p5 = require('p5');

const s = (p) => {
  p.setup = () => {
    p.createCanvas(p.windowWidth, p.windowHeight);
  };

  p.draw = () => {
    p.background(100,10);
    p.ellipse(p.mouseX, p.mouseY, 20, 20);
  };
}
const myp5 = new p5(s);
~~~


## sample_3： require('p5') は使わない　（これを採用）
3つめは、<strong>require('p5')</strong> は使わず、「p5js」の本体である「p5.js」や「p5.min.js」ファイルを、HTMLから読み込んでおく方法です。こちらは標準的なp5jsの使い方でもあり、Electronのセキュリティ設定もいじらず「renderer.js」に書けるので、今のところこれがよいのかなと思います。ただ、セーブしたりコンパイルするのに、p5js本体を読みこみにいくので、少し処理が遅くなってしまいました。

p5.min.jsをダウンロードして、同じフォルダに入れます。そのあとindex.htmlで「renderer.js」を読み込む前に、＠p5.min.js」を読み込んでおきます。

~~~index.html
    <script src="./p5.min.js"></script>
    <script src="./renderer.js"></script>
~~~

~~~renderer.js
const s = (p) => {
  p.setup = () => {
    p.createCanvas(p.windowWidth, p.windowHeight);
  };

  p.draw = () => {
    p.background(100,10);
    p.ellipse(p.mouseX, p.mouseY, 20, 20);
  };
}
const myp5 = new p5(s);
~~~

---

## sample_4_sippai：うまくいかなかったサンプル

試したけどうまくいかなかったことも書いておきます。これがうまくいくとスッキリする気がします。

こちらに「<strong><a href="https://qiita.com/Quantum/items/4841aa18643b8ef1fc11#preload%E3%82%92%E7%B5%8C%E7%94%B1%E3%81%99%E3%82%8B%E3%81%93%E3%81%A8%E3%81%A7require%E3%82%92%E6%B8%A1%E3%81%99%E3%81%93%E3%81%A8%E3%81%8C%E3%81%A7%E3%81%8D%E3%81%BE%E3%81%99">preloadを経由することでrequireを渡すことができます</a></strong>」とありました。

なるほど。「renderer.js」と「preload.js」で、contextBridgeという方法が用意されていて、変数やオブジェクトを行き来させることができるんですね。そこで試してみた案です。

- 1. 「preload.js」で「require('p5')」したオブジェクトの変数を、contextBrigdeで「renderer.js」に渡す。
- 2. 「renderer.js」で、そのオブジェクト変数にp5jsのスケッチを渡して、「new」する。

というのを試みたのですが、2で「new」するところでエラーが出てしまって、うまくいきませんでした。

~~~preload.js
const { contextBridge, ipcRenderer } = require('electron')
contextBridge.exposeInMainWorld(
  "api", {
    p5: require('p5')
  }
)
~~~

~~~renderer.js
const p5 = window.api.p5;

const s = (p) => {
  p.setup = () => {
    p.createCanvas(p.windowWidth, p.windowHeight);
  };

  p.draw = () => {
    p.background(100,10);
    p.ellipse(p.mouseX, p.mouseY, 20, 20);
  };
}
const myp5 = new p5(s);
~~~

![スクリーンショット 2021-11-01 14.36.34.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/668579/48a3279d-6c4c-552f-430a-209704dcb5a7.png)

renderer.jsの「const myp5 = new p5(s);」の行で、「Uncaught Error: Cannot call a class as a function」とエラーが出てしまい、ここから先に行けず諦めてしまいました。

---

# 参考にしたサイト

https://qiita.com/Quantum/items/4841aa18643b8ef1fc11


