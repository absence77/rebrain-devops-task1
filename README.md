Experimental QUIC support for nginx
-----------------------------------

1. Introduction
2. Installing
3. Configuration
4. Clients
5. Troubleshooting
6. Contributing
7. Links

1. Introduction

    This is an experimental QUIC [1] / HTTP/3 [2] support for nginx.

    The code is developed in a separate "quic" branch available
    at https://hg.nginx.org/nginx-quic.  Currently it is based
    on nginx mainline 1.23.x.  We merge new nginx releases into
    this branch regularly.

    The project code base is under the same BSD license as nginx.

    The code is currently at a beta level of quality, however
    there are several production deployments with it.

    NGINX Development Team is working on improving HTTP/3 support to
    integrate it into the main NGINX codebase.  Thus, expect further
    updates of this code, including features, changes in behaviour,
    bug fixes, and refactoring.  NGINX Development team will be
    grateful for any feedback and code submissions.

    Please contact NGINX Development Team via nginx-devel mailing list [3].

    What works now:

    IETF QUIC version 1 is supported.  Internet drafts are no longer supported.

    nginx should be able to respond to HTTP/3 requests over QUIC and
    it should be possible to upload and download big files without errors.

    + The handshake completes successfully
    + One endpoint can update keys and its peer responds correctly
    + 0-RTT data is being received and acted on
    + Connection is established using TLS Resume Ticket
    + A handshake that includes a Retry packet completes successfully
    + Stream data is being exchanged and ACK'ed
    + An H3 transaction succeeded
    + One or both endpoints insert entries into dynamic table and
      subsequently reference them from header blocks
    + Version Negotiation packet is sent to client with unknown version
    + Lost packets are detected and retransmitted properly
    + Clients may migrate to new address

2. Installing

    A library that provides QUIC support is required to build nginx, there
    are several of those available on the market:
    + BoringSSL [4]
    + LibreSSL [5]
    + QuicTLS [6]

    Clone the NGINX QUIC repository

    $ hg clone -b quic https://hg.nginx.org/nginx-quic
    $ cd nginx-quic

    Use the following command to configure nginx with BoringSSL [4]

    $ ./auto/configure --with-debug --with-http_v3_module         \
                       --with-cc-opt="-I../boringssl/include"     \
                       --with-ld-opt="-L../boringssl/build/ssl    \
                                      -L../boringssl/build/crypto"
    $ make

    Alternatively, nginx can be configured with QuicTLS [6]

    $ ./auto/configure --with-debug --with-http_v3_module         \
                       --with-cc-opt="-I../quictls/build/include" \
                       --with-ld-opt="-L../quictls/build/lib"

    Alternatively, nginx can be configured with a modern version
    of LibreSSL [7]

    $ ./auto/configure --with-debug --with-http_v3_module          \
                       --with-cc-opt="-I../libressl/build/include" \
                       --with-ld-opt="-L../libressl/build/lib"

    When configuring nginx, it's possible to enable QUIC and HTTP/3
    using the following new configuration options:

        --with-http_v3_module     - enable QUIC and HTTP/3
        --with-stream_quic_module - enable QUIC in Stream

3. Configuration

    The HTTP "listen" directive got a new option "http3" which enables
    HTTP/3 over QUIC on the specified port.

    The Stream "listen" directive got a new option "quic" which enables
    QUIC as client transport protocol instead of TCP or plain UDP.

    Along with "http3" or "quic", it's also possible to specify "reuseport"
    option [8] to make it work properly with multiple workers.

    To enable address validation:

        quic_retry on;

    To enable 0-RTT:

        ssl_early_data on;

    Make sure that TLS 1.3 is configured which is required for QUIC:

        ssl_protocols TLSv1.3;

    To enable GSO (Generic Segmentation Offloading):

        quic_gso on;

    To limit maximum UDP payload size on receive path:

        quic_mtu <size>;

    To set host key for various tokens:

        quic_host_key <filename>;


    By default, GSO Linux-specific optimization [10] is disabled.
    Enable it in case a corresponding network interface is configured to
    support GSO.

    A number of directives were added that configure HTTP/3:

        http3_stream_buffer_size
        http3_max_concurrent_pushes
        http3_max_concurrent_streams
        http3_push
        http3_push_preload
        http3_hq (requires NGX_HTTP_V3_HQ macro)

    In http, an additional variable is available: $http3.
    The value of $http3 is "h3" for HTTP/3 connections,
    "hq" for hq connections, or an empty string otherwise.

    In stream, an additional variable is available: $quic.
    The value of $quic is "quic" if QUIC connection is used,
    or an empty string otherwise.

Example configuration:

    http {
        log_format quic '$remote_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent" "$http3"';

        access_log logs/access.log quic;

        server {
            # for better compatibility it's recommended
            # to use the same port for quic and https
            listen 8443 http3 reuseport;
            listen 8443 ssl;

            ssl_certificate     certs/example.com.crt;
            ssl_certificate_key certs/example.com.key;
            ssl_protocols       TLSv1.3;

            location / {
                # required for browsers to direct them into quic port
                add_header Alt-Svc 'h3=":8443"; ma=86400';
            }
        }
    }
