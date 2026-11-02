# Microservices

**CTF:** GreyCatTheFlag 2023 Quals  
**Category:** web  
**Original writeup:** <https://jh123x.com/blog/2023/greyctf23-microservices/> — by **jh123x.com**. All credit goes to the original author; this is an archived copy.

---

This is the author’s writeup for the challenges Microservices in the web category.

This describes the intended solution when I made the challenge.

Challenge description #

I just learnt about microservices. That means my internal server is safe now right? I’m still making my website but you can have a free preview

Junhua

A dist.zip was also provided for this challenge with the server set up.

My Notes #

This was meant to be an easy challenge for veterans of Web CTFs and provide exposure to beginners to dig around.

Definitely did not expect the service to be down so quickly, I’m sorry for that.

Notes to future self when setting up this challenge. #

Spawn more gateways to handle more traffic. Create the dist.zip file only after the challenge is ready. Spend more time hunting for unexpected solutions.

Analyzing the files #

For this challenge, there are 3 different microservices.

Gateway Admin Page Home Page

The gateway is the main entry point for the challenge.

routes = { "admin_page" : "http://admin_page" , "home_page" : "http://home_page" } ... @app.route ( "/" , methods = [ "GET" ]) def route_traffic () -> Response: """Route the traffic to upstream""" microservice = request . args . get( "service" , "home_page" ) route = routes . get(microservice, None ) # Not that the GET request here does not attach cookies. if route is None : return abort( 404 ) # Fetch the required page with arguments appended raw_query_param = request . query_string . decode() print( f "Requesting { route } with q_str { raw_query_param } " , file = sys . stderr) res = get( f " { route } /? { raw_query_param } " ) headers = [ (k, v) for k, v in res . raw . headers . items() if k . lower() not in excluded_headers ] return Response(res . content, res . status_code, headers) ...

From the code, we can see that when the user visits the page, the gateway routes the traffic to the other endpoints. By default, the user will visit the home page. If the user passes in a route, it tries to match it with the routes dictionary. If the route is found, the user will be redirected to the route. Otherwise, it will return a 404.

From the routes dictionary, we can see that there are only 2 different for the user to choose from.

Home page Admin page

These are the only routes available for users to access.

Note: It does not attached the admin cookie in the get request from the gateway. From here, we can see that obtaining the correct cookies does not help in solving the challenge.

Now, let us take a look at the other services.

Home Page #

Next, we will be taking a look at the home page.

... @app.route ( "/" ) def homepage () -> Response: """The homepage for the app""" cookie = request . cookies . get( "cookie" , "Guest Pleb" ) # If admin, give flag if cookie == admin_cookie: return render_template( "flag.html" , flag = FLAG, user = "admin" ) # Otherwise, render normal page response = make_response(render_template( "index.html" , user = cookie)) response . set_cookie( "cookie" , cookie) return response ...

From the code, we can see that if the cookie from the request is a valid cookie, it will return the flag page with the flag. Otherwise, it will return the home page with a Guest Pleb cookie, together with a homepage as shown below.

Home Page of Microservices Challenge

The goal of this challenge is to reach this endpoint with the admin’s cookie.

Admin Page #

Last but not least, we have the admin page. Note that this services is using FastAPI instead of Flask.

@app.get ( "/" ) async def index (request: Request): requested_service = request . query_params . get( "service" , None ) if requested_service is None : return { "message" : "requested service is not found" } if requested_service == "admin_page" : return { "message" : "admin page is currently not a requested service" } requested_url = request . query_params . get( "url" , None ) if requested_url is None : return { "message" : "URL is not found" } response = get(requested_url, cookies = { "cookie" : admin_cookie}) return Response(response . content, response . status_code)

From the code, we can see that the admin page requires a service and url parameter. The service parameter is used to check if the user is requesting for the admin page. If the user is requesting for the admin page, it will return a message saying that the admin page is not a requested service.

If the user is not requesting for the admin page, it will check for the url parameter. If the url parameter is not found, it will return a message saying that the URL is not found.

Otherwise, it will make a get request to the URL with the admin’s cookie and return the response. This part will be very useful for us in finding the flag.

Now that we have taken a look at all the different services in this challenge, we should be able to come up with a solution.

From the code, we can see that the gateway does not attach the admin’s cookie when making a get request to the other services. This means that a cookie leak will not help us in solving the challenge.

However, we can see that the admin page makes a get request to the URL with the admin’s cookie. This means that we can use the admin page to make a get request to the home page with the admin’s cookie.

The main question is how do we get the admin page to make a get request to the home page with the admin’s cookie?

To get the gateway to route the traffic to the admin page, we need to pass in the service parameter with the value admin_page . However, at the admin page, it will check if the user is requesting for the admin page and reject requests if it is true. This seems contradictory.

We can print out the parameters which are received by the admin page and see what happens.

# Added below line 20 print( f "Params=' { request . query_params } ', Req Service=' { requested_service } '" , file = sys . stderr, )

With this line, we can see which query parameters are passed in to the admin page.

When visiting http://localhost:5004/?service=admin_page , the log is as expected and prints Params='service=admin_page', Req Service='admin_page' .

However, what happens when we try to define different services?

When visiting http://localhost:5004/?service=admin_page&service=home_page , we are shown the admin page and the log prints Params='service=admin_page&service=home_page', Req Service='home_page' .

From this, we can see that the gateway returns the first argument that matches the route while ignoring the rest. However, the FastAPI will return the last argument that matches the route while ignoring the rest.

To make both Flask and FastAPI process our request correctly, we need to pass in the service parameter twice. The 1st time will be to get the gateway to route the traffic to the admin page and the 2nd time will be to by pass the admin_page service check.

Combined with the url parameter, we can get the admin page to make a request to the home page with the admin’s cookie and return the results to us.

Solution Script #
