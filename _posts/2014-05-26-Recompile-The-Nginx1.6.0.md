---
layout: post
title: Recompile The Nginx1.6.0
description: For info security requirement, we need consider the web server basic safety. So following some security rules is mandatory.
category: operation
---

##Basic Environment
- [Centos6.3](http://www.centos.org)
- [Openssl1.0.1g](http://www.openssl.org)
- [ Nginx1.6.0](http://nginx.org)


##Some knowledge
About the ngix compile, we maybe need understand the some info, like so:

		./configure \
		--prefix=%{prefixdir} \
        --sbin-path=%{prefixdir}/sbin/nginx \
        --conf-path=%{prefixdir}/nginx.conf \
        --error-log-path=%{_localstatedir}/log/nginx/error.log \
        --http-log-path=%{_localstatedir}/log/nginx/access.log \
        --pid-path=%{_localstatedir}/run/nginx/nginx.pid \
        --lock-path=%{_localstatedir}/run/nginx/nginx.lock \
		--http-client-body-temp-path=%{_localstatedir}/cache/nginx/client_temp \
		--http-fastcgi-temp-path=%{_localstatedir}/cache/nginx/fastcgi_temp \
		--http-proxy-temp-path=%{_localstatedir}/cache/nginx/proxy_temp \
        --user=%{nginx_user} \
        --group=%{nginx_group} \
		--with-openssl=/opt/openssl-1.0.1g \
		--with-http_ssl_module \
		--with-http_gunzip_module \
        --with-http_gzip_static_module \
		--with-http_realip_module \
		--with-http_addition_module \
		--with-http_sub_module \
		--with-http_dav_module \
		--with-http_secure_link_module \
		--with-http_stub_status_module \
		--with-http_auth_request_module \
        --with-pcre \
		--with-ipv6 \
		--with-cc-opt="%{optflags} $(pcre-config --cflags)" \
        $*

Actually from the above configure parameters, we could find:

- We need openssl and openssl-devel libray(if we need enable ssl module).
- The pcre is for rewrite function.
- md5 and sha1 will also depend the openssl lib.
- We could find the execution of module  from "objs/ngx_modules.c" in Nginx source folder.
- How to explain the nginx static compile or dynamic

	>1. Nginx, not like other web container,apache or lighttpd, it can't be dynamic. 
	>2. Use the `nginx -V` to  check the compile parameter, if appear --with-openssl, it means nginx contains the openssl dynamically.	
	>3. Use `ldd ./sbin/nginx` to check the dependent libraries, if output doesn't include openssl.so, it mean compile openssl dynamically.
	>4. Additionally, we could check the system library openssl whether open when nginx is running:
			
			ps ax|grep nginx
			lsof -p nginx_pid|grep ssl
- When we compile nginx with openssl, sometimes ,we will meet the below error messages:

		make[3]: Leaving directory `/apps/lib/openssl-1.0.0k/crypto'
		make[2]: Leaving directory `/apps/lib/openssl-1.0.0k'
		make[1]: *** [/apps/lib/openssl-1.0.0k/.openssl/include/openssl/ssl.h] Error 2
		make[1]: Leaving directory `/root/rpmbuild/BUILD/nginx-1.4.4'
		make: *** [build] Error 2
		error: Bad exit status from /var/tmp/rpm-tmp.uh3FA8 (%build)
	The corresponding solution is(First you need make the openssl from its resource folder, you mean what I mean):
		
		edit $nginx-1.4.4_source/auto/lib/openssl/conf(31-34 lines)
		#CORE_INCS="$CORE_INCS $OPENSSL/.openssl/include"
		#CORE_DEPS="$CORE_DEPS $OPENSSL/.openssl/include/openssl/ssl.h"
		#CORE_LIBS="$CORE_LIBS $OPENSSL/.openssl/lib/libssl.a"
		#CORE_LIBS="$CORE_LIBS $OPENSSL/.openssl/lib/libcrypto.a"
		
		**Change As:**
	
        CORE_INCS="$CORE_INCS $OPENSSL/include"
        CORE_DEPS="$CORE_DEPS $OPENSSL/include/openssl/ssl.h"
        CORE_LIBS="$CORE_LIBS $OPENSSL/libssl.a"
        CORE_LIBS="$CORE_LIBS $OPENSSL/libcrypto.a"
        CORE_LIBS="$CORE_LIBS $NGX_LIBDL"</p>

	

- About spdy protocol for compiling Nginx, we need to know that we should use openssl.0.1 to active spdy module.Namely, it can use openssl<1.0.1 if we don't need `--with-http_spdy_module \`

##Openssl Heartbleed 
The vulnerability dubbed “Heartbleed” was found in the popular OpenSSL cryptographic software library (http://heartbleed.com).  OpenSSL is widely used, often with applications and web servers like Apache and Nginx.   OpenSSL versions 1.0.1 through 1.0.1f contain this vulnerability, which attackers can exploit to read the memory of the systems.  Gaining access to the memory could provide attackers with secret keys, allowing them to decrypt and eavesdrop on SSL encrypted communications and impersonate service providers. Data in memory may also contain sensitive information including usernames and passwords.

Heartbleed is not a vulnerability with SSL/TLS, but rather a software bug in the OpenSSL heartbeat implementation. SSL/TLS is not broken; it is still the gold standard for encrypting data in transit on the Internet. However, due to the popularity of OpenSSL, approximately 66% of the Internet or two-thirds of web servers (according to Netcraft Web server report ) could be using this software. Companies using OpenSSL should update to the latest fixed version of the software (1.0.1g) or recompile OpenSSL without the heartbeat extension as soon as possible.

As the world’s leading Certification Authority, Symantec has already taken steps to strengthen our systems. Our roots are not at risk; however, we are following best practices and have re-keyed all certificates on web servers that have the affected versions of OpenSSL.

After companies have updated or recompiled their systems, Symantec is recommending that customers replace all their certificates -regardless of issuer- on their web servers to mitigate the risks of security breach. Symantec will be offering free replacement certificates for all our customers.   

Finally, Symantec is asking customers to reset passwords to their SSL and code-signing management consoles.  Again, this is a best practice and we encourage companies to ask their end customers to do the same after their systems have applied the fix.  We will continue to work with our customers to minimize the impact of security risks from this vulnerability.

For your convenience, here is a summary of steps to take:
For businesses:

  - Anyone using OpenSSL 1.0.1 through 1.0.1f should update to the latest fixed version of the software (1.0.1g), or recompile OpenSSL without the heartbeat extension.  
  - Businesses should also replace the certificate on their web server after moving to a fixed version of OpenSSL.
  - Finally, and as a best practice, businesses should also consider resetting end-user passwords that may have been visible in a compromised server memory.

For consumers:

  - Should be aware their data could have been seen by a third party if they used a vulnerable service provider.
  - Monitor any notices from the vendors you use. Once a vulnerable vendor has communicated to customers that they should change their passwords, users should do so.
  - Avoid potential phishing emails from attackers asking you to update your password – to avoid going to an impersonated website, stick with the official site domain.

Check out: [heartbleed](http://symantec.com/heartbleed)  for more information, including on how to test if a server is vulnerable to Heartbleed attacks.




