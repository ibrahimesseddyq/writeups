# natural_flag_processing

**CTF:** TSG 2021  
**Category:** rev  
**Original writeup:** <https://nanimokangaeteinai.hateblo.jp/entry/2021/10/03/191152> — by **nanimokangaeteinai.hateblo.jp**. All credit goes to the original author; this is an archived copy.

---

[Reversing 227] Natural Flag Processing (16 solves)

このRNNモデルが受理するフラグ文字列を探してください。

main.py という以下のような内容のPythonスクリプトと、 model_final.pth というモデルが与えられている。

import string import torch from torch import nn FLAG_CHARS = string.ascii_letters + string.digits + "{}-" CHARS = "^$" + FLAG_CHARS def sanity_check (text): global FLAG_CHARS assert text[: 7 ] == "TSGCTF{" assert text[- 1 :] == "}" assert all ([t in FLAG_CHARS for t in text]) def embedding (text): global CHARS x = torch.zeros(( len (text), len (CHARS))) for i, t in enumerate (text): x[i, CHARS.index(t)] = 1.0 return x class Model (nn.Module): def __init__ (self, inpt, hidden): super ().__init__() self.cell = nn.RNNCell(inpt, hidden) self.out = nn.Linear(hidden, 1 ) def forward (self, xs): h = None for x in xs: h = self.cell(x, h) return self.out(h) def inference (model, text): model.eval() with torch.no_grad(): x = embedding( "^" +text+ "$" ).unsqueeze( 1 ) y = model(x)[ 0 ].sigmoid().cpu().item() return y model = Model( len (CHARS), 520 ) model.load_state_dict(torch.load( "model_final.pth" )) text = input ( "input flag:" ) sanity_check(text) if inference(model, text) > 0.5 : print ( "Congrats!" ) else : print ( "Wrong." )

Model の forward メソッドを以下のものに差し替える。

def forward (self, xs): h = None for i, x in enumerate (xs): h = self.cell(x, h) print ( max (h[ 0 ])) return self.out(h)

この状態で inference(model, 'AAAAAAA') と inference(model, 'TSGCTF{') を実行してみると、それぞれ以下のような結果になった。正解の文字でなければ結果が大きく変わるっぽい。これを使ってちょっとずつフラグを特定していこう。

今度は Model の forward メソッドを以下のものに差し替える。

def forward (self, xs): h = None for i, x in enumerate (xs): h = self.cell(x, h) result = float ( max (h[ 0 ])) if result < 0.5 : raise Exception (i) return self.out(h)

このまま以下のコードを実行してみると、 mRnA や mRNa などそれっぽい文字列を出力し始めた。定期的に止めて人の手でそれっぽい文字列だけを残すようにしてやり、再び実行するという流れを何度も繰り返すと、最終的に mRNA-st4nDs-f0r-mANuaLLy-tun3d-RecurrEn7-N3uRAl-AutoM4toN という文字列が得られた。

import queue q = queue.Queue() q.put( '' ) while not q.empty(): tmp = q.get() l = len (tmp) + 8 for c in FLAG_CHARS: try : inference(model, 'TSGCTF{' + tmp + c + '}' ) except Exception as e: if e.args[ 0 ] > l: print (tmp + c) q.put(tmp + c)

TSGCTF{mRNA-st4nDs-f0r-mANuaLLy-tun3d-RecurrEn7-N3uRAl-AutoM4toN}
