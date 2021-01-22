---
layout: post
title: Sending emails with C++ and GMail
date: 2018-09-03
---

In this post I'm going to show how to send emails using C++ and a GMail account.

For networking I'm going to be using Asio (non-boost verion).

If you just want to see the code, here it is: [github.com/tuket/send_gmail](https://github.com/tuket/send_gmail).

### SSL

You cannot simply connect to GMail through a regular socket. You will have to use SSL to stablish a secure connection. In Asio you will have to create an `ssl::context`:

``` c++
asio::ssl::context sslContext(asio::ssl::context::tlsv12_client);
```

Then we have to load the certificates.
``` c++
sslContext.set_default_verify_paths();
```
This should set the path where it looks for certificates to the default system one. But it didn't work for me. In Ubuntu I had to manually set the path:

``` c++
sslContext.add_verify_path("/etc/ssl/certs");
```

### Basic structure

This will be the structure of our main function:

``` c++
asio::ssl::context sslContext(asio::ssl::context::tlsv12_client);
sslContext.add_verify_path("/etc/ssl/certs");
asio::io_context ioc;

EPostMan postMan(sslContext, ioc, "smtp.gmail.com", 465, "myemail@gmail.com", "mypassprd");
postMan.send(EMail{"recipient@outlook.com", "subject", "body"},
[](EPostMan::ErrorCode ec) {
    if(ec) {
        cout << "error sending email" << endl;
    }
    else {
        cout << "email sent!" << endl;
    }
});

ioc.run();
```

As you can see we have two new classes here. `EMail`is going to be a simple struct with the email to be sent. `EPostMan`is where we are going to write the SMTP protocol stuff.


### Connect steps

1. **Resolve DNS endpoint**. We have to translate our URL (`smtp.gmail.com:465`) to the actual enpoint.
``` c++
asio::ip::tcp::resolver::iterator endpointIt = resolver.resolve(serverUrl, to_string(port));
```

2. **Create the socket**.
``` c++
typedef asio::ssl::stream<Socket> SslSocket;
SslSocket socket(ioc, sslContext)
```

3. **Connect to the socket**. We use the low level TCP socket for connecting.
``` c++
asio::connect(socket.lowest_layer(), endpointIt);
```

4. **Set verify mode and perform the handshake**. The verification mode that we have to use for clients is `ssl::verify_peer`.

``` c++
socket.set_verify_callback(make_verbose_verification(
    asio::ssl::rfc2818_verification(serverUrl)));  // this is so we can print certificate stuff

socket.set_verify_mode(asio::ssl::verify_peer);
try{
    socket.handshake(SslSocket::client);
}
catch(const std::exception& e){
    cout << e.what() << endl;
}
```

All these steps are located in the constructor of `EPostMan`.

### SMTP steps

For the SMTP steps, it will be easier to understand if we use the `openssl` command line tool.

So, we will start by typing this in the command line:
```
openssl s_client -connect smtp.gmail.com:465
```

That will print out a bunch of certificate related stuff.

**1. Say hello**. We are going to stablish a conversation, let's say hello! just type in `EHLO` and press enter. For some reason Gmail doesn't like `EHLO` by itself, so we will have to write something next to it. For example:
```
EHLO smtp.gmail.com
```

Gmail will respond with something like:
```
250-smtp.gmail.com at your service, [92.2.159.213]
250-SIZE 35882577
250-8BITMIME
250-AUTH LOGIN PLAIN XOAUTH2 PLAIN-CLIENTTOKEN OAUTHBEARER XOAUTH
250-ENHANCEDSTATUSCODES
250-PIPELINING
250-CHUNKING
250 SMTPUTF8
```


**2. Login**. In order to login write:
```
AUTH LOGIN
```

And the server will respond with:
```
334 VXNlcm5hbWU6
```
What is that? It's a message encoded in base64. Paste that code here: [base64decode.org](base64decode.org). It says `Username:`. So, we will respond in base64 too.

**3. User and password**
Go to this webpage [base64encode.org](base64encode.org) and encode you gmail address. Paste that into the command line.

The server will reply with:
```
334 UGFzc3dvcmQ6
```

Which means `Password:`. So, do repeat the same process with your password.

If everything goes fine you will get:
```
235 2.7.0 Accepted
```

**4. Specify emitter and recipient**. Now we have to specify those two:
```
mail from:<myuser@gmail.com>
250 2.1.0 OK (come code) - gsmtp
rcpt to:<dest@outlook.es>
250 2.1.5 OK (some code) - gsmtp
```

**5. Write the email**. Finally we will get to write the email.
Type in:
```
data
```

We will get a reply such as:
```
354  Go ahead (some code) - gsmtp
```

So let's write the email.
```
from:<myuser@gmail.com>
to:<dest@outlook.es>
subject:Here the goes subject
This is the body of the email.
We need to end the message with a single dot on it's own line.
.

```

In the Linux command line you have to press `shift+V + enter` to generate the correct sequence for a new line (CRLF).
The command line should display something like:
```
blabla
blabla last line
^M
.^M

```

In C code it has to end with the following string: `"\r\n.\r\n"`.

The server will respond with:
```
250 2.0.0 OK (some code) - gsmtp
```

And you have successfully sent an email! Check you inbox!

**6. End the session**. When you get tired of sending emails you can type in `quit` that will end the session.

That's all you have to do.

Find here the code sample: [github.com/tuket/send_gmail](https://github.com/tuket/send_gmail)
