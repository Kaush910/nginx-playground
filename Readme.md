# Nginx Assessment
**Q1.** It asks how to configure Nginx so that any content ending with specific file extensions like css, jpg, jpeg, js, json, png, mp4, pdf returns a 404 error when accessed via curl. <br>
**Solution.** Create a regex to match the specific file extensions stated above. The regex `\.(css|jpg|jpeg|js|json|png|mp4|pdf)$` will match any request URI ending with the listed file extensions. Inside the location block, we will use `return 404;` to send a 404 response for requests matching these file extensions.
```js
location ~* \.(css|jpg|jpeg|js|json|png|mp4|pdf)$ {
  return 404;
}
```

**Q2.** This question asks us on how to log various fields in the Nginx access log, including time, Nginx version, remote address, request ID, status, and few other parameters. <br>
**Solution.** We can do this by defining a custom log format `log_json` in the `http` block of our Nginx config using `log_format` module. We will include all the fields requested in the question in our custom format. And will then set the `access_log` directive below in our config to specify the log file path and `log_json` to use this format.
```js
http {
    log_format log_json '{"timestamp"="$time_local",'
                         '"server_protocol": "$server_protocol", '
                         '"remote_addr": "$remote_addr", '
                         '"request_id": "$request_id", '
                         '"status": $status, '
                         '"body_bytes_sent": $body_bytes_sent, '
                         '"http_user_agent": "$http_user_agent", '
                         '"proxy_protocol_addr": "$proxy_protocol_addr", '
                         '"server_name": "$server_name", '
                         '"upstream_addr": "$upstream_addr", '
                         '"request_time": $request_time, '
                         '"upstream_connect_time": $upstream_connect_time, '
                         '"upstream_header_time": $upstream_header_time, '
                         '"upstream_response_time": $upstream_response_time, '
                         '"request_uri": "$request_uri", '
                         '"upstream_status": $upstream_status, '
                         '"ssl_session_reused": $ssl_session_reused, '
                         '"x_forwarded_for": "$http_x_forwarded_for"}';

    access_log /var/log/nginx/access.log log_json;
}
```
```js
access_log /var/log/nginx/access.log custom;
```

**Q3.** The third question is about adding HTTP security headers in Nginx, but only if they are not already set in the response from the upstream server. The question also lists default values for various headers like Strict-Transport-Security, X-Content-Type-Options, X-XSS-Protection etc. <br>
**Solution.**  One way to solve your problem is to use map directive. <br>
    Note: We cannot use `if` here, because `if`, being a part of the rewrite module, is evaluated at a very early stage of the request processing, way before `proxy_pass` is called and the header is returned from the upstream server.
```js
map $upstream_http_strict_transport_security $sts_header {
    default "none";
    "" "max-age=31536000; includeSubDomains";
}

map $upstream_http_x_content_type_options $xcto_header {
    default "none";
    "" "nosniff";
}

map $upstream_http_x_xss_protection $xxp_header {
    default "none";
    "" "1; mode=block";
}

map $upstream_http_x_frame_options $xfo_header {
    default "none";
    "" "DENY";
}

map $upstream_http_content_security_policy $csp_header {
    default "none";
    "" "frame-ancestors 'none'";
}

map $upstream_http_access_control_allow_credentials $acac_header {
    default "none";
    "" "TRUE";
}

server {
    ...

    location / {
        proxy_pass http://127.0.0.1:8080;

        # Add the headers only if they are not already set by the upstream
        add_header Strict-Transport-Security $sts_header always;
        add_header X-Content-Type-Options $xcto_header always;
        add_header X-XSS-Protection $xxp_header always;
        add_header X-Frame-Options $xfo_header always;
        add_header Content-Security-Policy $csp_header always;
        add_header Access-Control-Allow-Credentials $acac_header always;
    }
}

```
Description
- For each header, a `map` directive is used to create a mapping between the value of the corresponding header from the upstream response and a variable representing the header value to be set.
- If the header is not set in the upstream response, the variable is set to the default value specified in the `default` directive of the `map` block.
- If the header has a value in the upstream response, the variable is set to an empty string, indicating that the header should not be added.
- Inside the `location /` block, `add_header` directives are used to conditionally add the headers based on the values of the corresponding variables.

Explanation of Variables
- `$upstream_http_strict_transport_security`: This is the variable whose values we want to map. It stores the value of the `Strict-Transport-Security` header from the upstream server response.
- `$sts_header`: This is variable we're mapping the values to. It will hold the final value after the mapping is performed.
- `default "none";`: If the `Strict-Transport-Security` header is not present in the upstream response, `$sts_header` will be set to "none".
- `"" "max-age=31536000; includeSubDomains";`: This line specifies a specific mapping. If the value of `$upstream_http_strict_transport_security` is an empty string (i.e., the header is present but has no value).