### wm_basic_auth

wm_basic_auth is a dead simple, flexible implementation of HTTP Basic authentication for Webmachine.  Just define a function to check usernames and passwords, add a small is_authorized/2 function to any resources you want protected, and you're done.  Since HTTP Basic auth sends passwords in cleartext, it should only be used with TLS/SSL.

### Using wm_basic_auth

#### Add wm_basic_auth as a dependency in your rebar.config

	{deps, [{wm_basic_auth, ".*", {git, "git://github.com/b/wm_basic_auth", "HEAD"}}]}.

#### Implement your authorization function

Your authentication function may use whatever means you like to verify access, but must match
this specification:

	-spec my_auth_fun(Realm :: string(),
	                  Username :: string(),
	                  Password :: string()) -> ok | {error, Reason :: string()}.

#### Call wm_basic_auth:is_authorized/3 from webmachine:is_authorized/2

	is_authorized(ReqData, State=#state{realm=Realm}) ->
	    Response = wm_basic_auth:is_authorized(ReqData, Realm, fun ?MODULE:my_auth_fun/3),
	    {Response, ReqData, State}.


#### Enable TLS/SSL

This step is optional, but strongly recommended.  The additional SSL configuration below is the minimal
amount needed to get you going, though it does assume a self-signed certificate.  Great for testing,
probably not what you want for production.

	WebConfig = [
	             {ip, Ip},
	             {port, 8443},
	             {log_dir, "priv/log"},
	             {dispatch, Dispatch},
	             {ssl, true},
				 {ssl_opts, [{keyfile, "priv/server.key"},
							 {certfile, "priv/server.pem"},
							 {cacertfile,"priv/server.pem"}]}],

### Example resource

	-module(basic_auth_resource).
	-export([init/1, to_html/2]).
	-export([is_authorized/2, authorize/3]).

	-include_lib("webmachine/include/webmachine.hrl").

	-record(state, {realm, params}).

	init([]) -> {ok, #state{realm="testrealm@b3k.us"}}.

	is_authorized(ReqData, State=#state{realm=Realm}) ->
	    Response = wm_basic_auth:is_authorized(ReqData, Realm, fun ?MODULE:authorize/3),
	    {Response, ReqData, State}.

	to_html(ReqData, State) ->
	    {"<html><body>Hello, new world</body></html>", ReqData, State}.

	authorize(_Realm, Username, Password) ->
	  	case Password == proplists:get_value(Username,
	    			                         [{"foo", "bar"}, {"baz", "bal"}]) of
			true -> ok;
			false -> {error, "Authorization failed"}
		end.

Add the example resource to your webmachine app, compile, and start it up.  You can test
authentication easily using curl:

	$ curl -v --user foo:bar -k https://localhost:8443/
	* About to connect() to localhost port 8443 (#0)
	*   Trying ::1... Connection refused
	*   Trying 127.0.0.1... connected
	* Connected to localhost (127.0.0.1) port 8443 (#0)
	* SSLv3, TLS handshake, Client hello (1):
	* SSLv3, TLS handshake, Server hello (2):
	* SSLv3, TLS handshake, CERT (11):
	* SSLv3, TLS handshake, Server key exchange (12):
	* SSLv3, TLS handshake, Server finished (14):
	* SSLv3, TLS handshake, Client key exchange (16):
	* SSLv3, TLS change cipher, Client hello (1):
	* SSLv3, TLS handshake, Finished (20):
	* SSLv3, TLS change cipher, Client hello (1):
	* SSLv3, TLS handshake, Finished (20):
	* SSL connection using DHE-RSA-AES256-SHA
	* Server certificate:
	* 	 subject: C=US; ST=Washington; L=Seattle; O=b3k; CN=localhost; emailAddress=b@b3k.us
	* 	 start date: 2012-05-09 19:58:27 GMT
	* 	 expire date: 2013-05-09 19:58:27 GMT
	* 	 common name: localhost
	* 	 issuer: C=US; ST=Washington; L=Seattle; O=b3k; CN=localhost; emailAddress=b@b3k.us
	* 	 SSL certificate verify result: self signed certificate (18), continuing anyway.
	* Server auth using Basic with user 'foo'
	> GET / HTTP/1.1
	> Authorization: Basic Zm9vOmJhcg==
	> User-Agent: curl/7.21.4 (universal-apple-darwin11.0) libcurl/7.21.4 OpenSSL/0.9.8r zlib/1.2.5
	> Host: localhost:8443
	> Accept: */*
	> 
	< HTTP/1.1 200 OK
	< Server: MochiWeb/1.1 WebMachine/1.9.0 (someone had painted it blue)
	< Date: Thu, 07 Jun 2012 07:17:57 GMT
	< Content-Type: text/html
	< Content-Length: 42
	< 
	* Connection #0 to host localhost left intact
	* Closing connection #0
	* SSLv3, TLS alert, Client hello (1):
	<html><body>Hello, new world</body></html>

### Blame

wm_basic_auth is made by Benjamin Black and can be found at https://github.com/b/wm_basic_auth.
