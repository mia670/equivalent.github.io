---
layout: article_post
categories: article
title:  "Rails CSRF protection for SPA"
disq_id: 51
description: "How to secure Rails API for SPA with CSRF protection. Is it needed for token headers (e.g.: Bearer token, JWT) ? Or just for session cookies?"
---

**update 2018-06-27** Added section and updates around CSRF Breach Attack

Topic of SPA (Single Page Applications like React) and Ruby on Rails as an API only is around for a while.
This Frontend & Backend split inspired lot of other technology
approaches like JWT (JSON Web Tokens)

But there is quite lot of confusion around security. One of the biggest
topics is CSRF (Cross Site Request Forgery).

I've already wrote an article on the topic before why it is needed in SPA that uses sessions
( [CSRF protection on single page app API](https://blog.eq8.eu/article/csrf-protection-on-single-page-app-api.html) )
but really didn't provide any "how to" guide.

Let's fix this in this article. All the methods mentioned here are
equally valid it just really depends on the usecase your application
needs.

> Special thanks goes to [johnunclesam](https://github.com/johnunclesam) for
> asking me about
> [this topic](https://github.com/equivalent/scrapbook2/issues/10#issuecomment-393104531)
> My answer was formed into this article.



## Recap on how CSRF works

```
Given I'm a user of my-bank.com
And I'm signed in to my-bank.com (cookie session id / cookie JWT)
When I click on a malicious link (email, `<img src="...">`, 3rd party website) with url `https://my-bank?transfer_all_my_money_to_user=1234`
Then all my money ends up transfered to hacker user 1234
```

> You can watch demo video of this attack [here](https://www.youtube.com/watch?v=5joX1skQtVE)

The whole point of CSRF token protection is that because of cookie
session_id (or cookie with JWT) is sent on every request therefore we need one more
information (CSRF token) that server will acknowledge as browser action
by user not by malicious link.

> GET requests are not CSRF protected in Rails because it's not
> common to do "actions" like "transfer all my money" with GET requests.
> This is just simple example.

So this leaves us to the point where should the CSRF be stored ?

In HTML rendering Rails application  the CSRF token is rendered in
Rails form as a hidden field therefore it's send automatically when HTML forms are submitted.

```erb
<%= form_for @user do |f| >
  # ....
<% end %>
```

Rails HTML render will also render the CSRF token in the  `<meta name="csrf-token" ...>` tag 
so when it comes to AJAX  or SPA rendered from Rails we just
configure them to pick up the CSRF token from `<meta>` tag and send it with the AJAX request.


```erb
<html>
  <head>
    <%= csrf_meta_tags %>
```

![](https://raw.githubusercontent.com/equivalent/equivalent.github.io/master/assets/2018/rails-rendering-spa.png)


This CSRF token is generated once and used trough out of lifecycle of session:

> Rails will appear to generate a new CSRF token on every request, but it
> will accept any generated token from that session. In reality, it is
> just masking a single token using a one-time pad per request, in order
> to protect against SSL BREACH attack. More details at
> <https://stackoverflow.com/a/49783739/2016618>. You don't need to
> track/store these tokens.
>
> source: <https://stackoverflow.com/a/50225227/473040>


When it comes to API only Ruby on Rails application where SPA is not
rendered by Rails (separate server serving static files) we don't have this
luxury of `<mata>` tag.


![](https://raw.githubusercontent.com/equivalent/equivalent.github.io/master/assets/2018/spa-is-independent-of-rails-api.png)

Therefore we need to provide the CSRF token to SPA some different way.

### SPA scenario: Should I store CSRF in a cookie ?

Nature of cookies is that they are sent on every request BUT Rails is
not stupid and is checking the CSRF token from `X-CSRF-TOKEN` header and
not cookie.

Lets look inside the [source_code](https://github.com/rails/rails/blob/e7feaff70f13b56a0507e9f4dfaf3ebc361cb8e6/actionpack/lib/action_controller/metal/request_forgery_protection.rb#L102)

```ruby
# inside the `protect_from_forgery` method this method is activated:
# ...
def verified_request?
  !protect_against_forgery? || request.get? || request.head? ||
    valid_authenticity_token?(session, form_authenticity_param) ||
    valid_authenticity_token?(session, request.headers['X-CSRF-Token'])
end
# ...
```

 So in theory
if you store the CSRF token  in a cookie
**it will not be read by server**. Therefore it should be ok.

> It doesn't matter if the server (Rails) set the cookie or the SPA
> (Angular, React, ...) set the cookie

The SPA just needs to send the CSRF as a header `X-CSRF-Token`.

Only way how the cookie CSRF store would be dangerous is if you write
your own CSRF token validation that would look something like this (**So don't do this!**):

```ruby
class MyController < ApplicationControlle
  def transfer_all_my_money
    raise 'CSRF token invalid' unless valid_authenticity_token?(session, cookie[:csrf_token])  # seriously never do this !!!!
    # ....
  end
end
```

> So remember, CSRF tokens should be sent via a header `X-CSRF-Token`. You need
> to configure your SPA to read the CSRF token from Local storage / Cookie
> and send it as this header.


You can configure your Rails application to set CSRF token in a
cookie after login:

```ruby
class LoginController < ApplicationController
  def create
    if user_password_match
      # ....
      cookie[:my_csrf_token]= form_authenticity_token
    end
  end
end
```

> Note: this is less secure solution than example below as it may be subject to "CSRF Breach Attack"
> described at the bottom of the article.

Or after every Post action

```ruby
class ApplicationController < ActionController::Base

  after_action :set_csrf_cookie

  def set_csrf_cookie
    cookies["my_csrf_token"] = form_authenticity_token
  end
end
```


I however prefer CSRF as JSON API Body response which I'll describe in
the
next section.

## Session Id Cookie + CSRF as JSON API Body response

Other way is to provide CSRF after login as a body of the response. Single page app
Angular, React, ...) stores it somewhere(Cookie/Local storage) and send it with every request as a header.

> If you use something like Devise for login, Devise will set session
> cookie after login. What you can do is to override the sessions
> controller in a way that we will provide
> CSRF token in response:

```
curl POST  https://api.my-app.com/login.json  -d "{"email":'equivalent@eq8.eu", "password":"Hello"}"  -H 'ContentType: application/json'

# Cookie with session_id was set

response:

{  "login": "ok", "csrf": 'yyyyyyyyy" }

next request:

curl POST  https://api.my-app.com/transfer_my_money  -d "{"to_user_id:":"1234"}"  -H "ContentType: application/json" -H "X-CSRF-Token: yyyyyyyyy"

response:

{ ok: ok }

```

But again due to "unchanging" nature of the token this may be subject to
CSRF Breach Attack (described at bottom of the article)

So providing fresh CSRF token on every response is better:


```
curl POST  https://api.my-app.com/login.json  -d "{"email":'equivalent@eq8.eu", "password":"Hello"}"  -H 'ContentType: application/json'

# Cookie with session_id was set

response:

{  "login": "ok", "csrf": 'yyyyyyyyy" }

next request:

curl POST  https://api.my-app.com/transfer_my_money  -d "{"to_user_id:":"1234"}"  -H "ContentType: application/json" -H "X-CSRF-Token: yyyyyyyyy"

response:

{ ok: ok, "csrf": 'xxxxxxxxxx" }  # <-- important, fresh new token that SPA should update
```

> you can do this via `render json `{ ok: :ok, csrf: form_authenticity_token }` 

#### Refresh CSRF on every request

You may work for a "super secure" project where your user should interact with
your website only in one browser tab/window from one IP location. Rails CSRF is NOT designed for this ([read more](https://stackoverflow.com/questions/47723379/why-does-the-csrf-token-in-rails-not-prevent-multiple-tabs-from-working-properly))

CSRF token is valid during the lifetime of the session ([source](https://stackoverflow.com/questions/7744459/rails-csrf-tokens-do-they-expire))

If you need to ensure that every token will be valid for only one
request (a.k.a: [Non transferable tokens](https://github.com/equivalent/scrapbook2/blob/master/security_notes.md#authenticated-sessions-should-not-be-transferable-or-should-they-)) you will have to implement custom solution where you would
create table of whitelisted per-user-tokens and either set that token  in another cookie or provide it with every response of any request:

```
curl POST  https://api.my-app.com/transfer_my_money  -d "{"to_user_id:":"1234"}"  -H "ContentType: application/json" -H "Prev-Token: yyyyyyy"

response:
{
  "status": "ok",
  "next_token": "zzzzzzzz"
}
```

But this is not topic of this article.

## SPA with JWT token via Authentication Header

Most ideal solution is not to store any authentication cookie and send JWT token (or
other form of token) as an `Authentication` header with every requests.

**No Cookie Authentication == No need for CSRF**

* <https://security.stackexchange.com/questions/170388/do-i-need-csrf-token-if-im-using-bearer-jwt>
* <https://security.stackexchange.com/questions/166724/should-i-use-csrf-protection-on-rest-api-endpoints/166798#166798>

Problem of authentication cookies is that they are sent on every
request. This way if your SPA needs to set the `Authentication` header you cannot just
accidentally click on malicious link that would made the request with
the `Authentication` header.


```ruby
class ApplicationController
  # no need for `protect_from_forgery`
  before_action :authenticate

  # will check if `Authentication` header `Bearer: xxxxxxx.xxxxxx.xxxx` is valid
  def authenticate
    auth_header_value = request.header[:Authorization]
    jwt_token = auth_header_value.split(' ').last
    JWT.decode jwt_token, # ...
    # ...
  end
```

Login request

```
curl POST  https://api.my-app.com/login.json  -d "{"email":'equivalent@eq8.eu", "password":"Hello"}"  -H 'ContentType: application/json'

response:

{ "login": "ok", "jwt": "xxxxxxxxxxx.xxxxxxxx.xxxxx" }
```

Subsequent request of the SPA would be:

```
curl POST  https://api.my-app.com/transfer_my_money  -d "{"to_user_id:":"1234"}"  -H "ContentType: application/json" -H "Authentication: Bearer xxxxxxxxxxx.xxxxxxxx.xxxxx"
```

If someone would send you a malicious link to `https://api.my-app.com/transfer_my_money` it would not contain the header !

There are also other benefits of not using cookie authentication:

* Backend API only application should NOT be telling  SPA
  how to store any information. This way we will give full responsibility to the SPA on
  "how to store" token and how to set the header. Rails application will only look
  at request header `Authentication` for the token.
* Cookies are Domain locked. If you need to create microservices
  that communicate with multi domain with same JWT you cannot do that with
  cookies (which is a good security thing in most cases but not so good
  if you need to do more advanced stuff).


### Different kind of attacks

You just solved one of the security issues not all of them. I will not
go into details in this article, but here are some hints:

If you decide to use session cookies, cookies should have
been set to attribute `secure` and `httpOnly`. This way cookies are
prevented to be read by JS and will be send over `https` only.

> If you use Devise for authentication check [here](https://github.com/equivalent/scrapbook2/blob/master/security_notes.md#use-secure-cookies)

* <http://guides.rubyonrails.org/security.html>
* <https://www.owasp.org/index.php/HttpOnly>

Also you should specify domain for your cookie [more here](https://www.mxsasha.eu/blog/2014/03/04/definitive-guide-to-cookie-domains/)
(article includes lot of information on session fixation attack, I recommend to read it)

```ruby
# exapmle of super paranoid cookie:
cookies['cookie_name'] = {
  :value => 'xxxxxxxxxx',
  :expires => 10.minutes.from_now,
  :domain => 'www.example.com',
  :httponly => true,
  :secure => true
}
```

> Use Chrome/Firefox inspector to see if this flags are truly set on
> your cookies. Don't just blindly conclude this will work !

In any case your API server should accept  `https` only !  (read more: [Enforcing HTTPs in Rails app](https://blog.eq8.eu/article/force_ssl_is_different_than_force_ssl.html)).

In case of header `Authentication` token API you need to store the token
somewhere. If you use local storage (browser storage) you need to take
extra care around XSS (Cross Site Scripting attack) in your FE.

This will be a problem even if you  consider to store tokens in a
cookie set on  the FE side as that cookie cannot have `httponly` flag
(as JS needs to interact with it)

> Remember: just because you have no CSRF
> issue it doesn't mean that Cross Site Scripting(XSS) cannot steal this token !

* <https://stackoverflow.com/questions/35291573/csrf-protection-with-json-web-tokens>

When backend API is just accepting headers
(and no cookies) there is more pressure on the frontend SPA to be secure.

> With token based solutions you should have whitelist of valid
> tokens (or at least blacklist of invalid tokens) on the Backend side
> (e.g. inside Redis, or DB). This applies to even such technology as
>  JWT which suppose to be "stateless". Imagine a scenario where you
>  need to revoke certain tokens
> if there was a breach (or user reset his password) [more here](https://youtu.be/67mezK3NzpU?t=31m32s) 

There is no silver bullet when it comes to security! Educate yourself
before you implement something in production :)

### CSRF Breach attack

Quoting [this](https://www.adweek.com/digital/breach-csrf/) and
[this](https://www.facebook.com/notes/protect-the-graph/preventing-a-breach-attack/1455331811373632)
article:


```
Under certain circumstances, attackers could figure out a victim’s
secret token even when the communication was encrypted. Specifically,
the attacker could discover information about the token because of the
way that a Web page gets compressed.

Compression is a powerful way to speed up communication because repeated
sections of text only need to get transmitted once. For example, if the
word “Beetlejuice” is repeated three times in a Web page, the second and
third instances get compressed into tiny shortcuts that refer to the
first word. Similarly — and unfortunately — if any letters in the Web
page match some of the letters in the CSRF token, compression makes the
Web page smaller. This size difference is still present after the Web
page is encrypted, and this effect could be used to reveal information
about the CSRF token to any attackers who can see the size of the
compressed Web page. A smart attacker could use a few hundred carefully
crafted Web requests to figure out the entire token from the size alone.
```

So Given you use sessions and need CSRF
in order to increase your security your Frontend (SPA) requests should be sending
fresh CSRF tokens and Backend should be providing fresh CSRF tokens:


```ruby
class ApplicationController < ActionController::Base

  after_action :set_csrf_cookie

  def set_csrf_cookie
    cookies["my_csrf_token"] = form_authenticity_token
  end
end
```

or


```ruby
class MyController < ApplicationController
  def create
    # ...
    render json: { ok: :ok, csrf: form_authenticity_token }
  end
end
```

> Some more of my thoughts on CSRF Breach attack & Rails are summarized [here](https://github.com/equivalent/scrapbook2/issues/10#issuecomment-400547708)



### Sources

Security is not easy.

* <https://github.com/equivalent/scrapbook2/issues/10#issuecomment-393104531>
* <https://stackoverflow.com/questions/50159847/single-page-application-and-csrf-token>
* <https://stackoverflow.com/questions/47723379/why-does-the-csrf-token-in-rails-not-prevent-multiple-tabs-from-working-properly/49783739#49783739>
* <https://stackoverflow.com/questions/50134071/authenticate-apis-for-all-clients-type-with-one-or-many-methods>
* <https://stackoverflow.com/questions/20504846/why-is-it-common-to-put-csrf-prevention-tokens-in-cookies>
* <https://stackoverflow.com/questions/1336126/does-every-web-request-send-the-browser-cookies>
* <https://www.owasp.org/index.php/HttpOnly>
* <https://github.com/equivalent/scrapbook2/blob/master/security_notes.md#use-secure-cookies>
* <http://api.rubyonrails.org/classes/ActionController/RequestForgeryProtection/ClassMethods.html#method-i-protect_from_forgery>
* <https://github.com/rails/rails/blob/e7feaff70f13b56a0507e9f4dfaf3ebc361cb8e6/actionpack/lib/action_controller/metal/request_forgery_protection.rb#L102>
* <https://stackoverflow.com/questions/8503447/rails-how-to-add-csrf-protection-to-forms-created-in-javascript>
* <https://security.stackexchange.com/questions/170388/do-i-need-csrf-token-if-im-using-bearer-jwt>
* <https://security.stackexchange.com/questions/166724/should-i-use-csrf-protection-on-rest-api-endpoints/166798#166798>
* <https://stackoverflow.com/questions/47723379/why-does-the-csrf-token-in-rails-not-prevent-multiple-tabs-from-working-properly>
* <https://stackoverflow.com/questions/7744459/rails-csrf-tokens-do-they-expire>
* <https://www.mxsasha.eu/blog/2014/03/04/definitive-guide-to-cookie-domains/>
* <http://michael-coates.blogspot.com/2010/07/html5-local-storage-and-xss.html>

Related article: [CSRF protection on single page app API](https://blog.eq8.eu/article/csrf-protection-on-single-page-app-api.html)

Related discussions:

* <https://twitter.com/equivalent8/status/1007366289336291328>
* <https://www.reddit.com/r/ruby/comments/8r55qf/rails_api_csrf_protection_for_spa/>
* <https://www.reddit.com/r/AskNetsec/comments/8r9kkc/security_article_on_csrf_peer_review_request/>
