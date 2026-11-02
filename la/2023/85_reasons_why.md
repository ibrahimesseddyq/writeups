# 85_reasons_why

**CTF:** LA 2023  
**Category:** web  
**Original writeup:** <https://blog.hamayanhamayan.com/entry/2023/02/13/224020> — by **blog.hamayanhamayan.com**. All credit goes to the original author; this is an archived copy.

---

[web] 85_reasons_why

フラグが明確には書いていないが、SQL Injectionできそうな箇所がviews.pyにあった。

入力も普通に外部から渡せそう。

@ app.route ( '/image-search' , methods=[ 'GET' , 'POST' ]) def image_search (): if 'image-query' not in request.files or request.method == 'GET' : return render_template( 'image-search.html' , results=[]) incoming_file = request.files[ 'image-query' ] size = os.fstat(incoming_file.fileno()).st_size if size > MAX_IMAGE_SIZE: flash( "image is too large (50kb max)" ); return redirect(url_for( 'home' )) spic = serialize_image(incoming_file.read()) try : res = db.session.connection().execute( \ text( "select parent as PID from images where b85_image = '{}' AND ((select active from posts where id=PID) = TRUE)" .format(spic))) except Exception : return ( "SQL error encountered" , 500 ) ...

入力はserialize_imageによって変換処理がかかっている。

Ascii85 - Wikipediaというエンコーディングらしい。

def serialize_image (pp): b85 = base64.a85encode(pp) b85_string = b85.decode( 'UTF-8' , 'ignore' ) b85_string = re.sub( ' \\\\\\\\\\\\\' ' , '~' , b85_string) b85_string = re.sub( ' \' ' , ' \'\' ' , b85_string) b85_string = re.sub( '~' , ' \' ' , b85_string) b85_string = re.sub( ' \\ :' , '~' , b85_string) return b85_string

文字種はかなりあり、 print(base64.a85encode(b'\x15\x09\xfb').decode('UTF-8', 'ignore')) とやればシングルクオートも作れる。

なんとなく文字は作れそうなので、後半の変換を回避することを考える。

~ OR 1=1 -- がうまくいきそうだが、~はちょうどAscii85にない。

なので、最初の変換を使って \\\\\\' OR 1=1 -- とすればいいよさそうだが、変換過程でスペースが消えてしまう。

だが、これはspace2commentで回避可能。

あとは変換でおかしくならないように適当にやると \\\\\\'/**/OR/**/1=1 -- d が最終的な答えになる。

From Base85, To Hex - CyberChefTo_Hex('%5C%5Cx',0)&input=XFxcXFxcJy8qKi9PUi8qKi8xPTEgLS0gZA)
