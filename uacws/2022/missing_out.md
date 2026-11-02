# Missing_Out

**CTF:** UACWS 2022  
**Category:** web  
**Original writeup:** <https://ctftime.org/writeup/34051> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

In this challenge was provided the flask app source code. The main goal was to spot the miss configurations.

Accessing the URL provided:

![Alt text](https://github.com/uac-ctf/UA-CSW-CTF2022-Writeups/blob/master/web/missing_out/images/home_page.png?raw=true)

There is also a upload file endpoint:

![Alt text](https://github.com/uac-ctf/UA-CSW-CTF2022-Writeups/blob/master/web/missing_out/images/upload_file.png?raw=true)

There is nothing else visible. Brute forcing directories was useless since we have the source code, and we can see the app endpoints. There is also a */get_file* endpoint:

```python

@app.route('/get_file/<path:name>')

def get_file(name):

return send_from_directory(app.config['FILES_FOLDER'], name, as_attachment=True)

```

So if we analyze the code we will spot two things, the first one is that the application logs are being written to a file in */tmp/app*, which is also the same directory used in the */get_file* endpoint.

```python

app.config['FILES_FOLDER'] = '/tmp/app'

logging.basicConfig(filename='/tmp/app/app.log', level=logging.DEBUG, format=f'%(asctime)s %(levelname)s %(name)s %(threadName)s : %(message)s')

```

So we will be able to fetch the application logs, but what can we do with that? That's where the second miss-configuration appears, the debug mode is on:

```python

if __name__ == '__main__':

app.run(host='0.0.0.0', port=8000, debug=True)

```

meaning that we can access the */console* endpoint, that is related with the [werkzeug debugger](https://werkzeug.palletsprojects.com/en/2.1.x/debug/).

![Alt text](https://github.com/uac-ctf/UA-CSW-CTF2022-Writeups/blob/master/web/missing_out/images/console_endpoint.png?raw=true)

But we need the PIN so that we can access it, which is written to the logs when the application starts. So we need to get the *app.log* file and get the console PIN that will be written there.

To get the logs file we can use the */get_file* endpoint, <http://cybersecweek.ua.pt:2002/get_file/app.log>. There we will find the PIN code for the console.

After entering the console we just need to find the flag and read its contents. Luckly the flag is in the same directory where the app is running.

![Alt text](https://github.com/uac-ctf/UA-CSW-CTF2022-Writeups/blob/master/web/missing_out/images/console_exec.png?raw=true)

And we get the flag,

```

CTFUA{nO_mi55ing_confs_w3Re_l3f7_bEh1Nd}

```
