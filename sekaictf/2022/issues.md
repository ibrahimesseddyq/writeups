# issues

**CTF:** SekaiCTF 2022  
**Category:** misc  
**Original writeup:** <https://blog.hamayanhamayan.com/entry/2022/10/03/075236> — by **blog.hamayanhamayan.com**. All credit goes to the original author; this is an archived copy.

---

[web] Issues

フラグを得るためには /api/flag にアクセスする必要があるが、その前にauthorizeによってtoken検証される。

有効な認証トークンを作成しよう。app.pyを見るとログインは作成中なので、自力でtokenを作成して、エラーを突破していく。

No Authorization header found Authorization: Bearer [token] をヘッダーに追加して認証を進める

をヘッダーに追加して認証を進める 鍵を適当に作ってRS256でJWTトークンを入れてみる

秘密鍵作成 openssl genrsa -out private.pem 4096

公開鍵作成 openssl rsa -in private.pem -pubout -outform PEM -out public.pem issuer not found in JWT header JWTのissuerに適当なURLを入れて試す Invalid issuer netloc: {issuer}. Should be: {valid_issuer} ここの突破が問題

任意のjwks.jsonファイルを参照させることで鍵を強制させたいが、ドメインをチェックされる

悪用可能なリダイレクトが /logout にあるので悪用する

にあるので悪用する http://localhost:8080/logout?redirect=http%3a%2f%2fgoogle.com みたいにすれば任意のURLが使える

みたいにすれば任意のURLが使える requestcatcherでリクエストが来るか試すといい感じに来てくれる Expecting value: line 1 column 1 (char 0) 単にapi.pyの34行目 key = resp["keys"][0]["x5c"][0] でのエラー

でのエラー フォーマットがあってないだけ。与えられているjwks.jsonを参考に作成した公開鍵を適用する user claim missing {'user':'admin'} が要求されているので作るとフラグが得られる。

SEKAI{v4l1d4t3_y0ur_i55u3r_plz}
