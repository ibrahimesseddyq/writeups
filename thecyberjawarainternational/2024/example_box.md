# Example_Box

**CTF:** TheCyberJawaraInternational 2024  
**Category:** web  
**Original writeup:** <https://araisantai.gitbook.io/araisantai-archives/ctf-archive/web-the-cyber-jawara-international-2024#example-box> — by **araisantai.gitbook.io**. All credit goes to the original author; this is an archived copy.

---

Example Box

Category Points Author Web; whitebox 495 farisv

Code Analysis

Here is the main code that provided

Copy from flask import Flask, abort, render_template, request, Response from re import sub from unidecode import unidecode from urllib3.util import parse_url import requests app = Flask(__name__) allowed_hostname = ["example.com"] allowed_path = ["", "/"] fallback = "http://example.com/" cache = {} def normalize(token): if token == None: token = "" return sub(r'\s+', '', unidecode(str(token))) def filter_url(url): parsed_url = parse_url(url) scheme = normalize(parsed_url.scheme) # http host = normalize(parsed_url.host) path = normalize(parsed_url.path) filtered_url = url if not scheme.startswith('http'): filtered_url = fallback if not host in allowed_hostname: filtered_url = fallback if not path in allowed_path: filtered_url = fallback return normalize(filtered_url) @app.route('/', methods=['GET', 'POST']) def index(): url = request.form.get('url', '') return render_template('index.html', url=url) @app.route('/fetch_url') def fetch_url(): url = request.args.get('url') filtered_url = filter_url(url) print("request from: ", request.remote_addr) # print("cache now: ") try: if filtered_url in cache: response = cache[filtered_url] else: response = requests.get(filtered_url) cache[filtered_url] = response return Response(response.content, status=response.status_code, content_type=response.headers.get('Content-Type')) except requests.exceptions.RequestException as e: return f"Error fetching the URL: {e}", 500 @app.route('/flag') def flag(): if request.remote_addr != '127.0.0.1': abort(403) with open('/flag.txt', 'r') as flag: return flag.read() if __name__ == '__main__': app.run(debug=False, host='0.0.0.0', port=20002)

Reviewing the code we know that, this must be related to ssrf but there is some url parsing filter that we should bypass. Although the code looks very simple but i found that its really tricky to bypass.

Our objective is to access path flag with 127.0.0.1 remote addres http://127.0.01/flag

Copy flag(): if request.remote_addr != '127.0.0.1': abort(403) with open('/flag.txt', 'r') as flag: return flag.read()

Reviewing the code the normalize function is to check if there is whitespace or any unicode in the url.

filter_url is check for the whitelist.

Copy def filter_url(url): parsed_url = parse_url(url) scheme = normalize(parsed_url.scheme) # http host = normalize(parsed_url.host) path = normalize(parsed_url.path) filtered_url = url if not scheme.startswith('http'): filtered_url = fallback if not host in allowed_hostname: filtered_url = fallback if not path in allowed_path: filtered_url = fallback return normalize(filtered_url)

the challenge is we need to bypass parse_url and fallback overwrite, to perform this there is a good reads that that i found for bypassing the parse_url() in python.

Yepp we can use @ to bypass the parse_url,

Exploitation

Because there are some whitelist so we need to use it to perform SSRF

Copy allowed_hostname = ["example.com"] allowed_path = ["", "/"] fallback = "http://example.com/" cache = {}

I perform some test code to debug the website and see how the parsed works

We can see we have finally bypass the parse_url and fallback. but the request python still accepting our request as example.com

the problem is if we access the path flag manually like this http://127.0.0.1/flag@example.com the parse will works again and our input will be only example.com .

So we need to use unicode because normalize function and we are using ? for the reuqest to not accessing path @example.com.

So here is the full payload:

Flag: CJ{enough_with_example.com_here_is_your_nice_Phl4g!}
